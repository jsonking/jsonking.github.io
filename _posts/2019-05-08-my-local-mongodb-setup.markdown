---
layout: post
title:  "My local mongoDB setup"
date:   2019-05-08 20:08:00 +0000
categories: mongodb
---
# Introduction

This post introduces two tools that I use when working with mongoDB on my local development machine.
- m: <https://github.com/aheckmann/m>. A mongoDB version manager. Allows you to easily install different versions of MongoDB. Quickly switch between releases to reproduce bugs, verify fixes or try out the latest features.
- mlaunch: <https://github.com/rueckstiess/mtools/>. A utility to quickly set up different test environments. It is part of mtools (a collection of mongo helper tools).

---

# Setting up m
m requires npm/node to be installed first.
To avoid permissions issues requiring sudo when installing modules we first install node via the nvm method:
<https://github.com/nvm-sh/nvm>
```
$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
$ nvm install node
```

Then create the following directories and set permissions before installing m:
```
$ sudo mkdir /usr/local/m
$ sudo chown `whoami` /usr/local/m
$ sudo chown `whoami` /usr/local/bin
$ npm install -g m
```

Once setup we are ready to download and install mongoDB. Let's install the latest stable version:
```
$ m stable
MongoDB version 4.0.9 is not installed.
Installation may take a while. Would you like to proceed? [Y/n] Y
```
Once complete let's verify by launching the shell:
```
$ mongo --nodb
MongoDB shell version v4.0.9
Welcome to the MongoDB shell.
...
> exit
```
The [homepage](https://github.com/aheckmann/m) describes how to use m in detail. Below are two commands I use frequently:
```
# list the available versions
$ m list 

# install and switch to a specific version
$ m 3.6.12 
```

Next let's setup mlaunch and create some environments.

---

# mlaunch
The steps for installing mtools are documented at <https://github.com/rueckstiess/mtools/blob/develop/INSTALL.md>. I used the following approach (as mlaunch wasn't working after using `sudo apt install python-pip`):
```
$ git clone git://github.com/rueckstiess/mtools.git
$ cd mtools
$ sudo python setup.py install
```
Once installed, it's time to create our first environment. 
Let's create a standalone instance:
```
$ mlaunch init --single
launching: "mongod" on port 27017

$ mongo
MongoDB shell version v4.0.9
connecting to: mongodb://127.0.0.1:27017/?gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("529a6e9f-f504-4e66-9b2f-dc52011915e6") }
MongoDB server version: 4.0.9
Server has startup warnings: 
...
> 
```

Next let's insert a document to verify everything is working:
```
> db.foo.insert({_id:"test"})
WriteResult({ "nInserted" : 1 })
> db.foo.find()
{ "_id" : "test" }
> show collections
foo
> exit
```

---

# Under the hood
Let's take a look under the hood:
```
$ ps -ef | grep mongod
jason     5814  1761  1 19:52 ? 00:00:00 
mongod --dbpath /home/jason/data/db --logpath /home/jason/data/mongod.log --port 27017 --fork --wiredTigerCacheSizeGB 1
```
We can see that mongod is running. `--dbpath` is `/home/jason/data/db` which is the directory where the data files are stored. The mongod log file at `/home/jason/data/mongod.log` is useful to debug what the server is doing.

In the `data` directory there is a configuration file `.mlaunch_startup`. This can be edited; for example to change the default port the instance starts on.

Finally, let's stop the cluster:
```
$ mlaunch stop
sent signal 15 to 1 process.
```
The next time you want to start this instance just run `mlaunch start`.  
There is good help for each of the mlaunch sub-commands e.g. `mlaunch init -h`.

---

# Creating a sharded cluster

Next we shall create a 3-shard cluster with a single node per shard (a 3x1 cluster). This is the kind of environment I use when working with sharding related features.

To change from the default `./data/` directory mlaunch commands accept the ``--dir DIR`` argument. This gives flexibility over where the files are stored. Here we will locate our environment in the `./sharded/` directory.

```
$ mlaunch init --dir sharded --sharded 3 --nodes 1 --replicaset
launching: "mongod" on port 27018
launching: "mongod" on port 27019
launching: "mongod" on port 27020
launching: config server on port 27021
replica set 'configRepl' initialized.
replica set 'shard01' initialized.
replica set 'shard02' initialized.
replica set 'shard03' initialized.
launching: mongos on port 27017
adding shards. can take up to 30 seconds...
```
5 nodes have been started: 3 mongod's, a config server and a mongos router.
Our connection to the cluster is through the mongos started on default port `27017`:
```
$ mongo
mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("5cc21bd3123f6f02d9c8a0da")
  }
  shards:
        {  "_id" : "shard01",  "host" : "shard01/localhost:27018",  "state" : 1 }
        {  "_id" : "shard02",  "host" : "shard02/localhost:27019",  "state" : 1 }
        {  "_id" : "shard03",  "host" : "shard03/localhost:27020",  "state" : 1 }
  active mongoses:
        "4.0.9" : 1
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours: 
                No recent migrations
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
```
Yay! We have sucessfully created and connected to a sharded cluster.

Once finished remember to shut everything down:
```
$ mlaunch stop --dir sharded
sent signal 15 to 5 processes.
```

---

# Final Words
The documentation and help describes in more detail what these tools can do.  
I've used these on Linux, macOS and Windows 10 under Windows Linux Subsystem (WLS) so you there are options no matter what your preferred setup is.

Thank you for reading this post - I hope it will be useful for your adventures with mongoDB!  
Feel free to ask any questions or send me feedback - I also welcome any additional tips you have.