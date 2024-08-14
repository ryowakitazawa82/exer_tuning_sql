## Usage

1. DB コンテナを立てる

```
docker-compose up -d
```

2. コンテナの中に入る

```
docker exec -it mysql-container bash
```

3. mysql に接続する

```
mysql -u root -p # パスワードはroot
use demo;
```
