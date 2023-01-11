---
title: "Data migration from CHT 3.x to CHT 4.x"
linkTitle: "Data migration to 4.x"
weight: 10
description: >
  Guide to migrate existent data from CHT 3.x to CHT 4.x
aliases:
relatedContent: >
---

The hosting architecture differs entirely between CHT-Core 3.x and CHT-Core 4.x. Migrating data from an existing instance running CHT 3.x requires a few manual steps. 
This guide will present the required steps while using a migration helping tool, called `couchdb-migration`. This tool interfaces with CouchDb, to update shard maps and database metadata. 
Using this tool is not required, and the same result can be achieved by calling CouchDb endpoints directly. [Consult CouchDB documentation for details about moving shards](https://docs.couchdb.org/en/3.2.2-docs/cluster/sharding.html#moving-a-shard). 

### 1. Install CHT data migration tool

Open your terminal and run these commands. They will create a new directory, download a docker compose file and download the required docker image. 
```shell
mkdir -p ~/couchdb-migration/ 
cd ~/couchdb-migration/ 
curl -s -o ./docker-compose.yml https://github.com/medic/couchdb-migration/blob/main/docker-compose.yml
docker-compose up
```

For the following steps, the tool needs access to your CouchDb installation. To allow this access, you will need to provide a URL to your CouchDB installation that includes authentication. 
Additionally, if your CouchDb runs in docker, the tool needs to be added to the same docker network in order to access protected endpoints:

```shell
export COUCH_URL=http://<authentication>@<host-ip>:<port>
```
or 
```shell
export CHT_NETWORK=<docker-network-name>
export COUCH_URL=http://<authentication>@<docker-service-name>:<port>
```

For simplicity, you could store these required values in an `.env` file: 
```shell
cat > ${HOME}/couchdb-migration/.env << EOF
CHT_NETWORK=<docker-network-name>
COUCH_URL=http://<authentication>@<docker-service-name>:<port>
EOF
```

### 2. Prepare CHT-Core 3.x installation for upgrading
To ensure no changes happen to your CouchDB data in your CHT 3.x server, stop the API by getting shell on your Medic OS container and calling `/boot/svc-stop medic-api`.


To minimize downtime when upgrading, it's advised to prepare the 3.x installation for the 4.x upgrade, and pre-index all views that are required by 4.x.

The migration tool provides a command which will download all 4.x views to your 3.x CouchDb, and initiate view indexing. `<desired CHT version>` is any version at or above `4.0.0`:

```shell
cd ~/couchdb-migration/ 
docker-compose run couch-migration pre-index-views <desired CHT version>
```

Once view indexing is finished, proceed with the next step.

{{% alert title="Note" %}} If this step is omitted, 4.x API will fail to respond to requests until all views are indexed. Depending on the size of the database, this could take many hours, or even days. {{% /alert %}} 

### 3. Save existent CouchDb configuration

Some CouchDb configuration values must be ported from existent CouchDb to the 4.x installation. Store them in a safe location before shutting down 3.x CouchDb.
Use the migration tool to obtain these values:
```shell
cd ~/couchdb-migration/ 
docker-compose run couch-migration get-env
```

##### a. CouchDB secret 
Used in encrypting all CouchDb passwords and session tokens.
##### b. CouchDb server uuid
Used in generating replication checkpointer documents, which track where replication progress between every client and the server, and ensure that clients don't re-download or re-upload documents.

### 4. Locate and backup CouchDb Data folder
a) If running in MedicOS, [CouchDb data folder]({{< relref "apps/guides/hosting/3.x/self-hosting#backup" >}}) can be found at `/srv/storage/medic-core/couchdb/data`.

b) If running a custom installation of CouchDb, data would be typically stored at `/opt/couchdb/data`.

### 5. Launch 4.x CouchDb installation

Depending on your project scalability needs and technical possibilities, you must decide whether you will deploy CouchDb in a single node or in a cluster with multiple nodes.
Please consult this guide about clustering and horizontal scalability to make an informed decision. <insert link>

{{% alert title="Note" %}} You can start with single node and then change to a cluster. This involves running the migration tool again to distribute shards from the existent node to the new nodes. {{% /alert %}}

Depending on your choice, follow the instructions that match your deployment below: 

#### Single node

a) Download 4.x single-node CouchDb docker-compose file:
```shell
mkdir -p ~/couchdb-single/ 
cd ~/couchdb-single/ 
curl -s -o ./docker-compose.yml https://staging.dev.medicmobile.org/_couch/builds_4/medic:medic:<desired CHT version>/docker-compose/cht-couchdb.yml
```
a) Make a copy of the 3.x CouchDb data folder from **step 4**.

b) Set the correct environment variables:
```shell
cat > ${HOME}/couchdb-single/.env << EOF
COUCHDB_USER=<admin>
COUCHDB_PASSWORD=<password>
COUCHDB_SECRET=<COUCHDB_SECRET from step 3>
COUCHDB_UUID=<COUCHDB_UUID from step 3>
COUCHDB_DATA=<absolute path to folder created in step 5.a>
EOF
```
c) Start 4.x CouchDb and wait until it is up. You'll know it is up when the `docker-compose` call exits without errors and logs `CouchDb is Ready`.
```shell
cd ~/couchdb-single/ 
docker-compose up -d
cd ~/couchdb-migration/ 
docker-compose run couch-migration check-couchdb-up
```

