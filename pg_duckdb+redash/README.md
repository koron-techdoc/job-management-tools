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
