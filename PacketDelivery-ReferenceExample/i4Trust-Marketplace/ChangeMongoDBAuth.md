# Changing auth scheme of MongoDB

The Charging Backend of the BAE includes an older MongoDB client which uses an 
outdated auth schema.

Per default, the MongoDB 3.6 installed with this instructions is not using this 
older auth schema. In order to change the auth schema after the databases and users 
have been created, perform the following steps.


## Disable authorisation of MongoDB 
In MongoDB Helm values, set: 
```yaml
auth:
  enabled: false
```
Re-deploy the MongoDB via `helm upgrade`.

In case of Volume attach errors: Scale down and up the depyloment
```shell
kubectl -n marketplace scale deployment mongodb --replicas=0  # =1
```


## Update auth schema of MongoDB

Login to the running MongoDB pod, run the MongoDB client and note down the current users. Then Delete all users.
```shell
# Move into MongoDB pod
kubectl -n marketplace exec --stdin --tty mongodb-xyz -- /bin/bash
# Within MongoDB pod, connect to DB (auth has been disabled)
mongo
# Within mongo client:
> use admin
> db.system.users.find()
<LIST OF USERS IN JSON FORMAT> # Note this down
> db.system.users.remove({})
```

Update the auth schema

```shell
> db.system.version.update({_id: "authSchema"}, {$set: { "currentVersion" : 3}})
```

Restart DB by re-deploying or scaling deployment
```shell
kubectl -n marketplace scale deployment mongodb --replicas=0  # =1
```


## Recreate users

Move into MongoDB pod: 
```shell
kubectl -n marketplace exec --stdin --tty mongodb-xyz -- /bin/bash
```


```shell
# Within MongoDB pod, connect to DB (auth is still disabled)
mongo
# Within mongo client:
# Now recreate the users you have noted down before with their parameters. 
# Also add the passwords as specified during setting up the BAE.
# Below is an example.
> use charging_db;
> db.createUser(
     {
       user: "charging",
       pwd: "charging-password",
       roles: [
         {
           role: "readWrite",
           db: "charging_db"
         }
       ]
     });

> use belp_db
> db.createUser(
     {
       user: "belp",
       pwd: "belp-password",
       roles: [
         {
           role: "readWrite",
           db: "belp_db"
         }
       ]
     });
	 
> use admin
> db.createUser(
     {
       user: "root",
       pwd: "root-password",
       roles: [
         {
           role: "root",
           db: "admin"
         }
       ]
     });
```


## Enable authorisation of MongoDB

In MongoDB Helm values, set: 
```yaml
auth:
  enabled: true
```
Re-deploy the MongoDB via `helm upgrade`.

In case of Volume attach errors: Scale down and up the depyloment
```shell
kubectl -n marketplace scale deployment mongodb --replicas=0  # =1
```
