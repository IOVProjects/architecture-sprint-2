# mongo-sharding

Запускаем mongodb и приложение:

```shell
docker compose up -d
```

Инициализация реплики сервера конфигурации:

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
exit();
EOF
```

Инициализация реплики shard1:
```shell
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
rs.initiate(
    {
      _id : "shard1_replSet",
      members: [
        { _id : 0, host : "shard1:27018" }
      ]
    }
);
exit();
EOF
```

Инициализация реплики shard2:
```shell
docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
rs.initiate(
    {
      _id : "shard2_replSet",
      members: [
        { _id : 0, host : "shard2:27019" }
      ]
    }
  );
exit();
EOF
```

Инициализация mongos_router и наполнение его данными:
```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
sh.addShard( "shard1_replSet/shard1:27018");
sh.addShard( "shard2_replSet/shard2:27019");
sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" });
use somedb;
for(var i = 0; i < 1000; i++) db.helloDoc.insertOne({age:i, name:"ly"+i});
exit();
EOF
```

### Как проверить на локальной машине

Откройте в браузере http://localhost:8080
```
{
  "mongo_topology_type": "Sharded",
  "mongo_replicaset_name": null,
  "mongo_db": "somedb",
  "read_preference": "Primary()",
  "mongo_nodes": [
    [
      "mongos_router",
      27020]
  ],
  "mongo_primary_host": null,
  "mongo_secondary_hosts": [],
  "mongo_address": [
    "mongos_router",
    27020],
  "mongo_is_primary": true,
  "mongo_is_mongos": true,
  "collections": {
    "helloDoc": {
      "documents_count": 1000
    }
  },
  "shards": {
    "shard1": "shard1/shard1:27018",
    "shard2": "shard2/shard2:27019"
  },
  "cache_enabled": false,
  "status": "OK"
}
```

Откройте в браузере http://localhost:8080/helloDoc/count
```
{
  "status": "OK",
  "mongo_db": "somedb",
  "items_count": 1000
}
```

### Доступные эндпоинты
Список доступных эндпоинтов, swagger http://localhost:8080/docs