d) Change metadata to match the new CouchDb node
```shell
cd ~/couchdb-migration/ 
docker-compose run couch-migration move-node
```
w) Run the `veryfy` command to check whether the migration was successful.
```shell
docker-compose run couch-migration verify
```
If all checks pass, you should see a message `Migration verification passed`. It is then safe to proceed with starting CHT-Core 4.x, using the same environment variables you saved in `~/couchdb-single/.env`.

#### Multi node

a) Download 4.x clustered CouchDb docker-compose file:
```shell
mkdir -p ~/couchdb-cluster/ 
cd ~/couchdb-cluster/ 
curl -s -o ./docker-compose.yml https://staging.dev.medicmobile.org/_couch/builds_4/medic:medic:<desired CHT version>/docker-compose/cht-couchdb-clustered.yml
```
b) Create a data folder for every one of the CouchDb nodes.
If you were going to a 3 cluster node, this would be:
```shell
mkdir -p ~/couchdb-data/main
mkdir -p ~/couchdb-data/secondary1
mkdir -p ~/couchdb-data/secondary2
```

c) Copy the 3.x CouchDb data folder into `~/couchdb-data/main`, which will be your main CouchDb node. This main node will create your cluster and the other secondary nodes will be added to it. In `main`'s environment variable file, define `CLUSTER_PEER_IPS`. In all other secondary nodes, declare the `COUCHDB_SYNC_ADMINS_NODE` variable instead.

d) Create a `shards` and a `.shards` directory in every secondary node folder. 

e) Set the correct environment variables:
```shell
cat > ${HOME}/couchdb-cluster/.env << EOF
COUCHDB_USER=<admin>
COUCHDB_PASSWORD=<password>
COUCHDB_SECRET=<COUCHDB_SECRET from step 3>
COUCHDB_UUID=<COUCHDB_UUID from step 3>
COUCHDB1_DATA=<absolute path to main folder created in step 5.a>
COUCHDB2_DATA=<absolute path to secondary1 folder created in step 5.a>
COUCHDB3_DATA=<absolute path to secondary2 folder created in step 5.a>
EOF
```

f) Start 4.x CouchDb and wait until it is up. You'll know it is up when the `docker-compose` call exits without errors and logs `CouchDb Cluster is Ready`.
```shell
cd ~/couchdb-cluster/ 
docker-compose up -d
cd ~/couchdb-migration/ 
docker-compose run couch-migration check-couchdb-up <number of nodes>
```

g) Generate the shard distribution matrix and get instructions for final shard locations. 
```shell
cd ~/couchdb-migration/ 
shard_matrix=$(docker-compose run couch-migration generate-shard-distribution-matrix)
docker-compose run couch-migration shard-move-instructions $shard_matrix
``` 

h) Follow the instructions from the step above and move the shard files to the correct location, according to the shard distribution matrix. 

Example of moving one shard from one node to another:

```shell
/couchdb_data_main
   /.delete
   /_dbs.couch
   /_nodes.couch
   /_users.couch
   /.shards
     /00000000-15555554
     /2aaaaaaa-3ffffffe
     /3fffffff-55555553
     /6aaaaaa9-7ffffffd
     /7ffffffe-95555552
     /15555555-2aaaaaa9
     /55555554-6aaaaaa8
     /95555553-aaaaaaa7
   /shards
     /00000000-15555554
     /2aaaaaaa-3ffffffe
     /3fffffff-55555553
     /6aaaaaa9-7ffffffd
     /7ffffffe-95555552
     /15555555-2aaaaaa9
     /55555554-6aaaaaa8   
     /95555553-aaaaaaa7
     
/couchdb_data_secondary
   /.shards   
   /shards
```
After moving two shards: `55555554-6aaaaaa8` and `6aaaaaa9-7ffffffd`
```shell
/couchdb_data_main
   /.delete
   /_dbs.couch
   /_nodes.couch
   /_users.couch
   /.shards
     /00000000-15555554
     /2aaaaaaa-3ffffffe
     /3fffffff-55555553     
     /7ffffffe-95555552
     /15555555-2aaaaaa9     
     /95555553-aaaaaaa7
   /shards
     /00000000-15555554
     /2aaaaaaa-3ffffffe
     /3fffffff-55555553     
     /7ffffffe-95555552
     /15555555-2aaaaaa9
     /95555553-aaaaaaa7
          
/couchdb_data_secondary
   /.shards   
     /6aaaaaa9-7ffffffd
     /55555554-6aaaaaa8
   /shards
     /6aaaaaa9-7ffffffd
     /55555554-6aaaaaa8
```
i) Change metadata to match the new shard distribution. We declared `$shard_matrix` in step "g" above, so it is still set now:

```shell
docker-compose run couch-migration move-shards $shard_matrix
``` 

j) Run the `veryfy` command to check whether the migration was successful. 
```shell
docker-compose run couch-migration verify
```
If all checks pass, you should see a message `Migration verification passed`. It is then safe to proceed with starting CHT-Core 4.x, using the same environment variables you saved in `~/couchdb-cluster/.env`.