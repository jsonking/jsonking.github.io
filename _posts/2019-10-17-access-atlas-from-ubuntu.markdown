---
layout: post
title:  "Accessing Atlas from Ubuntu"
date:   2019-10-17 20:00:00 +0000
categories: mongodb
---
# Introduction

This post explains how to access an mongoDB Atlas cluster with the mongo shell from an Ubuntu Laptop.

Earlier this week I setup my first atlas cluster via <https://www.mongodb.com/cloud/atlas> and was quickly up and running with a free cluster and imported a sample dataset.

However when I tried to connect to the cluster with the mongo shell from my Ubuntu laptop - no joy.

The output in the shell reported a problem with the SSL certificate:
```
$ mongo "mongodb+srv://freecluster-vzni6.mongodb.net/test" --username jason --password xxx 
MongoDB shell version v4.0.12
connecting to: mongodb://freecluster-shard-00-02-vzni6.mongodb.net.:27017,freecluster-shard-00-00-vzni6.mongodb.net.:27017,freecluster-shard-00-01-vzni6.mongodb.net.:27017/test?authSource=admin&gssapiServiceName=mongodb&replicaSet=FreeCluster-shard-0&ssl=true
2019-10-17T21:36:22.874+0100 I NETWORK  [js] Starting new replica set monitor for FreeCluster-shard-0/freecluster-shard-00-02-vzni6.mongodb.net.:27017,freecluster-shard-00-00-vzni6.mongodb.net.:27017,freecluster-shard-00-01-vzni6.mongodb.net.:27017
2019-10-17T21:36:23.122+0100 E NETWORK  [js] SSL peer certificate validation failed: unable to get local issuer certificate
```

A quick search showed other users encountering this error too, but no immediate fix could be found. 

---
# The solution
The atlas ui has a conversations window that can be used to ask support questions.

I posted details of what i was trying to do and the error above. The response was not immediate: but within a couple of hours a support engineer answered and suspected that the operating system did not include the rootCA for the Atlas SSL certificate.

This is what was suggested as a workaround:
1. Download https://www.digicert.com/CACerts/DigiCertGlobalRootCA.crt
1. Convert the DER certificate to a PEM file

```
$ wget https://www.digicert.com/CACerts/DigiCertGlobalRootCA.crt --no-check-certificate
$ openssl x509 -inform der -in DigiCertGlobalRootCA.crt -out DigiCertGlobalRootCA.pem
```

Then include `--sslCAFile DigiCertGlobalRootCA.pem --ssl` flags in the connection command:
```
$ mongo "mongodb+srv://freecluster-vzni6.mongodb.net/test" --username jason --password xxx --sslCAFile DigiCertGlobalRootCA.pem --ssl
```

Hey Presto it works!
```
MongoDB shell version v4.0.12
connecting to: mongodb://freecluster-shard-00-02-vzni6.mongodb.net.:27017,freecluster-shard-00-01-vzni6.mongodb.net.:27017,freecluster-shard-00-00-vzni6.mongodb.net.:27017/test?authSource=admin&gssapiServiceName=mongodb&replicaSet=FreeCluster-shard-0&ssl=true
...
2019-10-17T21:59:54.035+0100 I NETWORK  [ReplicaSetMonitor-TaskExecutor] Successfully connected to freecluster-shard-00-02-vzni6.mongodb.net:27017 (1 connections now open to freecluster-shard-00-02-vzni6.mongodb.net:27017 with a 5 second timeout)
Implicit session: session { "id" : UUID("97089a86-6496-48ee-900d-ed4a17982199") }
MongoDB server version: 4.0.12
MongoDB Enterprise FreeCluster-shard-0:PRIMARY> 
```

---
# 4.2.x mongo shell

An alternative when using the mongo 4.2.x shell is with the `--tls -tlsCAFile` options instead as the ssl ones are now deprecated:

```
$ m 4.2.1
Activating 4.2.1
$ mongo "mongodb+srv://freecluster-vzni6.mongodb.net/test" --username jason --password xxx --tlsCAFile DigiCertGlobalRootCA.pem --tls

```

Output:
```
MongoDB shell version v4.2.1
connecting to: mongodb://freecluster-shard-00-01-vzni6.mongodb.net:27017,freecluster-shard-00-02-vzni6.mongodb.net:27017,freecluster-shard-00-00-vzni6.mongodb.net:27017/test?authSource=admin&compressors=disabled&gssapiServiceName=mongodb&replicaSet=FreeCluster-shard-0&ssl=true
...
Implicit session: session { "id" : UUID("a1b8cbac-53d9-4dc8-8358-5077d465875e") }
MongoDB server version: 4.0.12
WARNING: shell and server versions do not match
MongoDB Enterprise FreeCluster-shard-0:PRIMARY> 
```

Thanks to Sean and Clevy from the Ireland Cloud Triage Support Team.