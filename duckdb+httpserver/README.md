# DuckDB + httpserver 拡張

DuckDB を Web API 化する拡張を調査する。

<https://github.com/Query-farm/httpserver/>

## First touch

### 起動

実際に入力するコマンド:

```
duckdb
INSTALL httpserver FROM community;
LOAD httpserver;
SELECT httpserve_start('localhost', '9999', 'user:pass');
```

実行した際の出力例

```console
$ duckdb
DuckDB v1.4.3 (Andium) 136393de26
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
D INSTALL httpserver FROM community;
D LOAD httpserver;
D SELECT httpserve_start('localhost', '9999', 'user:pass');
┌───────────────────────────────────────────────────┐
│ httpserve_start('localhost', '9999', 'user:pass') │
│                      varchar                      │
├───────────────────────────────────────────────────┤
│ HTTP server started on localhost:9999             │
└───────────────────────────────────────────────────┘
D
```

2回目以降は `INSTALL httpserver FROM community;` は不要。
`~/.duckdb/extensions/{バージョン}/{プラットフォーム名}`
にてキャッシュされてる。

プラグインがネットからダウンロードされ、DuckDBにアタッチする。
`httpserve_start()` でバックグラウンドでWebサーバーが起動する。
上記の通り localhost の 9999 ポートで BASIC Auth で待ち受けている。

APIを試すべく以下を投げたら duckdb が segmentation fault で止まった。

```
$ curl -X POST -d "SELECT 'hello', version()" "http://user:pass@localhost:9999/"
{"'hello'":"hello","\"version\"()":"v1.4.3"}
```

MSYS2のMinGWの duckdb から実行したのが原因のようだ。
公式配布の duckdb.exe を使ったところ正常に動作した。

<details>
<summary>落ちた原因の調査と考察</summary>

おそらくプラグインは ducdb/duckdb 公式のバイナリと同じく
MSVC ランタイム向けにビルドされている。
MinGWのものはMinGWのランタイム向けにビルドされている。
duckdb 本体と httpserver プラグインで異なるランタイムを使おうとし、
問題が発生したと考えられる。

MSVC版にリンクされたDLLは以下の通り。

```
$ dumpbin -imports /d/bin/duckdb.exe  | grep -i dll
    WS2_32.dll
    RstrtMgr.DLL
    KERNEL32.dll
    MSVCP140.dll
    VCRUNTIME140.dll
    VCRUNTIME140_1.dll
    api-ms-win-crt-runtime-l1-1-0.dll
    api-ms-win-crt-heap-l1-1-0.dll
    api-ms-win-crt-environment-l1-1-0.dll
    api-ms-win-crt-string-l1-1-0.dll
    api-ms-win-crt-stdio-l1-1-0.dll
    api-ms-win-crt-filesystem-l1-1-0.dll
    api-ms-win-crt-utility-l1-1-0.dll
    api-ms-win-crt-time-l1-1-0.dll
    api-ms-win-crt-math-l1-1-0.dll
    api-ms-win-crt-convert-l1-1-0.dll
    api-ms-win-crt-locale-l1-1-0.dll
```

MinGW版にリンクされたDLLは以下の通り。

```
$ dumpbin -imports /mingw64/bin/duckdb.exe  | grep -i dll
    libgcc_s_seh-1.dll
    KERNEL32.dll
    msvcrt.dll
    libwinpthread-1.dll
    RstrtMgr.DLL
    libstdc++-6.dll
    WS2_32.dll
```

</details>

サーバーを止めるには `SELECT httpserve_stop();` を実行する。

### 認証

`user:pass` を空 (`''`) にすれば認証なしで動く。
すると http://localhost:9999/ をブラウザを開けば実験UIが表示される。

    SELECT httpserve_start('localhost', '9999', '');

`sometoken` のように `:` を含まなければ `X-API-Key` ヘッダーで渡すトークンによる認証になる。

    SELECT httpserve_start('localhost', '9999', 'securitykey');

```console
$ curl -X POST -H 'X-API-Key: securitykey' -d "SELECT 'hello', version()" "http://localhost:9999/"
{"'hello'":"hello","\"version\"()":"v1.4.4"}
```

### API endpoints

