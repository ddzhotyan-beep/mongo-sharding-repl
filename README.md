# pymongo-api

## Как запустить

Запускаем mongodb и приложение

```shell
docker compose up -d
```

## Настройка шардирования

Инициализация конфигурационного сервера

```shell
docker compose exec -T configSrv mongosh --port 27017 --quiet <<EOF 
rs.initiate(
  {
    _id : "config_server",
       configsvr: true,
    members: [
      { _id : 0, host : "configSrv:27017" }
    ]
  }
);
EOF
```

Инициализация шардов и по 3 реплики на каждый шард

```shell
docker compose exec -T shard1-a mongosh --port 27018 --quiet <<EOF 
rs.initiate(
    {
      _id : "shard1",
      members: [
        { _id : 0, host : "shard1-a:27018" },
       { _id :  1, host : "shard1-b:27019" },
       { _id :  2, host : "shard1-c:27020" }
      ]
    }
);
EOF 
```

```shell
docker compose exec -T shard2-a mongosh --port 27021 --quiet <<EOF 
rs.initiate(
    {
      _id : "shard2",
      members: [
        { _id : 0, host : "shard2-a:27021" },
       {  _id : 1, host : "shard2-b:27022" },
       {  _id : 2, host : "shard2-c:27023"}
      ]
    }
);
EOF 
```

Инициализация роутера и заполнения его тестовыми данными

```shell
docker compose exec -T mongos_router mongosh --port 27016 --quiet <<EOF 
sh.addShard( "shard1/shard1-a:27018");
sh.addShard( "shard2/shard2-a:27021");

sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )

use somedb
for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})

db.helloDoc.countDocuments()
EOF 
```

Проверка документов на репликах

```shell
docker compose exec -T shard1-a mongosh --port 27018 --quiet <<EOF 
use somedb
db.helloDoc.countDocuments()
EOF 
```

Для 1-ой реплики 1-го шарда будет результат 492

```shell
docker compose exec -T shard1-b mongosh --port 27019 --quiet <<EOF 
use somedb
db.helloDoc.countDocuments()
EOF 
```

Для 2-ой реплики 1-го шарда будет результат 492

```shell
docker compose exec -T shard2-a mongosh --port 27021 --quiet <<EOF 
use somedb
db.helloDoc.countDocuments()
EOF 
```

Для 1-ой реплики 2-го шарда будет результат 508

```shell
docker compose exec -T shard2-b mongosh --port 27022 --quiet <<EOF 
use somedb
db.helloDoc.countDocuments()
EOF 
```

Для 2-ой реплики 2-го шарда будет результат 508

Аналогично можно проверить для 3-й реплики по каждому шарду.