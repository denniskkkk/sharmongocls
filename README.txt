#Mongodb Shard Cluster Setup guide
#May, 2020
 
#create this docker document
cat  <  EOF > docker-compose.yaml
version: '3.8'
services:
  mongors1n1:
    container_name: mongors1n1
    image: mongo
    deploy:
      restart_policy:
        condition: any      
    command: mongod --shardsvr --replSet mongors1 --dbpath /data/db --port 27017
    ports:
      - 27017:27017
    expose:
      - "27017"
    environment:
      TERM: xterm
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/lib/docker/mongo_shard/data1:/data/db

  mongors1n2:
    container_name: mongors1n2
    image: mongo
    deploy:
      restart_policy:
        condition: any      
    command: mongod --shardsvr --replSet mongors1 --dbpath /data/db --port 27017
    ports:
      - 27027:27017
    expose:
      - "27017"
    environment:
      TERM: xterm
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/lib/docker/mongo_shard/data2:/data/db
  mongors1n3:
    container_name: mongors1n3
    image: mongo
    deploy:
      restart_policy:
        condition: any      
    command: mongod --shardsvr --replSet mongors1 --dbpath /data/db --port 27017
    ports:
      - 27037:27017
    expose:
      - "27017"
    environment:
      TERM: xterm
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/lib/docker/mongo_shard/data3:/data/db
  mongors2n1:
    container_name: mongors2n1
    image: mongo
    deploy:
      restart_policy:
        condition: any      
    command: mongod --shardsvr --replSet mongors2 --dbpath /data/db --port 27017
    ports:
      - 27047:27017
    expose:
      - "27017"
    environment:
      TERM: xterm
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/lib/docker/mongo_shard/data4:/data/db
  mongors2n2:
    container_name: mongors2n2
    image: mongo
    deploy:
      restart_policy:
        condition: any      
    command: mongod --shardsvr --replSet mongors2 --dbpath /data/db --port 27017
    ports:
      - 27057:27017
    expose:
      - "27017"
    environment:
      TERM: xterm
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/lib/docker/mongo_shard/data5:/data/db
  mongors2n3:
    container_name: mongors2n3
    image: mongo
    deploy:
      restart_policy:
        condition: any      
    command: mongod --shardsvr --replSet mongors2 --dbpath /data/db --port 27017
    ports:
      - 27067:27017
    expose:
      - "27017"
    environment:
      TERM: xterm
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/lib/docker/mongo_shard/data6:/data/db
  mongocfg1:
    container_name: mongocfg1
    image: mongo
    deploy:
      restart_policy:
        condition: any      
    command: mongod --configsvr --replSet mongors1conf --dbpath /data/db --port 27017
    environment:
      TERM: xterm
    expose:
      - "27017"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/lib/docker/mongo_shard/config1:/data/db
  mongocfg2:
    container_name: mongocfg2
    image: mongo
    deploy:
      restart_policy:
        condition: any      
    command: mongod --configsvr --replSet mongors1conf --dbpath /data/db --port 27017
    environment:
      TERM: xterm
    expose:
      - "27017"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/lib/docker/mongo_shard/config2:/data/db
  mongocfg3:
    container_name: mongocfg3
    image: mongo
    deploy:
      restart_policy:
        condition: any      
    command: mongod --configsvr --replSet mongors1conf --dbpath /data/db --port 27017
    environment:
      TERM: xterm
    expose:
      - "27017"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/lib/docker/mongo_shard/config3:/data/db
  mongos1:
    container_name: mongos1
    image: mongo
    deploy:
      restart_policy:
        condition: any      
    depends_on:
      - mongocfg1
      - mongocfg2
    command: mongos --configdb mongors1conf/mongocfg1:27017,mongocfg2:27017,mongocfg3:27017 --port 27017 --bind_ip_all
    ports:
      - 27019:27017
    expose:
      - "27017"
    volumes:
      - /etc/localtime:/etc/localtime:ro
  mongos2:
    container_name: mongos2
    image: mongo
    deploy:
      restart_policy:
        condition: any      
    depends_on:
      - mongocfg1
      - mongocfg2
    command: mongos --configdb mongors1conf/mongocfg1:27017,mongocfg2:27017,mongocfg3:27017 --port 27017 --bind_ip_all
    ports:
      - 27020:27017
    expose:
      - "27017"
    volumes:
      - /etc/localtime:/etc/localtime:ro

  mongoshell:
    container_name: mongoshell
    image: mongo
    deploy:
      restart_policy:
        condition: any
    depends_on:
      - mongos1
      - mongos2
    command: bash -c "while true; do sleep 1; done"
    volumes:
      - /etc/localtime:/etc/localtime:ro
    secrets:
      - MONGO_SHARD_KEY
      - mongo_secret_key
      


secrets:
   MONGO_SHARD_KEY:
      external: true
   mongo_secret_key:
     file: ./secret.key 

EOF

##############################################################################
#start docker swarms
docker-compose up -d
#check output 
docker-compose logs

