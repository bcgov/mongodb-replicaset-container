# TL;DR

This repo is all about a MongoDB 3.6 container to replace the deprecated image from RedHat. When started, it will want to create a replica set (RS) for high availability (HA); if you don't want HA, better to use the [official mongoDB image](https://hub.docker.com/_/mongo/)

# Introduction

This image was crafted to be a drop-in replacement for the now deprecated [RedHat mongoDB image](registry.redhat.io/rhscl/mongodb-36-rhel7). Unlike the RH image,
this image will only run in a high availability (HA) configuration. In your dev, test, or stage environment just run with a single member (pod) of the RS. In production, run with a minimum of 3 pods.

This image will run the official mongoDB RPM packages. While they are stable, the management scripts for this image (start/stop/management) are fairly new. As such, this image should be considered BETA.

# How To

This section will take you through how to build and run your very own HA mongoDB cluster, and, in the event of an emergency, offer some pro-tips on how to recover. 

This image is available from Artifactory:
https://artifacts.developer.gov.bc.ca/ui/repos/tree/General/plat-common-images%2Fmongodb-36-ha%2F1?projectKey=plat

If that image does not suit your needs for some reason, you may fork or clone this repo and build your own.

This image was previously made available through each cluster's `bcgov` namespace and that will continue to be the case for the time being.

## Build

If you need to build your own image, run the templates included with this repository against your **tools** namespace:

```console
oc process -f openshift/templates/build.yaml| \
oc apply -f -
```

After a few minutes you'll have a newly minted image you can deploy in place of the RedHat image, or as a new instance.

**Pro Tip 🤓**

Included in this repo is a `CronJob`. Use it to periodically rebuild the image so that it can automatically pickup any security updates and bug fixes for both the base image (RedHat UBI 8) or mongoDB.

## Run

This image will always start as a replica set. In your **dev** environment just spin up a single pod; in **test and prod** run three pods. If you want nothing to do with a replica set run the official mongoDB images.

This is a drop-in replacement for the now deprecated RedHat mongoDB image. It will need the following environment variables set:

| NAME                    | DESCRIPTION |
| :---------------------: |
| MONGODB_ADMIN_PASSWORD  | mongoDB `admin` user password.
| MONGODB_KEYFILE_VALUE   | keyfile value used for replica authentication. |
| MONGODB_REPLICA_NAME    | The name of the replica set. Use `rs0` if you're not sure what to do. |
| MONGODB_USER            | The user your **application** will use to access the database. |
| MONGODB_PASSWORD        | Password for the application user. |
| MONGODB_DATABASE        | The name of the application database; this is where the `MONGODB_USER` will live. |
| MONGODB_REPLICA_COUNT   | The number of pods in the replica set. |

Make things easy on yourself and just add all of these to a `kind: Secret` named `mongodb-creds` like this:

```yaml
apiVersion: v1
kind: Secret
metadata:
  labels:
    app: my-cool-app
  name: mongodb-creds
type: Opaque
data:
  MONGODB_ADMIN_PASSWORD: d29ybGQ0Mgo=
  MONGODB_ADMIN_USERNAME: d29ybGQK
  MONGODB_PASSWORD: aGVsbG8K
  MONGODB_USER: d29ybGQ0Mgo=
  MONGODB_KEYFILE_VALUE: Y2FrZTEyMwo=
  MONGODB_DATABASE: dHVya2V5Cg==
  MONGODB_REPLICA_NAME: Ymx1ZQo=
  MONGODB_REPLICA_COUNT: 3
```

Then import them all at once in your deployment like this:

```yaml
  envFrom: 
    - secretRef:
        name: mongodb-creds
```

**Pro Tip 🤓**

- Generate a keyfile value with the command `openssl rand -base64 756` if you're not sure what to use.


## Get Out of Trouble

This is general wisdom to help you run and mange a MongoDB HA replica set.

1. CLI

The `mongo` CLI is your friend. You can connect to the mongoDB directly with the admin or application account, in general, use the admin account for administration. If you're doing any work with the RS you **must** connect to the PRIMARY.

```console
mongo -u ${MONGODB_ADMIN_USERNAME} -p ${MONGODB_ADMIN_PASSWORD} --host ${MONGODB_SERVICE_NAME}
```

You may omit the `--host` parameter if you run this command on the primary (mongodb-0), as it will connect to the mongoDB instance running on the local server.

2. Replica Set Management

Learn about and manage the RS with the `rs` command set:

`rs.conf()`
View the replica set configuration.

`rs.status()`
Use this command to learn about your RS.

`rs.add()`
Use this command to add a RS member. In general, it's not required, the container deals with this.

`rs.remove()`
This one you may need from time to time. As you suspect, it will remove a member from the RS.

Review the online documentation for more commands as needed.

3. Shutdown

When the container (pod) needs to shut down it will also turn down the mongoDB instance in a sane way. If you want to do this manually, here are a few points to note:

- Run `db.runCommand({ replSetFreeze: numOfSeconds })` on SECONDARY to prevent it/them fom promoting to primary.
  
- Run `rs.stepDown(seconds)` on the PRIMARY. This will check to make sure at least one of the secondaries is sufficiently caught up to oplog before stepping down. Choose a reasonably long wait time depending on the size of your database and how far behind you think it is.

Also use this command if a secondary becomes a primary and you don't want that.

- Run `db.adminCommand({shutdown: 1,force: false})` on any member to gracefully shut down mongoDB.

4. Startup

When the container starts it will automatically initialize the RS and add itself as a member. If you want to bring it up manually, you can freeze a SECONDARY to stop it from promoting.

- Run `rs.freeze(seconds)` on all secondaries with a lengthy timeout (say, 1-2 minutes) to prevent them from promoting to PRIMARY.

# TODO

- Look into using `ping` to check for server status rather than `/tmp/initalized`:

```console
mongo --quiet --host 127.0.0.1 --port 27017 -u $MONGO_INITDB_ROOT_USERNAME -p $MONGO_INITDB_ROOT_PASSWORD --eval "db.adminCommand('ping')"
```

- Add mongo-shell to the image and use that to set up and configure the hosts: https://downloads.mongodb.com/compass/mongosh-1.0.0-linux-x64.tgz

