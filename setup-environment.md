# kanister-demo on Centos 7

## Setup Environment

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

### Install Kanister Operator
```
helm repo add kanister https://charts.kanister.io/
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
