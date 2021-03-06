
    idn wrapper - Windows におけるクライアント側での IDN 変換ソフトウェア

    Copyright (c) 2000,2001,2002 Japan Network Information Center.
                All rights reserved.

    *** 注意 **********************************************************
    もしもすでに mDN Wrapper (idn wrapper の前身) がインストールされて
    いるマシンに idn wrapper をインストールする場合には、インストール前
    に、ラップされているすべてのプログラムをアンラップしてください。
    *******************************************************************


1. はじめに

    Windows で国際化ドメイン名を扱えるようにするためには、Windows 上の
    クライアントアプリケーションにおいて、解決しようとする名前のエンコー
    ディングを、DNS サーバが受付ける形式のものに変換する必要があります。
    これは、Windows 上のアプリケーションが、きちんと国際化ドメイン名を
    扱えるようになっていなければならない、ということであり、本来はそれ
    ぞれのプログラムの作成者が行なうべきことです。
    
    現在 IETF にて国際化ドメイン名のフレームワークを標準化する努力が続
    けられており、その結果として一連の RFC がもうすぐ発行されることに
    なっていますが、それでも国際化ドメイン名に対応したアプリケーション
    はまだまだ少ないのが現状です。

    そこで、既存のアプリケーションを国際化ドメイン名に対応させるための
    ヘルパーアプリケーションが必要になります。idnkit に含まれる runidn 
    コマンドは Unix 系の OS での一つの解決策ですし、Windows に対する解
    決策としてはここで説明する idn wrapper があります。
    
    Windows において、多くの場合、ドメイン名解決の要求はWINSOCK DLL に
    渡されます。そこで、WINSOCK DLL を国際化ドメイン名対応のものに置き
    換えてやれば、既存のプログラムからでも国際化ドメイン名を使うことが
    できるようになります。

2. 実現方法

2.1. ラッパーDLL

    ラッパーDLL は、アプリケーションと元のDLL との間に割り込んで、アプリ
    ケーションからのDLL の呼び出しを横取りして、本来のDLL とは異なった処
    理をさせるものです。

    +------------+  Call  +------------+  Call  +------------+
    |            |------->|            |------->|            |
    |Application |        |Wrapper DLL |        |Original DLL|
    |            |<-------|            |<-------|            |
    +------------+ Return +------------+ Return +------------+
                           additional
			   processing
			   here

    アプリケーションからのDLL の呼び出しはラッパー DLLに渡されます。ラッ
    パー DLLはそこで、付加的な処理を行なって、元のDLL のエントリを呼び出
    します。また、元のDLL の処理結果は一旦ラッパー DLLに返され、ここでも
    付加的な処理を行なって、最終的な結果がアプリケーションに返されること
    になります。

    idn wrapper では、WINSOCK DLLの
    
        WSOCK32.DLL     WINSOCK V1.1
	WS2_32.DLL      WINSOCK V2.0

    に対するラッパーDLL を提供して、国際化ドメイン名の名前解決ができるよ
    うにします。16ビット版のWINSOCK (WINSOCK.DLL) は対象外です。

2.2. 処理対象のAPI

    idn wrapper はWINSOCK の名前解決に関連したAPI についてのみ付加的な処
    理を行ないます。処理の対象となるWINSOCK APIは以下のものです。

    WINSOCK 1.1, WINSOCK 2.0 の両方にあるもの
    
        gethostbyaddr
	gethostbyname
	WSAAsyncGetHostByAddr
	WSAAsyncGetHostByName
	
    WINSOCK 2.0 だけにあるもの
    
        WSALookupServiceBeginA
	WSALookupServiceNextA
	WSALookupServiceEnd

    アプリケーションによっては、これらのAPI を使わないで独自にドメイン名
    の解決を行なうものもあります。例えば、nslookupは、これらのAPI を使わ
    ないで、内部で独自にDNS リクエストの生成、解釈を行なっています。当然
    のことながら、これらのアプリケーションについては、idn wrapper では多
    言語化対応させることはできません。
    
    注：WINSOCK 2.0 には、WIDE CHARACTER ベースの名前解決のAPI として

            WSALookupServiceBeginW
            WSALookupServiceNextW
    
        もありますが、これらについてはラップしません。これらのAPI はマ
        イクロソフト仕様による国際化に対応したものですから、そのフレー
        ムワーク上で使うべきものです。これらについては他の多言語化フレー
        ムワークに変換してしまうのは危険ではないと判断しました。

