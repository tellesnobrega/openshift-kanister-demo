# kanister-demo on Centos 7

### Setup Environment

Start your environment setup by following this [tutorial](https://github.com/tellesnobrega/kanister-demo/blob/master/setup-environment.md)

### Add some content to WordPress

Now that the environment is set up you can add a post to WordPress

### Prepare the backup setup

#### Create the kanister profile

This profile is created to manage needed information to connect to databases and S3 object storage.
```
helm install kanister/profile -g --set defaultProfile=true \
                                 --namespace kanister \
                                 --set location.type='s3Compliant' \
                                 --set location.bucket='kubedemo' \
                                 --set location.endpoint='<URL>:<PORT>' \
                                 --set aws.accessKey='<ACCESS_KEY>' \
                                 --set aws.secretKey='<SECRET_KEY>' \
                                 --set mysqlRootPassword='<MYSQL_ROOT_PASSWORD>'

```

#### Create the wordpress and mysql blueprint
```
kubectl -n kanister create -f kanister-demo/wordpress-blueprint.yaml
kubectl -n kanister create -f kanister-demo/mysql-blueprint.yaml
```

#### Run the backup actionset for the mysql container
```
kanctl create actionset --action backup \
                        --namespace kanister \
                        --blueprint mysql-blueprint \
                        --deployment wordpress/wordpress-mysql \
                        --profile default-profile
```
Annotate the backup name

#### Run the backup actionset for the wordpress container
```
kanctl create actionset --action backup \
                        --namespace kanister \
                        --blueprint wordpress-blueprint \
                        --deployment wordpress/wordpress \
                        --profile default-profile
```
Annotate the backup name

#### Destroy WordPress deployment

With Kanister we can't recover a lost namespace, but we can rebuild an environment with a backup from a destroyed one.
We will recover two different scenarios:

1 - We will break the database and also wordpress and recover from backup.

2 - We will delete the wordpress namespace, recreate it and recover from backup.

##### Scenario 1

Go into the mysql container and delete the wp-posts tables from wordpress database.

Go into the wordpress container and delete wp-content/ folder.

Refreshing the Wordpress page you will see that it is not working anymore.

###### Run restore command
```
kanctl create actionset --action restore --namespace kanister --from <MYSQL_BACKUP>
kanctl create actionset --action restore --namespace kanister --from <WORDPRESS_BACKUP>
```

The backup of the WordPress container will take a couple minutes because it stops the container, mounts a restore container
and recreates it.

Once the wordpress pod is running again, refresh the wordpress and make sure wordpress is completely recovered.


##### Scenario 2

```
kubectl delete ns wordpress
kubectl create ns wordpress
kubectl create -n wordpress secret generic mysql-pass --from-literal=password=<MYSQL_ROOT_PASSWORD>
kubectl -n wordpress create -f kanister-demo/mysql-deployment.yaml
kubectl -n wordpress create -f kanister-demo/wordpress-deployment.yaml
```

###### Run restore command
```
kanctl create actionset --action restore --namespace kanister --from <MYSQL_BACKUP>
kanctl create actionset --action restore --namespace kanister --from <WORDPRESS_BACKUP>
kubectl -n wordpress patch svc wordpress -p '{"spec": { "type": "NodePort", "ports": [ { "nodePort": <PORT>, "port": 80, "protocol": "TCP", "targetPort": 80 } ] } }'
```
Replace <PORT> with the port from previous wordpress deployment. This is needed because wordpress keeps url information
on the database and after the restore minikube gives the service a new PORT. Patching this solves this redirecting issue.

Refresh WordPress and make sure it is working as expected.
