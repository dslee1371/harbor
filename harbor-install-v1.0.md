
## Install package
```
# Install yum package
sudo yum -y install wget docker

# Install docker composer
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

## Download harbor and upzip 
```
wget https://github.com/goharbor/harbor/releases/download/v1.10.1/harbor-offline-installer-v1.10.1.tgz
tar xvzf harbor-offline-installer-v1.10.1.tgz
```

## Create CA

```
mkdir -p /opt/certs
cd /opt/certs
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -key ca.key \
 -out ca.crt
```

## Create Server certificate
```
openssl genrsa -out registry.futurgen.lab.key 4096
openssl req -sha512 -new \
    -key registry.futurgen.lab.key \
    -out registry.futurgen.lab.csr
```

## Generate an x509 v3 extension file
```
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=registry.futuregen.lab
DNS.2=futuregen.lab
DNS.3=registy
EOF
```

## Use the v3.ext file to generate a certificate fore your harbor
```
openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in registry.futurgen.lab.csr \
    -out registry.futurgen.lab.crt
```

## Provide the certificates to harbor
```
cp registry.futurgen.lab.crt /data/cert
cp registry.futurgen.lab.key /data/cert
```

## Edit `prepare` file (refer below)
```
docker run --rm -v $input_dir:/input:z \
                    -v $data_path:/data:z \
                    -v $harbor_prepare_path:/compose_location:z \
                    -v $config_dir:/config:z \
                    -v $secret_dir:/secret:z \
                    -v `/data/cert`:/hostfs:z \
```

## Run Service
If Harbor in running, stop and remove the existing instance.
Your image data remains in the file system, so no data is lost.
```
./prepare
docker-compose down -v
docker-compose up -d
```

## Test container images
```
docker pull nginx
docker tag docker.io/nginx:latest registry.futuregen.lab/library/nginx:20200304
docker login registry.futuregen.lab
docker push registry.futuregen.lab/library/nginx:20200304
```

## Reference Documents

- (install example)https://thenewstack.io/tutorial-install-the-docker-harbor-registry-server-on-ubuntu-18-04/
- (download link)https://github.com/goharbor/harbor/releases
- (git) https://github.com/goharbor/harbor
- (install guide) https://github.com/goharbor/harbor/tree/master/docs/1.10
- (prerequiste)https://github.com/goharbor/harbor/blob/master/docs/1.10/install-config/installation-prereqs.md

## Error logs
```
[Issue 1] FileNotFoundError: [Errno 2] No such file or directory: '/hostfs/opt/cert/registry.dsleecom.lab.key'
[Solution] check create and copy to `/data/cert` folder as crt certificate
```