2.3. 処理対象外のAPI

    上記以外のWINSOCK API については、idn wrapper はなにもしないで、元の
    WINSOCK API を呼び出します。

    idn wrapper では、元のWINSOCK DLL を名前を変えてコピーし、それを
    呼び出すように作られています。
    
        wsock32.dll     ->  wsock32o.dll
	ws2_32.dll      ->  ws2_32o.dll

    ラッパーDLL は元のWINSOCK DLL と同じ名前で作成されます。従ってidn
    wrapper がインストールされた状態では、
    
        wsock32.dll         idn wrapper for WINSOCK V1.1
	ws2_32.dll          idn wrapper for WINSOCK V2.0
	wsock32o.dll        Original WINSOCK V1.1 DLL
	ws2_32o.dll         Original WINSOCK V2.0 DLL 

    となります。

2.4. 非同期 API

    ドメイン名の変換は、以下のタイミングで行なわれる必要があります。

        DNS へのリクエスト時
	
            ローカルエンコーディング -> DNS エンコーディング

        DNS からの応答受信時

            DNS エンコーディング -> ローカルエンコーディング

    同期API においては、ローカルエンコーディングからDNS エンコーディング
    への変換は、元のAPI を呼び出す前に行われ、DNS エンコーディングからロー
    カルエンコーディングへの変換は、元のAPI から復帰してきたところで行な
    われます。

    しかし、WINSOCK の以下のAPI は非同期API で、DNS からの応答受信前に復
    帰してしまいます。

	WSAAsyncGetHostByAddr
	WSAAsyncGetHostByName

    これらのAPI においては、名前解決の完了は、Windows へのメッセージによっ
    て通知されます。このため、DNS エンコーディングからローカルエンコーディン
    グへの変換を行なうには、ラッパーは通知先のウィンドウプロシジャのメッ
    セージキューをフックして、この完了メッセージを捕獲する必要があります。

    そこで、非同期API が呼び出された場合には、idn wrapper は、通知先のウィン
    ドウプロシジャ（これはAPI のパラメタで指示されます）にフックを設定し
    ます。フックが完了メッセージ（これもAPI のパラメタで指示されます）を
    検出したなら、フックは結果の格納領域（これもAPI のパラメタで指示され
    ています）のドメイン名を、DNS 側のエンコーディングからローカルエンコー
    ディングに変換するものとします。

2.5. Wrapper DLL のインストール

    WINSOCK DLL はWindows のシステムディレクトリに置かれています。
    WINSOCK を確実にラップするには、システムディレクトリにおいて
    
        オリジナルWINSOCK DLL の名前の変更

	    ren wsock32.dll wsock32o.dll
	    ren ws2_32.dll  ws2_32o.dll

	ラッパーDLL の導入
	
	    copy somewhere\wsock32.dll wsock32.dll
	    copy somewhere\ws2_32.dll  ws2_32.dll
	    copy another DLLs also

    を行なう必要があります。

    しかし、システムディレクトリでこのようなDLL の置き換えを行なうのは大
    変危険な操作になります。
    
    a)  DLL を入れ替えた状態で、もういちど同じ操作を行なうと、オリジナル
        のWINSOCK DLL が失われてしまうことになります。

    b)  サービスパックやアプリケーションなどで、WINSOCK DLL を再導入する
        ものがありますが、これによってもWINSOCK が利用不能になることがあ
        ります。

    このような状態になると、ネットワーク機能が全く使えなくなったり、最悪
    はWindows の起動すら出来なくなる可能性があります。

    そこで、idn wrapper では、上のようなシステムレベルのラップではなく、
    アプリケーションに対するラップを基本機能として提供するものとします。

    Windows において、DLL は、基本的には

        アプリケーションのロードディレクトリ
	%SystemRoot%\System32
	%SystemRoot%
	PATH で指示されるディレクトリ

    の順序で検索されて、最初に見つかったものがロードされます。ですから、
    一般的には、DLL をアプリケーションのロードディレクトリにインストール
    すれば、そのアプリケーションからのWINSOCK の呼び出しをラップすること
    ができます。

    ただし、いくつかのアプリケーション、DLL では、検索パスを経由せずに特
    定のDLL をリンクするようになっているものがあります。このような構成の
    アプリケーション、DLL が使われた場合には idn wrapperでは対処すること
    はできません。

    注：Netscapeは特定DLL にバインドされているようで、アプリケーションディ
        レクトリへのインストールではラップできません。WINSOCK DLL 自体も
	システムディレクトリの関連DLL にバインドされているようです。一方、
	Internet ExploreやWindows Media Playerは標準のサーチパスに従って
        いるので、ラップすることができます。

