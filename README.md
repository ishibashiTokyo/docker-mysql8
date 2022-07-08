# docker-mysql8

Dockerを利用してMySQL8環境を用意する。

## 使い方

.env でポートを変えれば複数起動可能

@todo レプリカ設定の YAML も作成する。

## 起動

```shell
$ chmod 777 logs/ mysql/ logs/bin_log/
$ docker-compose up -d
Starting mysql_3316 ... done
$ docker-compose ps
   Name                Command             State                          Ports
------------------------------------------------------------------------------------------------------
mysql_3316   docker-entrypoint.sh mysqld   Up      0.0.0.0:3316->3306/tcp,:::3316->3306/tcp, 33060/tcp
```

基本操作

```shell
# コンテナの停止
$ docker-compose stop
# コンテナの開始
$ docker-compose start
# コンテナ一覧
$ docker-compose ps -a
# ログ
$ docker compose logs
```

## ホストから Docker の MySQL に接続

Unix ソケットは使えないので注意

```shell
$ mysql -u db_user -p -h localhost -P 3316 --protocol=tcp
```

## ディレクトリ構成

```shell
$ tree ./
./
├── docker-compose.yml
├── .env    # ymlで使用する定数ファイル
├── my.cnf  # MySQL設定ファイル
├── logs    # ログ保存先
│   ├── bin_log   # レプリケーション設定使用時のバイナリログ
│   │   ├── repl-bin.000001
│   │   └── repl-bin.index
│   ├── mysql-error.log
│   ├── mysql-query.log
│   └── mysql-slow.log
└── mysql   # MySQLスキーマの物理保存先
    ├── mysql.sock -> /var/run/mysqld/mysqld.sock
    ├── performance_schema
    └── ...
```

## AWS での速度検証

### 検証コマンド

同じインスタンスを別々に構築し、標準インストールしたものと、Docker 上で動いているものそれぞれで`mysqlslap`コマンドを利用して実行時間を計測。

mysqlslap は`-h`に localhost を指定すると`--protocol=TCP`を使っていても Unix ソケット接続を使うので注意

使用したコマンド：

```shell
mysqlslap -u root -pdb_user_password -P 3316 -h 127.0.0.1 --protocol=TCP \
 --auto-generate-sql \
 --engine=innodb \
 --auto-generate-sql-add-autoincrement \
 --number-int-cols=10 \
 --number-char-cols=10 \
 --iterations=10 \
 --concurrency=5 \
 --auto-generate-sql-write-number=1000 \
 --number-of-queries=1000 \
 --auto-generate-sql-load-type=read
```

パラメータオプション：

- `auto-generate-sql` ... SQL 自動発行
- `engine=innodb` ... データベースエンジン：innodb
- `auto-generate-sql-add-autoincrement` ... AI 追加
- `number-int-cols=10` ... INT カラム 10 個
- `number-char-cols=10` ... CHAR カラム 10 個
- `iterations=10` ... 試行回数：10 回
- `concurrency=5` ... クライアント数：5
- `auto-generate-sql-write-number=1000` ... 初期時 1000 行のデータ
- `number-of-queries=1000` ... クエリ実行回数
- `auto-generate-sql-load-type=read` ... クエリ実行種類
  - read(テーブルスキャン)
  - write(テーブルへの挿入)
  - key(主キー読み取り)
  - mixed(挿入とテーブルスキャンを半々)
  - update(更新)

### 結果

単位：秒

| 計測オプション / インスタンス | m6i.large - Docker | r6i.large - Docker MySQL | r6i.large - 標準インストール MySQL |
| ----------------------------- | ------------------ | ------------------------ | ---------------------------------- |
| read                          | 4.498              | 4.288                    | 2.843                              |
| write                         | 1.673              | 1.493                    | 1.196                              |
| key                           | 0.151              | 0.15                     | 0.092                              |
| mixed                         | 0.847              | 0.828                    | 0.603                              |
| update                        | 1.52               | 1.448                    | 1.249                              |