-   `/` - クエリー。
    GETなら `query` もしくは `q` パラメーターがあればクエリー実行モード、
    そうでなければプレイグラウンド(実験UI)モードになる。
    POSTはBODYがそのままクエリーとして扱われる。

    レスポンスの出力フォーマットは `default_format` パラメーター、
    `X-ClickHouse-Format` ヘッダー、
    `format` ヘッダーのいずれかで指定できる。
    指定可能な値は `JSONEachRow`, `JSONCompact`, `CSV`, `XML` で
    デフォルトは `JSONEachRow`

    <details>
    <summary>フォーマットの実例</summary>

    ```console
    $ curl -X POST -d "SELECT 'hello', version()" "http://localhost:9999/"
    {"'hello'":"hello","\"version\"()":"v1.4.4"}
    
    $ curl -X POST -d "SELECT 'hello', version()" "http://localhost:9999/?default_format=JSONCompact"
    {"meta":[{"name":"'hello'","type":"VARCHAR"},{"name":"\"version\"()","type":"VARCHAR"}],"data":[["hello","v1.4.4"]],"rows":1,"statistics":{"elapsed":0.0,"rows_read":0,"bytes_read":0}}
    
    $ curl -X POST -d "SELECT 'hello', version()" "http://localhost:9999/?default_format=CSV"
    'hello',"version"()
    hello,v1.4.4
    
    $ curl -X POST -d "SELECT 'hello', version()" "http://localhost:9999/?default_format=XML"
    <results>
      <row>
        <column name="&apos;hello&apos;">hello</column>
        <column name="&quot;version&quot;()">v1.4.4</column>
      </row>
    </results>
    ```

    </details>

-   `/ping` - 死活監視用。活きていれば 200 で単に `OK` と返ってくる

### 気になる点

-   プラグインのロード時になにかしらテレメトリーを送ってそう ([参考リンク](https://github.com/Query-farm/httpserver/blob/54e972c111221f4695d20234f6477cc88be4a54f/src/httpserver_extension.cpp#L610))
-   セッションが無く1つのインスタンスを共用するため、直前のAPIによるオブジェクトが残る
-   ローカルファイルを、duckdbを動かしているユーザー権限の範囲内ではあるが、無制限に弄れてしまう
-   デーモン化を想定していない
    -   どうやってバックグラウンド(プロセス)で動かすのが良いか
    -   `DUCKDB_HTTPSERVER_FOREGROUND=1` で動かせば tty は解放できそう
    -   監査ログがない。アクセスログは syslog を使ってる

## 内部解析

duckdb にバンドルされている httplib ヘッダーライブラリで HTTP サーバー機能を実装している。
std::thread で複数リクエストをさばいている。
並列数は `{コア数} - 1` か `8` の大きい方。

Windows は `DUCKDB_HTTPSERVER_FOREGROUND=1` をサポートしていない。

`DUCKDB_HTTPSERVER_SYSLOG=1` でログがでそう。
SYSLOGで対応できない範囲、フォーマット等を変えるのは、
ソースコードに手を入れて自分でビルドする必要がありそう。
- <https://github.com/Query-farm/httpserver/issues/16>
- <https://github.com/Query-farm/httpserver/pull/17>

環境変数は以下の通り。ベースパス以外は、Windowsでは機能しない。

-   `DUCKDB_HTTPSERVER_BASEPATH` - Webサーバーのエントリーポイントのベースパス。パスプレフィックスを与えたいときに利用する。デフォルトは `/`
-   `DUCKDB_HTTPSERVER_DEBUG` - `1` でデバッグ用のログを標準出力に出力する。デフォルトは未指定で無効化
-   `DUCKDB_HTTPSERVER_SYSLOG` - `1` でsyslogを有効化。デバッグログに劣後するので同時には利用できない。デフォルトは未指定で無効
-   `DUCKDB_HTTPSERVER_FOREGROUND` - `1` で `httpserve_start` をフォアグラウンド化。DuckDBプロセスの端末から追加コマンドを送れなくなる。デフォルトは未指定で無効

ログの内容は以下の通り

-   リモートアドレス
-   メソッド
-   パス
-   HTTPバージョン
-   ステータスコード
-   レスポンスサイズ
-   referer
-   user agent
-   forwarded for

## その他

-   Core Extension に UI があった <https://duckdb.org/docs/stable/core_extensions/ui>

## まとめ

-   テーブル残存やファイル操作など、環境汚染をどうするか?
    -   コンテナに閉じ込め定期的に再起動する?
-   監査ログはない。アクセスログの出力内容で十分か?
-   テレメトリーを送信する可能性を許容できるか?

コンテナを用いて、定期的に再起動が良さそう。
`DUCKDB_HTTPSERVER_DEBUG=1` でログは標準出力になる。

-   <https://duckdb.org/docs/stable/operations_manual/duckdb_docker>
-	<https://hub.docker.com/r/duckdb/duckdb>

サンプル: [docker-compose.yml](./docker-compose.yml)