2.6. エンコーディングの変換位置

    WINSOCK 2.0 をサポートしているWindows には、WINSOCK の1.1 と2.0 のそ
    れぞれに対応するDLL があり、WINSOCK 1.1 のAPI の呼び出しは2.0 の同じ
    エントリにリダイレクトされるようになっています。

        +------------+  Call  +------------+  Call  +------------+
        |            |------->|            |------->|            |
        |Application |        |WINSOCK 1.1 |        |WINSOCK 2.0 |
        |            |<-------|            |<-------|            |
        +------------+ Return +------------+ Return +------------+

    この場合には1.1 に対する呼び出しも2.0 に対する呼び出しも、ともにV2.0
    用のDLL に渡されるので、2.0用のラッパーDLL 側だけでエンコーディングの
    変換を行なうようにするべきでしょう。

    一方、WINSOCK 1.1 しかサポートしていない場合(Win95)には、1.1 に対応し
    たDLL しかありません。

        +------------+  Call  +------------+
        |            |------->|            |
        |Application |        |WINSOCK 1.1 |
        |            |<-------|            |
        +------------+ Return +------------+

    この場合には必然的に1.1 用のラッパーDLL でエンコーディングを変換しな
    ければなりません。
    
    idn Wrapepr がwindows のシステムディレクトリにインストールされた場合
    には、上の通りに動作するので、
    
        WINSOCK 2.0 あり        2.0 ラッパーで変換
	WINSOCK 1.1 のみ        1.1 ラッパーで変換

    する必要があります。
    
    しかし、アプリケーションディレクトリにインストールされた場合には動作
    が変わってきます。Windows 付属の WINSOCK 1.1 DLLは、システムディレク
    トリのWINSOCK 2.0 にバインドされているため、アプリケーションディレク
    トリ側のWINSOCK 2.0 ラッパーDLL にはリダイレクトされてきません。この
    ため、アプリケーションディレクトリへのインストールにおいては、1.1DLL、
    2.0DLLの両方でエンコーディングを変換する必要があります。

    このようなDLL 間のバインディングはドキュメントされていませんので、環
    境、バージョンによっては異なった動作をするかも知れません。そこでidn 
    wrapper では、レジストリ値によって、ラッパーDLL のどこで変換を行なう
    かを決定するようにして、インストール先による差異、あるいはバージョン
    による差異を吸収するようにします。
    
    idn wrapper 用のレジストリ設定は
    
        HKEY_LOCAL_MACHINE\SOFTWARE\JPNIC\IDN
	HKEY_CURRENT_USER\SOFTWARE\JPNIC\IDN

    以下に配置されます。エンコーディング変換を行なう位置については、この
    直下のレジストリ値 Where（REG_DWORD） によって決定します。有効な値は、
    
        レジストリ Where (REG_DWORD)

        0       WINSOCK 1.1、WINSOCK 2.0 の両方で変換する
	1       WINSOCK 2.0 があれば、WINSOCK 2.0だけで変換する
	        WINSOCK 1.1 だけの場合には WINSOCK 1.1 で変換する
	2       WINSOCK 1.1 だけで変換する
	3       WINSOCK 2.0 だけで変換する
    
    の４通りです。アプリケーションディレクトリにインストールする場合には
    「０」を、システムディレクトリにインストールする場合には「１」を設定
    する必要があります。レジストリ値が存在しない場合には「０」を想定しま
    す。これはアプリケーションディレクトリへのインストールを標準としたも
    のです。

