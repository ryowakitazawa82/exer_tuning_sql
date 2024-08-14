## Usage

1. DB コンテナを立てる

```
docker-compose up -d
```

2. コンテナの中に入る

```
docker exec -it mysql-container bash
```

3. mysql に接続

```
mysql -u root -p # パスワードはroot
```

4. 後はdemoデータベースを自由に操作
```
use demo;
```
