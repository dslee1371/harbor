## Server info
`192.168.1.10`

## Centos install
`D-2@`

## configure ip
`192.168.10.10 	bastion`

## Install Package
```
yum update
yum install wget
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker
#sudo yum install docker-ce docker-ce-cli containerd.io

```

## Install Docker composer
```
(old)sudo curl -L https://github.com/docker/compose/releases/download/1.24.1/docker-compose-uname -s-uname -m -o /usr/local/bin/docker-compose
(new)sudo curl -L "https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

## Download harbor
```
(old)wget https://storage.googleapis.com/harbor-releases/release-1.8.0/harbor-offline-installer-v1.8.1.tgz
(latest) wget https://github.com/goharbor/harbor/releases/download/v1.10.1/harbor-offline-installer-v1.10.1.tgz
```

## Unzip harbor zip file
```
tar xvzf harbor-offline-installer-v1.10.1.tgz
cd harbor # check file list > ??
mkdir -p /opt/certs
cd /opt/certs
```

## Create CA
```
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -key ca.key \
 -out ca.crt

#openssl req -x509 -new -nodes -sha512 -days 3650 \
# -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=registry.dsleecom.lab" \
# -key ca.key \
# -out ca.crt
```
## Create server certificate
```
openssl genrsa -out registry.dsleecom.lab.key 4096
openssl req -newkey rsa:4096 -nodes -sha256 -keyout domain.key -x509 -days 365 -out domain.crt
openssl req -sha512 -new \
    -key registry.dsleecom.lab.key \
    -out registry.dsleecom.lab.csr

#openssl req -sha512 -new \
#   -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=registry.dsleecom.lab" \
#    -key registry.dsleecom.lab.key \
#    -out registry.dsleecom.lab.csr
```
> doamin : registry.dsleecom.lab 


## Generate an x509 v3 extension file
```
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=registry.dsleecom.lab
DNS.2=dsleecom.lab
DNS.3=registry
EOF
```

# Use the v3.ext file to generate a certificate for your habor
```
openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in registry.dsleecom.lab.csr \
    -out registry.dsleecom.lab.crt
```

## Provide the Certificates to Harbor and Docker
1. Copy the server certificate and key into the certificates folder on your Horbor host.
```
cp registry.dsleecom.lab.crt /data/cert
cp registry.dsleecom.lab.key /data/cert
```
2. Convert registry.dsleecom.lab.crt to registry.dsleecom.lab.cert, for user by Docker.
The Docker daemon interprets .crt files as CA certificates and .cert files as client certificates.
```
openssl x509 -inform PEM -in registry.dsleecom.lab.crt -out registry.dsleecom.lab.cert
openssl x509 -inform PEM -in ca.crt -out ca.cert
```
3. Copy the server certificate, key and CA files into the Docker certificates folder on the Harbor host. You must create the appropriate folders first.
```
cp registry.dsleecom.lab.cert /etc/docker/certs.d/registry.dsleecom.lab/
cp registry.dsleecom.lab.key /etc/docker/certs.d/registry.dsleecom.lab/
cp ca.crt /etc/docker/certs.d/registry.dsleecom.lab/
cp registry.dsleecom.lab.crt /etc/docker/certs.d/registry.dsleecom.lab/
cp ca.key /etc/docker/certs.d/registry.dsleecom.lab/
```

> If you mapped the default nginx port 443 to a different port, create the folder
'/etc/docker/cert.d/registry.dsleecom.lab:port' or /'etc/docker/cert.d/harbor_IP:port'.



## Restart Docker Engine.

`systemctl restart docker`

## Deploy or Reconfigure Harhor
```
IF you have not yet deployed Harbor, see Configure the Harbor YML for information about hot to configure Harbor to use the certificates by specifying the hostname and https attributes in harbor.yml
https://github.com/goharbor/harbor/blob/master/docs/1.10/install-config/configure-yml-file.md

If you already deployed Harbor with HTTP and want to reconfigure it to use HTTPS, perfoem the following steps.

1. Run the prepare script to enable HTTPS.