2.7. 変換元/先のエンクコーディング

    ラッパーDLL では、解決しようとするドメイン名を、マシンのローカルエン
    コーディングからDNS サーバのエンコーディングに変換し、また、DNS が返
    してきたドメイン名(DNS サーバのエンコーディング)をマシンのローカルエン
    コーディングに戻します。

    現在、DNS 側の国際化エンコーディングについては、いくつもの方式が提
    案されています。ラッパーDLL はそれらのDNS 側エンコーディングのどれか
    ひとつに変換するように構成されます。このDNS 側エンコーディングはレジ
    ストリで指示されます。このレジストリには、idn wrapper のインストール
    時に（現時点では未定の）デフォルトエンコーディングが設定されます。当
    然、このレジストリは、後で他のものに変更することもできます。

    idn wrapper 用のレジストリ設定は
    
        HKEY_LOCAL_MACHINE\SOFTWARE\JPNIC\IDN
	HKEY_CURRENT_USER\SOFTWARE\JPNIC\IDN

    以下に配置されます。DNS 側のエンコーディングはレジストリ値 Encoding 
    （REG_SZ）で指示されます。このエンコーディング名は、libmdnで認識され
    るものでなければなりません。

        レジストリ  Encoding    (REG_SZ)
	    DNS サーバ側のエンコーディング名を設定します
    
    一方、アプリケーションが使用しているローカルエンコーディングは、通常
    はプロセスのコードページから求めます。ラッパーDLL が使用する 'iconv' 
    ライブラリは、windows のコードページ名をエンコーディング名として受付
    けることができるので、コードページ名をそのままローカルエンコーディン
    グ名として使用します。

    しかし、アプリケーションによっては、特定の国際化エンコーディングで
    ドメイン名をエンコーディングしてしまうものもあります。例えば、IEでは
    ドメイン名をUTF-8 で表記するように指示することができるようになってい
    ます。UTF-8 によるエンコーディングは、提案されている国際化方式のひ
    とつですが、国際化されたDNS サーバは他のエンコーディングしか受付け
    ないかも知れません。
    
    このような状況に対処するため、idn ラッパーは、ローカルエンコーディン
    グとしてプログラム特有のエンコーディングも受付けることができるように
    します。このようなプログラム特有のローカルエンコーディングはレジスト
    リ記載されるものとします。

    idn wrapper 用のプログラム特有のレジストリ設定は
    
        HKEY_LOCAL_MACHINE\SOFTWARE\JPNIC\IDN\PerProg
	HKEY_CURRENT_USER\SOFTWARE\JPNIC\IDN\PerProg

    以下に、プログラム名（実行モジュールファイル名）をキーとして配置され
    ます。例えば、Internet Explore の場合には、実行モジュール名の
    IEXPLOREをキーとして

        HKEY_LOCAL_MACHINE\SOFTWARE\JPNIC\IDN\PerProg\IEXPLORE

    以下に置かれます。ローカルエンコーディング名は、レジストリ値
    Encoding （REG_SZ）で指示します。これもlibmdnで認識されるものでなけれ
    ばなりません。

        レジストリ  Encoding    (REG_SZ)
	
	    アプリケーションプログラム特有のエンコーディング名（デフォル
            トのエンコーディング以外を必要とする場合）を指定します。

3.セットアップとコンフィギュレーション

    idn wrapper は、基本インストレーションとして、アプリケーションディレ
    クトリでWINSOCK をラップします。これに合わせて、セットアッププログラ
    ムとコンフィギュレーションプログラムとを提供します。
    
    注：システムディレクトリでのラップも可能ですが、これは危険な設定です
        ので、標準インストレーションとしては提供しません。システムディレ
        クトリへのインストールを行なう場合には、自己責任でやってください。
    
