```
docker run -itd --name mongo1 -p 27001:27017 --restart=always -v /data/mongo/mongo1/db:/data/db -v /data/mongo/mongo1/conf:/data/conf -v /data/mongo/mongo1/log:/data/log mongo --replSet "rs"
 
docker run -itd --name mongo2 -p 27002:27017 --restart=always -v /data/mongo/mongo2/db:/data/db -v /data/mongo/mongo2/conf:/data/conf -v /data/mongo/mongo2/log:/data/log mongo --replSet "rs"
 
docker run -itd --name mongo3 -p 27003:27017 --restart=always -v /data/mongo/mongo3/db:/data/db -v /data/mongo/mongo3/conf:/data/conf -v /data/mongo/mongo3/log:/data/log mongo --replSet "rs"
```

```
docker exec -it mongo1 mongo admin
```

```
var config={
     _id:"rs",
     members:[
         {_id:0,host:"你服务器ip:27001"},
         {_id:1,host:"你服务器ip:27002"},
         {_id:2,host:"你服务器ip:27003"}
]};
```

```
rs.initiate(config)
rs.status()
```