Harbor uses an nginx instance as a reverse proxy for all services. You use the prepare script to configure nginx to use HTTPS. The prepare is in the Harbot installer bundle, at the same level as the install.sh script.
```

## Connfigure firewall
```
firewall-cmd --permanent --zone=public --add-service=http
firewall-cmd --permanent --zone=public --add-service=https
firewall-cmd --reload
```

## edit `prepare` file Run prepare script
```
docker run --rm -v $input_dir:/input:z \
                    -v $data_path:/data:z \
                    -v $harbor_prepare_path:/compose_location:z \
                    -v $config_dir:/config:z \
                    -v $secret_dir:/secret:z \
                    -v /data/cert:/hostfs:z \
```

## run services
```
# If Harbor in running, stop and remove the existing instance.\
# Your image data remains in the file system, so no data is lost.
docker-compose down -v

# Restart Harbor:
docker-compose up -d

```

## refer url
```
(install example)https://thenewstack.io/tutorial-install-the-docker-harbor-registry-server-on-ubuntu-18-04/
(download link)https://github.com/goharbor/harbor/releases
(git) https://github.com/goharbor/harbor
(install guide) https://github.com/goharbor/harbor/tree/master/docs/1.10
(prerequiste)https://github.com/goharbor/harbor/blob/master/docs/1.10/install-config/installation-prereqs.md
```

## install nginx
``` 
#nginx installation create and edit file as below example code scripts
vi /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1

# install nginx package  
yum install -y nginx

# run and enable service
systemctl start nginx
systemctl enable nginx 
```

## Error Log
```
FileNotFoundError: [Errno 2] No such file or directory: '/hostfs/opt/cert/registry.dsleecom.lab.key'
```

## package install log as `yum install docker-ce`
```
(1/3): containerd.io-1.2.13-3.1.el7.x86_64.rpm                                                         |  23 MB  00:00:06
(2/3): docker-ce-19.03.6-3.el7.x86_64.rpm                                                              |  24 MB  00:00:07
(3/3): docker-ce-cli-19.03.6-3.el7.x86_64.rpm    
```
## remove old versions as `yum install docker-ce`
```
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

## check docker version 
```
#yum install docker-ce
[root@registry docker]# docker --version
Docker version 19.03.6, build 369ce74a3c

#yum install docker
[root@registry sysconfig]# docker --version
Docker version 1.13.1, build 4ef4b30/1.13.1
```

## docker config example
```
# /etc/sysconfig/docker

# Modify these options if you want to change the way the docker daemon runs
OPTIONS='--selinux-enabled --log-driver=journald --signature-verification=false --insecure-registry registry.futuregen-ocp.lab:5000 --insecure-registry futuregen.icp:8500 --insecure-registry docker-registry-default.apps.futuregen-ocp.lab'
if [ -z "${DOCKER_CERT_PATH}" ]; then
    DOCKER_CERT_PATH=/etc/docker
fi

# Do not add registries in this file anymore. Use /etc/containers/registries.conf
# instead. For more information reference the registries.conf(5) man page.

# Location used for temporary files, such as those created by
# docker load and build operations. Default is /var/lib/docker/tmp
# Can be overriden by setting the following environment variable.
# DOCKER_TMPDIR=/var/tmp

# Controls the /etc/cron.daily/docker-logrotate cron job status.
# To disable, uncomment the line below.
# LOGROTATE=false

# docker-latest daemon can be used by starting the docker-latest unitfile.
# To use docker-latest client, uncomment below lines
#DOCKERBINARY=/usr/bin/docker-latest
#DOCKERDBINARY=/usr/bin/dockerd-latest
#DOCKER_CONTAINERD_BINARY=/usr/bin/docker-containerd-latest
#DOCKER_CONTAINERD_SHIM_BINARY=/usr/bin/docker-containerd-shim-latest
ADD_REGISTRY='--add-registry registry.futuregen-ocp.lab:5000'
INSECURE_REGISTRY='--insecure-registry registry.futuregen-ocp.lab:5000 --insecure-registry futuregen.icp:8500'

```