3.1.セットアッププログラム

    idn wrapper をインストールするには"setup.exe" を実行します。セットアッ
    ププログラムは以下の処理を実行します。

    ファイルのインストール

        ディレクトリ「\Program Files\JPNIC\idn wrapper」 （ セットアップ
	時点で変更可能）以下に、idn wrapper を構成するファイルをコピーし
	ます。

    レジストリの設定
    
        HKEY_LOCAL_MACHINE\Software\JPNIC\IDN 以下に必要なレジストリキー、
        レジストリ値を作成、設定します。
	
	InstallDir	REG_SZ	"<インストールディレクトリ>"
	    idn wrapper のインストールディレクトリのパス名です。セット
	    アッププログラムはこのディレクトリにオリジナルのWINSOCK
	    DLL のコピーを作成します。idn wrapper のラッパー DLL は実
	    行時にこの DLL を参照します。

        ConfFile    REG_SZ      "<インストールディレクトリ>\idn.conf"
	    idn wrapper が国際化ドメイン名の変換処理に使用している
	    idnkit のコンフィギュレーションファイルの名前です。このファ
	    イルは国際化ドメイン名の処理に必要な各種のパラメータを設定
	    するためのものです。詳しくはファイルの内容をご覧ください。
	    この値は後述するコンフィギュレーションプログラムで変更する
	    ことができます。
    
        LogFile     REG_SZ      "<インストールディレクトリ>\idn_wrapper.log"
	    idn wrapper のログファイルの名前です。この値もコンフィギュ
	    レーションプログラムで変更することができます。
    
	LogLevel	DWORD	-1
	    ログレベルの指定です。デフォルトは -1 で、これは全くログを
	    出力しないという意味です。この値もコンフィギュレーションプ
	    ログラムで変更することができます。

        PerProg     キー
	
	    プログラム毎の設定値を格納するためのキーです。この下に、プロ
            グラムの実行モジュール名をキーとしてプログラム個別の設定が記
            録されます。設定される情報は以下の二つです。
	    
	    PerProg\<progname>\Where        REG_DWORD 変換位置
	    PerProg\<progname>\Encoding     REG_SZ    エンコーディング名

            エンコーディング名は通常コンフィギュレーションプログラムによっ
	    て設定されます。変換位置は、標準インストールでは不要です。シ
            ステムディレクトリへのインストールを行なった場合には、レジス
            トリエディタで環境に合わせて設定する必要があります。

    アイコンの作成
    
        コンフィギュレーションプログラムのアイコンを作成し、スタートメニュー
        に登録します。これによってコンフィギュレーションプログラムを起動
        することができます。

    アンインストールするには、コントロールパネルの「アプリケーションの追
    加と削除」で、「idn wrapper」 を選択して削除（「追加と削除」ボタン）
    します。