# configure config servers
docker exec -it mongocfg1 bash
#connect mongo client to config server
mongo
#run command to add config server cluster
rs.initiate({_id: "mongors1conf",configsvr: true, members: [{ _id : 0, host : "mongocfg1" },{ _id : 1, host : "mongocfg2" }, { _id : 2, host : "mongocfg3" }]})
#get server status
rs.status()
#wait and repeat server should change from SECONDARY TO PRIMARY

# build replica set server
docker exec -it mongors1n1 bash
#connect mongo client on replica server
mongo
#run command to add replica replSet
rs.initiate({_id : "mongors1", members: [{ _id : 0, host : "mongors1n1" },{ _id : 1, host : "mongors1n2" },{ _id : 2, host : "mongors1n3" }]})

#check status
rs.status ()

#wait and repeat server should change from SECONDARY TO PRIMARY

#introduce our shar to mongos routers
docker exec -it mongos1 bash

#connect client to routers 
mongo
#add shard servers to router
sh.add ("mongors1/mongors1n1")
sh.add ("mongors1/mongors1n2")
sh.add ("mongors1/mongors1n3")

sh.status ()
#if display all lines set verbose mode
sh.status (1)

#router server
#create db with namespace
use test
sh.enableSharding ("test")
# should reply
{
        "ok" : 1,
        "operationTime" : Timestamp(1600087527, 7),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1600087527, 7),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}

# create a collections
mongos> db.createCollection ("debuglist")
#reply
{
        "ok" : 1,
        "operationTime" : Timestamp(1600087564, 3),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1600087564, 3),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}

# then we have to create a shard field
sh.shardCollection ("develop.debuglist", {"shardfield" : "hashed"})
#reply
{
        "collectionsharded" : "develop.debuglist",
        "collectionUUID" : UUID("ad374138-b6ef-498c-a85f-e19c8b8c9427"),
        "ok" : 1,
        "operationTime" : Timestamp(1600087658, 9),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1600087658, 9),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}

#create some data and CRUD operation
#switch to database
use develop
#get current database
db.getName()
#insert some data
db.debuglist.insert ({ developer: "namehere" , bugdesc: "this is a bugger"})
#find all 
db.debuglist.find ()
{ "_id" : ObjectId("5f5f6704e1cdd72daa585a0b"), "developer" : "namehere", "bugdesc" : "this is a bugger" }
{ "_id" : ObjectId("5f5f6719e1cdd72daa585a0c"), "developer" : "namehere", "bugdesc" : "this is a bugger" }
{ "_id" : ObjectId("5f5f671ae1cdd72daa585a0d"), "developer" : "namehere", "bugdesc" : "this is a bugger" }
{ "_id" : ObjectId("5f5f671ae1cdd72daa585a0e"), "developer" : "namehere", "bugdesc" : "this is a bugger" }
{ "_id" : ObjectId("5f5f671ce1cdd72daa585a0f"), "developer" : "namehere", "bugdesc" : "this is a bugger" }

# remove _id from output, we have to provide projection 
db.debuglist.find ({} , {_id: 0})
{ "developer" : "namehere", "bugdesc" : "this is a bugger" }
{ "developer" : "namehere", "bugdesc" : "this is a bugger" }
{ "developer" : "namehere", "bugdesc" : "this is a bugger" }
{ "developer" : "namehere", "bugdesc" : "this is a bugger" }
{ "developer" : "namehere", "bugdesc" : "this is a bugger" }

#create a collection with schema
db.createCollection( "contacts", {
   validator: { $jsonSchema: {
      bsonType: "object",
      required: [ "name" ],
      properties: {
         name: {
            bsonType: "string",
            description: "must be a string and is required"
         },
         address: {
            bsonType : "string",
            description: "must be a string"
         },
         status: {
            enum: [ "accept", "reject" ],
            description: "only be one of enum values"
         }
      }
   } }
} )

#reply>
{
        "ok" : 1,
        "operationTime" : Timestamp(1600088581, 1),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1600088581, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}

db.contacts.insert ({ name: "mmmm" , address: "12323rer", status: "accept" })
WriteResult({ "nInserted" : 1 })

# but if we inser wrong data 
db.contacts.insert ({ name: "mmmm" , address: "12323rer", status: "ac" })

#reply>
WriteResult({
        "nInserted" : 0,
        "writeError" : {
                "code" : 121,
                "errmsg" : "Document failed validation"
        }
})



##########################################################
# system security
#create admin user
#create a user admin with password, with all database right
use admin
db.createUser(
  {
    user: "admin",
    pwd: passwordPrompt(), // or cleartext password
    roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
  }
)


# create some user
use test
db.createUser(
  {
    user: "user1",
    pwd:  passwordPrompt(),   // or cleartext password
    roles: [ { role: "readWrite", db: "test" },
             { role: "read", db: "develop" } ]
  }
)


#to login 
db.auth ("admin")
# return 
1 
# indicate login allow

#remote mongo client login-in
#
docker exec -it mongoshell  bash
# connect to router s1
mongo --host mongos1
#test some data
use develop
show collections



