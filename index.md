# Running Backup & Restore on a Kubernetes environment using Kanister and Minio

The goal of this post is to provide a step-by-step tutorial on how to set up, backup and restore a WordPress application running on Minikube, using Kanister for Backup and Restore and Minio as S3-like Object Storage.

This is part of a series of introductions to backup and restore tools I'm playing with. If you are interested also in Velero, check [Velero Blog Post](https://tellesnobrega.github.io/velero-demo/)

## Setting up the Environment

### Install docker
```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
```

### Running minio container
```
docker pull minio/minio
docker run -p 9000:9000 --name minio -e "MINIO_ACCESS_KEY=minio" -e "MINIO_SECRET_KEY=minio123" -v /mnt/data:/data minio/minio server /data
```
### Install Kubectl
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
yum install -y kubectl
```

### Install minikube
```
sudo yum -y install conntrack
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-1.9.2-0.x86_64.rpm
sudo rpm -ivh minikube-1.9.2-0.x86_64.rpm
minikube start --driver=none
```
### Install Helm
```
yum -y install openssl
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Install GO
```
wget https://dl.google.com/go/go1.14.4.linux-amd64.tar.gz
tar -C /usr/local -xzf go$VERSION.$OS-$ARCH.tar.gz
export PATH=$PATH:/usr/local/go/bin
```

### Install Kanctl
```
curl https://raw.githubusercontent.com/kanisterio/kanister/master/scripts/get.sh -o get.sh
sed -i 's/shasum/sha256sum/g' get.sh
sed -i 's/sha256sum -a 256/sha256sum/g' get.sh
chmod +x get.sh
./get.sh
```

### Install Kanister Operator
```
helm repo add kanister https://charts.kanister.io/
kubectl create ns kanister
helm install myrelease --namespace kanister kanister/kanister-operator --set image.tag=0.29.0
```

### Clone this repo
```
git clone https://github.com/tellesnobrega/kanister-demo.git
```

### Deploy wordpress application
```
kubectl create ns wordpress
kubectl create -n wordpress secret generic mysql-pass --from-literal=password=<MYSQL_ROOT_PASSWORD>
kubectl create -n wordpress -f kanister-demo/mysql-deployment.yaml
kubectl create -n wordpress -f kanister-demo/wordpress-deployment.yaml
```
#### Check for wordpress url
```
minikube -n wordpress service wordpress --url
```

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
Annotate the generated backup name.

#### Run the backup actionset for the wordpress container
```
kanctl create actionset --action backup \
                        --namespace kanister \
                        --blueprint wordpress-blueprint \
                        --deployment wordpress/wordpress \
                        --profile default-profile
```
Annotate the generate backup name.

#### Destroy WordPress deployment

With Kanister we can't recover a lost namespace, but we can rebuild an environment with a backup from a destroyed one.
We will recover two different scenarios:

1. We will break the database and also wordpress and recover from backup.

2. We will delete the wordpress namespace, recreate it and recover from backup.

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