3.2.コンフィギュレーションプログラム

    コンフィギュレーションプログラムは、アプリケーションを特定してラップ
    したり、アプリケーションのラップを解除するためのツールです。

    起動すると以下のような画面が表示されます。

    ┌─┬─────────────────────────┬─┬─┬─┐
    │　│idn wrapper - Configuration                       │＿│□│×│
    ├─┴─────────────────────────┴─┴─┴─┤
    │          idn wrapper Configuration Program version X.X           │
    ├─────────────────────────────────┤
    │                  Wrapped Program                   ┌─────┐│
    │┌──────────────────────┬─┐│  Wrap..  ││
    ││                                            │∧│└─────┘│
    ││                                            ├─┤┌─────┐│
    ││                                            │  ││ Unwrap.. ││
    ││                                            │  │└─────┘│
    ││                                            │  │┌─────┐│
    ││                                            │  ││UnwrapAll.││
    ││                                            │  │└─────┘│
    ││                                            │  │┌─────┐│
    ││                                            │  ││RewrapAll.││
    ││                                            │  │└─────┘│
    ││                                            │  │┌─────┐│
    ││                                            │  ││  Log..   ││
    ││                                            │  │└─────┘│
    ││                                            │  │┌─────┐│
    ││                                            ├─┤│Advanced..││
    ││                                            │∨│└─────┘│
    │├─┬──────────────────┬─┼─┘┌─────┐│
    ││〈│                                    │〉│    │   Exit   ││
    │└─┴──────────────────┴─┘    └─────┘│
    └─────────────────────────────────┘

    リストボックスには、その時点でラップされているプログラムが表示されま
    す。最初に実行した場合には空になっています。

    プログラムをラップするには、"wrap"ボタンを押します。"wrap"ボタンを押
    すと以下のようなダイアログが表示されます。

    ┌─┬────────────────────────┬─┬─┬─┐
    │　│idn wrapper - Wrap Executable                   │＿│□│×│
    ├─┴────────────────────────┴─┴─┴─┤
    │          ┌───────────────────┐┌────┐│
    │ Program: │                                      ││Browse..││
    │          └───────────────────┘└────┘│
    │          ┌───┐                                            │
    │Encoding: │      │  ○Default  ○UTF-8                        │
    │          └───┘                                            │
    │           □ Force local DLL reference                         │
    ├────────────────────────────────┤
    │                                        ┌────┐┌────┐│
    │                                        │  wrap  ││ cancel ││
    │                                        └────┘└────┘│
    └────────────────────────────────┘

    最初に、ラップするプログラムの実行ファイル名を設定します。直接入力
    するか、ブラウズボタンでファイルを探してください。次にそのプログラ
    ムのローカルエンコーディングを指定します。通常は「Default」 でかま
    いません。プログラムが国際化エンコーディングに従っている場合にのみ
    「UTF-8」 を指示します。

    「Force local DLL reference」ボタンにより、ラップするプログラムの
    DLL の探索順序を変更することができます (ただし Windows95 にはこの
    機能がないため、このボタンも表示されません)。このボタンをチェック
    すると、たとえプログラムが別の場所の DLL を指定していても、常に実
    行ファイルがあるディレクトリの DLL が優先されるようになります。も
    しプログラムがうまくラップできない場合には、このボタンをチェックす
    るとうまくいくかもしれません。ただし同時に他の問題が発生する可能性
    もあります。

    最後に「wrap」ボタンを押せば、プログラムが、指定されたエンコーディ
    ングでラップされます。ラップされたプログラムは、最初のウィンドウの
    リストボックスに反映されます。

    idn wrapper をバージョンアップした場合には、ラップ用の DLL をアップ
    デートするために、プログラムを再ラップする必要があります。このために、
    現在ラップされているプログラムに対して再度ラップを行うための「rewrap
    all」ボタンが用意されています。

    プログラムに対するラップを解除するには、リストボックスで解除するプロ
    グラムを選択して、「unwrap」ボタンを押します。以下の確認用のダイアロ
    グが表示されますので、間違いがなければ「unwrap」ボタンを押してくださ
    い。

    ┌─┬────────────────────────┬─┬─┬─┐
    │　│idn wrapper - Unwrap Executable                 │＿│□│×│
    ├─┴────────────────────────┴─┴─┴─┤
    │          ┌─────────────────────────┐│
    │Program:  │                                                  ││
    │          └─────────────────────────┘│
    ├────────────────────────────────┤
    │                                        ┌────┐┌────┐│
    │                                        │ Unwrap ││ Cancel ││
    │                                        └────┘└────┘│
    └────────────────────────────────┘

    ラップが解除されると、そのプログラムは最初のウィンドウのリストボック
    スからも削除されます。

    プログラムに対して現在設定されているラップをすべて解除するための
    「unwrap all」ボタンも用意されています。

    ログの設定を行うには、"log" ボタンを押します。次のようなダイアログが
    表示されます。

    ┌─┬────────────────────────┬─┬─┬─┐
    │　│idn wrapper - Log Configuration                 │＿│□│×│
    ├─┴────────────────────────┴─┴─┴─┤
    │    Log Level: ○None ○Fatal ○Error ○Warning ○Info ○Trace  │
    │              ┌─────────────────┐┌────┐│
    │     Log File:│                                  ││Browse..││
    │              └─────────────────┘└────┘│
    │              ┌───┐ ┌───┐                             │
    │Log Operation:│ View │ │Delete│                             │
    │              └───┘ └───┘                             │
    ├────────────────────────────────┤
    │                                        ┌────┐┌────┐│
    │                                        │   OK   ││ Cancel ││
    │                                        └────┘└────┘│
    └────────────────────────────────┘

    ログレベルは次の中から選択することができます。
	None	ログを出力しない
	Fatal   致命的エラーのみ記録する
	Error	致命的でないエラーも記録する
	Warning	警告メッセージも記録する
	Info	その他の情報も記録する
	Trace	トレース出力も記録する
    ここにあげたログレベルの設定は、IDN ライブラリ (idnkit.dll) が出力する
    ログに対してのみ有効です。idn wrapper 自身が出力するログは ON/OFF
    しかできません。None を指定すると OFF に、それ以外のレベルを指定すると
    ON になります。

    このダイアログを用いて、ログファイルのパス名を指定することもできます。

    また、ログファイルの内容を表示させたり、ログファイルを削除することも
    可能です。

    ログレベルやログファイルの設定は、設定時にすでに動作しているプロセス
    には影響を与えないことに気をつけてください。

    "advanced" ボタンを押すと「advanced configuration」用のダイアログ
    が表示されます。このダイアログは上級ユーザのためのもので、適切なデ
    フォルトが設定されているために通常ユーザが変更する必要のないような
    基本的なパラメータを変更することができます。

    ┌─┬────────────────────────┬─┬─┬─┐
    │　│idn wrapper - Advanced Configuration            │＿│□│×│
    ├─┴────────────────────────┴─┴─┴─┤
    │                    IDN Wrapping Mode                           │
    │  ○Wrap both WINSOCK 1.1 and WINSOCK 2.0                       │
    │  ○Wrap only WINSOCK 1.1                                       │
    │  ○Wrap only WINSOCK 2.0                                       │
    │  ○Wrap only WINSOCK 2.0 if it exists.                         │
    │    Otherwise wrap only WINSOCK 1.1                             │
    ├────────────────────────────────┤
    │                    IDN Configuration                           │
    │              ┌─────────────────┐┌────┐│
    │     Log File:│                                  ││Browse..││
    │              └─────────────────┘└────┘│
    │              ┌───┐                                        │
    │              │ Edit │                                        │
    │              └───┘                                        │
    ├────────────────────────────────┤
    │                                        ┌────┐┌────┐│
    │                                        │   OK   ││ Cancel ││
    │                                        └────┘└────┘│
    └────────────────────────────────┘
    
    このダイアログを使用して、次の3種類の設定を行うことができます。

    Wrapping Mode
	ラップ方法を設定します。通常はデフォルトで設定されている項目を
	選択しておけばよいはずですが、問題が起ったときには別の項目にす
	ると動くようになるかもしれません。

    IDN Configuration
	コンフィギュレーションファイル名を指定します。また "Edit" ボタ
	ンを押すことにより、ファイルの内容を編集することも可能です。
    
