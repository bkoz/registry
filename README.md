# Private Docker registry setup on RHEL7.5
## 7/18/2018

### These are crude unedited notes from a recent installation.

### Create the host-mounted directories used to persist the registry.

#### This should be combined into a single directory.

```
mkdir /registry-certs
chmod -R 777 /registry-certs
chcon -R system_u:object_r:svirt_sandbox_file_t:s0 /registry-certs
mkdir -p /registry
chmod -R 777 /registry
chcon -R system_u:object_r:svirt_sandbox_file_t:s0 /registry
mkdir -p /registry-auth
chmod -R 777 /registry-auth
chcon -R system_u:object_r:svirt_sandbox_file_t:s0 /registry-auth
```

### Create the certificates
#### Need to replace this step with LetsEncrypt certs.

```
cd /registry-certs

openssl x509 -text -in domain.key
openssl req  -newkey rsa:4096 -nodes -sha256 -keyout /certs/domain.key  -x509 -days 365 -out /certs/domain.crt
openssl req  -newkey rsa:4096 -nodes -sha256 -keyout /domain.key  -x509 -days 365 -out /domain.crt
cp domain.crt /etc/pki/ca-trust/source/anchors/dist.example.com.crt

openssl genrsa -out domain-CA.key 2048
openssl req -x509 -new -nodes -key devdockerCA.key -days 10000 -out devdockerCA.crt
openssl req -x509 -new -nodes -key domain-CA.key -days 10000 -out domain-CA.crt
openssl genrsa -out domain.key 2048
openssl req -new -key domain.key -out dist.example.com.csr
openssl req -new -key domain.key -out dist.example.com.csr
openssl x509 -req -in dist.example.com.csr -CA domain-CA.crt -CAkey domain-CA.key -CAcreateserial -out domain.crt -days 10000
```

#### Dump the cert to text.

```
openssl x509 -text -in  domain.crt
```

#### Create a user/password and run the registry.

```
htpasswd -Bbn /registry-auth/htpasswd demo demo

docker run --restart=always -d -p 5000:5000 --name registry --restart=always -v /registry:/var/lib/registry -v /registry-certs:/certs -v /registry-auth:/auth -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -e REGISTRY_HTTP_ADDR=0.0.0.0:443   -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt   -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key   -p 8443:443 registry:2
```

#### Client configuration

```
cp /registry-certs/domain.crt /etc/pki/ca-trust/source/anchors/

mv /etc/pki/ca-trust/source/anchors/domain.crt /etc/pki/ca-trust/source/anchors/dist.example.com.
crt

update-ca-trust
  
systemctl restart docker

docker registry logs

time="2018-07-17T21:54:27Z" level=info msg="listening on [::]:443, tls" go.version=go1.7.6 instance.id=7f4d70e0-2618-4117-ae23-8e9e356c454d version=v2.6.2
time="2018-07-17T21:54:38Z" level=warning msg="error authorizing context: basic authentication challenge for realm \"Registry Realm\": invalid authorization credential" go.version=go1.7.6 http.request.host="dist.example.com:8443" http.request.id=6d02cb26-0147-4249-8172-1c53f3a80324 http.request.method=GET http.request.remoteaddr="10.0.0.3:37432" http.request.uri="/v2/" http.request.useragent="docker/1.13.1 go/go1.9.2 kernel/3.10.0-862.6.3.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.13.1 \\(linux\\))" instance.id=7f4d70e0-2618-4117-ae23-8e9e356c454d version=v2.6.2
10.0.0.3 - - [17/Jul/2018:21:54:38 +0000] "GET /v2/ HTTP/1.1" 401 87 "" "docker/1.13.1 go/go1.9.2 kernel/3.10.0-862.6.3.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.13.1 \\(linux\\))"
time="2018-07-17T21:54:38Z" level=info msg="response completed" go.version=go1.7.6 http.request.host="dist.example.com:8443" http.request.id=8ee626e7-bb14-44a7-9f3d-376ba1f2f6e9 http.request.method=GET http.request.remoteaddr="10.0.0.3:37434" http.request.uri="/v2/" http.request.useragent="docker/1.13.1 go/go1.9.2 kernel/3.10.0-862.6.3.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.13.1 \\(linux\\))" http.response.contenttype="application/json; charset=utf-8" http.response.duration=5.477611ms http.response.status=200 http.response.written=2 instance.id=7f4d70e0-2618-4117-ae23-8e9e356c454d version=v2.6.2
10.0.0.3 - - [17/Jul/2018:21:54:38 +0000] "GET /v2/ HTTP/1.1" 200 2 "" "docker/1.13.1 go/go1.9.2 kernel/3.10.0-862.6.3.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.13.1 \\(linux\\))"
```

#### Push test

```
docker login dist.example.com:8443 -u demo -p demo
Login Succeeded

docker tag docker.io/alpine/latest dist.example.com:8443/alpine/latest

docker push dist.example.com:8443/alpine/latest

The push refers to a repository [dist.example.com:8443/alpine/latest]
73046094a9b8: Pushed
latest: digest: sha256:0873c923e00e0fd2ba78041bfb64a105e1ecb7678916d1f7776311e45bf5634b size: 528

```

#### insecure test

```
docker run -p 5000:5000 --name registry -v /registry:/var/lib/registry -v /registry-certs:/certs -v /registry-auth:/auth  registry:2

docker push dist.example.com:5000/alpine:latest
The push refers to a repository [dist.example.com:5000/alpine]
73046094a9b8: Pushed
latest: digest: sha256:0873c923e00e0fd2ba78041bfb64a105e1ecb7678916d1f7776311e45bf5634b size: 528
[root@dist ~]# curl dist.example.com:5000
[root@dist ~]# curl dist.example.com:5000/v2
<a href="/v2/">Moved Permanently</a>.

# curl dist.example.com:5000/v2/
```