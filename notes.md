###创建带TLS认证的docker私有仓库
1. 环境  
FT1500A台式机，kylin.5.server，内核4.4.13，IP地址192.168.0.191，docker1.12.3

2. 不带tls认证的私有仓库  
一开始用的是docker-registry2.3.0软件，搭建不带tls的私有仓库时，需要在`/etc/default/docker`中
添加`DOCKER_OPTS="--insecure-registry 192.168.0.191:5000"`，重启docker.service及docker-registry.service即可

3. 带tls认证的私有仓库  
* 创建证书
修改`/etc/ssl/openssl.cnf`

```  
[CA_default]
dir = /etc/ssl/demoCA

[v3_ca]
subjectAltName = IP:192.168.0.191
```  

在`/etc/ssl`路径下

```
cd /etc/ssl
mkdir demoCA demoCA/certs demoCA/crl demoCA/newcerts demoCA/private
touch /etc/ssl/demoCA/index.txt
echo 01 > /etc/ssl/demoCA/serial
cd /etc/ssl/demoCA
openssl req -x509 -days 365 -nodes -newkey rsa:2048 -keyout cakey.pem -out cacert.pem
mv cacert.pem certs/
mv cakey.pem private/
```

其中cakey.pem为私钥，cacert.pem为公钥和证书，在生成证书过程中会要求用户输入信息，其中的Common Name字段要
填写为192.168.0.191或者自定义的域名（如registry.com），如果要用域名，则需要在/etc/hosts中添加一行`192.168.0.191 registry.com`，然后执行`cat /etc/ssl/certs/cacert.pem >> /etc/ssl/cert/ca-certifiates.crt`之后重启docker
修改`/etc/docker/registry/config.yml`之后重启docker-registry

```
http:
  addr: :5000
  tls: 
    certificate: /etc/ssl/demoCA/certs/cacert.pem
    key: /etc/ssl/demoCA/private/cakey.pem
```

4. 问题
按照前面的步骤执行后，
`docker tag aarch/ubuntu 192.168.0.191:5000/ubuntu`
`docker push 192.168.0.191:5000/ubuntu`
一直卡在retrying in x seconds，push不成功，在网上各种查找，没有找到解决方法  

5. 换用registry镜像
实在没办法，从网上下载了registry镜像，先将`/etc/ssl/demoCA/private/cakey.pem`放到`/demoCA/certs`中

```
docker run -d -p 5000:5000 --name registry -v /etc/ssl/demoCA/certs:/certs \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/cacerts.pem \
-e REGISTRY_HTTP_TLS_KEY=/certs/cakey.pem \
registry
```

待容器registry成功运行后，再执行步骤4的docker tag和docker push，成功！
在另外一台机器上docker pull，需要先将`/etc/ssl/demoCA/certs/cacert.pem`证书添加到其本地，可通过scp
命令远程拷贝，试验中另一台机器IP为192.168.0.192
`scp /etc/ssl/demoCA/certs/cacert.pem 192.168.0.192:~/`
然后在192上`cat cacert.pem >> /etc/ssl/certs/ca-certificates.crt`
之后重启docker，即可实现docker pull