4. 制限事項

4.1. DLL バージョン

    ラッパーDLL は、元のWINSOCK のDLL のバージョンに強く依存します。これ
    は、非公開のエントリも含めてすべてのエントリを提供する必要があるため
    です。このためWINSOCK DLL のバージョンが変わると、idn wrapper が動作
    しなくなる可能性があります。
    
    今回作成されたidn wrapper は、
    
        Win2000         (WINSOCK 1.1 + 2.0)
	WinME           (WINSOCK 1.1 + 2.0)

    で動作を確認しています。ただ、将来にわたって動作する保証はありません。

4.2. DNS, WINS, LMHOSTS

    Windows では、DNS だけではなく、WINSやLMHOSTS によってもドメイン名、
    ホスト名の解決が行なわれます。idn wrapper を使った場合には、ドメイン
    名の変換が、これらの方式へのディスパッチを行なう場所よりも上位層で行
    なわれるので、これらのすべての方式について、ドメイン名、ホスト名の変
    換が行なわれることになります。このため、Windows が、WINSやLMHOSTS を
    使っている場合には、予期しない問題が発生する可能性があります。これに
    ついては、idn wrapper を使う場合には、名前解決にDNS だけを使用するこ
    とをお勧めします。

3.3. ドメイン名以外の名前の解決

    WINSOCK 2.0 の名前解決API 

        WSALookupServiceBeginA
	WSALookupServiceNextA
	WSALookupServiceEnd

    は、ドメイン名以外でも使用できる、汎用の名前解決用のAPI として定義さ
    れています。現時点では、これらはもっぱらドメイン名の解決で使用されて
    いますが、他の名前(例えばサービス名)の解決にも使用できることになって
    います。

    idn wrapper は、名前の対象の如何にかかわらず、名前のエンコーディング
    を変換してしまうので、これらのAPI が、ドメイン名以外の解決に使われて
    いる場合には、問題を引き起こす可能性があります。

