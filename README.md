# emby-app-carck
* 签发证书
  ```
  # 创建目录
  mkdir -p embycert && cd embycert
  # 生成 CA 密钥
  openssl genrsa -out ca.key 2048 
  # 生成 CA 证书
  openssl req -x509 -new -nodes -key ca.key -subj "/C=CN/ST=Beijing/L=Beijing/O=CHINA/OU=CHINA/CN=CHINA/emailAddress=mail@emby.cn" -days 36500 -out ca.crt
  # 将 CA 转换成 p12 格式，并指定密码 （CHINA）
  openssl pkcs12 -export -clcerts -in ./ca.crt -inkey ca.key -out ca.p12 -password pass:CHINA
  # 将 p12 格式的证书 Base64 编码
  base64 ca.p12
  # Base64 一行不能超过 76 字符，超过则添加回车换行符。如果因为换行的原因，不能安装证书，可以使用 -w 参数
  base64 -w 0 ca.p12
  # 将 CA 转换成 pem 格式
  openssl x509 -outform pem -in ca.crt -out ca.pem
  ```
  ```
  # 生成服务端私钥 server.key
  openssl genrsa -out server.key 2048
  # 生成服务端证书请求 server.csr
  openssl req -new -sha256 -key server.key -out server.csr -subj "/C=CN/L=Beijing/O=CHINA/OU=CHINA/CN=mb3admin.com/CN=*.mb3admin.com"
  # 生成服务端证书 server.crt
  openssl x509 -req -extfile <(printf "subjectAltName=DNS:mb3admin.com,DNS:*.mb3admin.com") -days 3650 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt
  ```
* 安装 Nginx
  ```
  # 安装 Nginx
  apt install nginx
  # 创建目录
  mkdir -p /etc/nginx/cert/emby-carck
  # 复制证书
  cp server.{crt,key} /etc/nginx/cert/emby-carck
  # 编辑配置文件
  vim /etc/nginx/sites-enabled/zz-emby.conf
  vim /etc/nginx/sites-enabled/zz-emby-carck.conf
  # 测试配置文件
  nginx -t
  # 重载配置文件
  nginx -s reload
  # 查看端口
  netstat -antulp | grep 443
  # 放行端口
  apt install firewalld && systemctl start firewalld
  firewall-cmd --add-port=443/tcp --zone=public --permanent && firewall-cmd --reload
  ```
* 创建配置文件
```
# /etc/nginx/sites-enabled/emby-carck.conf

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name mb3admin.com;

    ssl_certificate /etc/nginx/cert/emby-carck/server.crt;
    ssl_certificate_key /etc/nginx/cert/emby-carck/server.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;

    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Headers *;
    add_header Access-Control-Allow-Method *;
    add_header Access-Control-Allow-Credentials true;

    location /admin/service/registration/validateDevice {
        default_type application/json;
        return 200 '{"cacheExpirationDays": 365,"message": "Device Valid","resultCode": "GOOD"}';
    }

    location /admin/service/registration/validate {
        default_type application/json;
        return 200 '{"featId":"","registered":true,"expDate":"2099-01-01","key":""}';
    }

    location /admin/service/registration/getStatus {
        default_type application/json;
        return 200 '{"deviceStatus":"0","planType":"Lifetime","subscriptions":{}}';
    }
}
```
