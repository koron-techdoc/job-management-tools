# Redash + pg\_duckdb の実験

## 準備

Redash用にDBのスキーマをセットアップ

    docker compose run --rm redash_server create_db

Redash の管理者ユーザーを追加

    docker compose run --rm redash_server manage users create_root --password {YOUR_PASSWORD} {YOUR@MAIL_ADDRESS.COM} Admin

テスト用のデータベース作成

    psql -U postgres -c 'CREATE DATABASE testdb'
    psql -U postgres -c 'CREATE EXTENSION pg_duckdb' testdb

起動

    docker compose up -d

Redash (<http://127.0.0.1:5000>) にアクセスして、以下のData Sourceを追加

-   Name: (任意: わかりやすい名前)
-   Host: postgres
-   Port: 5432
-   User: postgres
-   Database Name: testdb

S3にアクセスするために鍵を登録

    SELECT duckdb.create_simple_secret(
        type := 'S3',
        key_id := '{YOUR_ACCCESS_KEY}',
        secret := '{YOUR_SECRET}',
        region := '{YOUR_REGION}'
    )

Oct-2025 に Redash がネィティブで DuckDB に対応していた。
[redash/preview:25.12.0-dev](https://hub.docker.com/layers/redash/preview/25.12.0-dev/images/sha256-738a24ff697d5a3e6b6f79c173c33d822d86a32e1caa7ab55a69395103f619e4) 試したところ確かに利用できた。
DBインスタンスの再利用具合とか、
細かいところがわからないので留意が必要。
S3シークレットが共有されちゃう、もしくは毎回設定しなきゃかも?

対応が追加されたコミット: [1cc2008](https://github.com/getredash/redash/commit/1cc200843c9f8ec791f2019e80a28346a25565fa)

前述の「テスト用のデータベース作成」が要らなくなる。