4.4. 名前解決API を使わないプログラム

    アプリケーションによっては、ドメイン名の解決にこれらのAPI を使用しな
    いものもあります。例えば、'nslookup'は、これらのAPI を使用しないで、
    直接DNS サーバと通信してしまいます。このようなアプリケーションについ
    ては、idn wrapper は役に立ちません。

4.5. 特定WINSOCK DLL にバインドされたアプリケーション

    アプリケーションによっては、標準のDLL サーチパスに従わないで、特定の
    パスのDLL にバインドされているものがあります。よく使われるプログラム
    の中では、Netscape Communicator がそうなっています。このようなプログ
    ラムについては、標準のインストール／コンフィギュレーションではラップ
    することはできません。
    
    このようなプログラムについて、どうしてもラップする必要があるなら、シ
    ステムディレクトリへのインストールを行なうことができます。ただし、こ
    のインストールは大変危険で、場合によってはシステムを再起動不能にして
    しまう可能性もあります。

5. レジストリ設定 - まとめ

5.1. レジストリの優先順位

    idn wrapper の設定情報は、HKEY_LOCAL_MACHINE、HKEY_CURRENT_USERの

        Software\JPNIC\IDN

    以下に格納されます。idn wrapperは最初にHKEY_LOCAL_MACHINEの設定を読
    み込み、HKEY_CURRENT_USER側にも設定があれば、これで上書きします。通
    常は、HKEY_LOCAL_MACHINE 側だけを設定します。ユーザ個別に異なった設
    定を使いたい場合のみ、HKEY_CURRENT_USERを設定するようにしてください。

    なお、コンフィギュレーションプログラムは HKEY_LOCAL_MACHINE の設定
    だけを読み書きするようになっています。
    
4.2. レジストリキー

    全体の共通の設定と、プログラム個別設定とがあります。

＿共通定義

	Software\JPNIC\IDN\InstallDir	 インストールディレクトリ
        Software\JPNIC\IDN\Where         変換位置
	                    0:WINSOCK1.1 WINSOCK2.0の両方で
                            1:WINSOCK2.0 があればそちらで
			    2:WINSOCK1.1 だけで
			    3:WINSOCK2.0 だけで
	Software\JPNIC\IDN\ConfFile	 idnkit の設定ファイル
        Software\JPNIC\IDN\LogLevel      ログレベル
        Software\JPNIC\IDN\LogFile       ログファイル

＿プログラム個別設定

    変換位置、およびプログラム側のエンコーディングはプログラム毎に特定す
    ることもできます。これらは、以下のキーハイブの下に、プログラム名をキー
    とする値で設定します。

        Software\JPNIC\IDN\PerProg\<name>\Where
        Software\JPNIC\IDN\PerProg\<name>\Encoding

    指定されていない場合には、
    
        Where       0       1.1、2.0 の両方で変換
	Encoding            プロセスのコードページ

    とみなします。

