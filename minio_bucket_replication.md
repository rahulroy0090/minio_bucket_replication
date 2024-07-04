

### Create Podman Network

```
podman network create minio-network

```

### Generate SSL Certificates

```
# For MinIO DC
podman run -v /home/redhat/minio-dc/certs:/certs:Z \
    -e CA_SUBJECT="root CA" \
    -e CA_EXPIRE="1825" \
    -e SSL_EXPIRE="365" \
    -e SSL_SUBJECT="minio.dc.in" \
    -e SSL_DNS="minio.dc.in" \
    -e SILENT="true" \
    superseb/omgwtfssl

# For MinIO DR
podman run -v /home/redhat/minio-dr/certs:/certs:Z \
    -e CA_SUBJECT="root CA" \
    -e CA_EXPIRE="1825" \
    -e SSL_EXPIRE="365" \
    -e SSL_SUBJECT="minio.dr.in" \
    -e SSL_DNS="minio.dr.in" \
    -e SILENT="true" \
    superseb/omgwtfssl

```


### Run MinIO Containers

```
# For MinIO DC
podman run -d --name minio-dc --network minio-network --ip 10.89.0.20 \
  -p 9001:9000 \
  -v /home/redhat/minio-dc/data:/data:Z \
  -v /home/redhat/minio-dc/certs/cert.pem:/root/.minio/certs/public.crt:Z \
  -v /home/redhat/minio-dc/certs/key.pem:/root/.minio/certs/private.key:Z \
  -e "MINIO_ROOT_USER=minio" \
  -e "MINIO_ROOT_PASSWORD=minio123" \
  minio/minio server /data

# For MinIO DR
podman run -d --name minio-dr --network minio-network --ip 10.89.0.21 \
  -p 9002:9000 \
  -v /home/redhat/minio-dr/data:/data:Z \
  -v /home/redhat/minio-dr/certs/cert.pem:/root/.minio/certs/public.crt:Z \
  -v /home/redhat/minio-dr/certs/key.pem:/root/.minio/certs/private.key:Z \
  -e "MINIO_ROOT_USER=minio" \
  -e "MINIO_ROOT_PASSWORD=minio123" \
  minio/minio server /data

```


### Verify Containers

```
podman ps

CONTAINER ID  IMAGE                         COMMAND       CREATED       STATUS         PORTS                   NAMES
45dd451a3e5e  docker.io/minio/minio:latest  server /data  22 hours ago  Up 20 minutes  0.0.0.0:9001->9000/tcp  minio-dc
f541961d2158  docker.io/minio/minio:latest  server /data  22 hours ago  Up 20 minutes  0.0.0.0:9002->9000/tcp  minio-dr


```

### Add /etc/hosts for Name Resolution


```
10.89.0.20 minio.dc.in
10.89.0.21 minio.dr.in
```

### View Generated CA Certificates


```
cat /home/redhat/minio-dc/certs/cert.pem
cat /home/redhat/minio-dr/certs/cert.pem

```


### Import CA Certificates

```
# Import minio-dr’s CA-cert on minio-dc container
podman exec -u root -it minio-dc bash
echo '-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
' >> /etc/ssl/certs/ca-certificates.crt

# Import minio-dc’s CA-cert on minio-dr container
podman exec -u root -it minio-dr bash
echo '-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
' >> /etc/ssl/certs/ca-certificates.crt

```

### Restart MinIO Containers


```
podman restart minio-dc minio-dr

```


### Download and Install MinIO Client


```
# Download the MinIO client binary
curl https://dl.min.io/client/mc/release/linux-amd64/mc --create-dirs -o $HOME/minio-binaries/mc

# Make the binary executable
chmod +x $HOME/minio-binaries/mc

# Add the binary to your PATH
export PATH=$PATH:$HOME/minio-binaries/

# Verify the installation
mc --version

```

### Set Aliases for MinIO DC and DR


```
mc alias set minio-dc https://minio.dc.in:9000 minio minio123
mc alias set minio-dr https://minio.dr.in:9000 minio minio123

```

### Import CA Certificates on Local System

```
openssl s_client -connect minio.dc.in:9000 -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM > minio-dc-in.crt
sudo cp minio-dc-in.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

openssl s_client -connect minio.dr.in:9000 -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM > minio-dr-in.crt
sudo cp minio-dr-in.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

```

### Create Buckets and Enable Versioning

```
mc mb minio-dr/rahul
mc mb minio-dc/fino

mc version enable minio-dc/fino
mc version enable minio-dr/rahul

```

### Add Replication Configuration


```
mc replicate add minio-dc/fino --remote-bucket https://minio:minio123@minio.dr.in:9000/rahul --replicate existing-objects,delete,delete-marker,replica-metadata-sync
mc replicate status minio-dc/fino

mc replicate add minio-dr/rahul --remote-bucket https://minio:minio123@minio.dc.in:9000/fino --replicate existing-objects,delete,delete-marker,replica-metadata-sync --priority 1
mc replicate status minio-dr/rahul

```

### Copy a File to Test Replication


```
mc cp minio-dc-in.crt minio-dc/fino
mc ls minio-dc/fino

```

### Check the Replication status-


```
## mc replicate status minio-dr/rahul
  Replication status since 18 minutes                                                         
                                                                                              
  Summary:                                                                                    
  Replicated:                   0 objects (0 B)                                               
  Queued:                       ● 0 objects, 0 B (avg: 0 objects, 0 B ; max: 0 objects, 0 B)  
  Workers:                      0 (avg: 0; max: 0)                                            
  Received:                     0 objects (0 B)                                               
  Transfer Rate:                0 B/s (avg: 0 B/s; max: 0 B/s)                                
  Errors:                       0 in last 1 minute; 0 in last 1hr; 0 since uptime  
  
  
  
  ## mc replicate status minio-dc/fino
  Replication status since 49 minutes                                                         
  minio.dr.in:9000                                                                            
  Replicated:                   1 objects (1.1 KiB)                                           
  Queued:                       ● 0 objects, 0 B (avg: 0 objects, 0 B ; max: 0 objects, 0 B)  
  Workers:                      0 (avg: 0; max: 0)                                            
  Transfer Rate:                0 B/s (avg: 0 B/s; max: 0 B/s                                 
  Latency:                      2ms (avg: 2ms; max: 16ms)                                     
  Link:                         ● online (total downtime: 0 milliseconds)                     
  Errors:                       0 in last 1 minute; 0 in last 1hr; 0 since uptime 
```
