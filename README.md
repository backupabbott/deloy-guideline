## 1. Chose EC2
Bước 1: 
Để chọn cấu hình EC2 Sever bạn nên chọn sever: ubuntu 20.04 trở lên, cấu hình: (Strapi cần tối thiểu 2GB ram để buid)
1 core, 2 GB ram, ổ cứng EBS - 
Thường sẽ chọn 2 EBS để gắn vào hệ thống: 
- 1 EBS làm lưu trữ dung lượng thấp (dùng để lưu session, logs, cài phần mềm), 
- 1 EBS dùng làm swap (X2 dung lượng ram)
Nếu sever sử dụng Redis ở bên ngoài làm nơi lưu session thì có thể chọn dung lượng thấp.
Tùy chọn Security group: cho phép truy cập Inbound and Outbound vào EC2, mở những port nào để xài ứng dụng:

Khi khởi tạo thành công user mặc định là: `ubuntu` và để đăng nhập bắt buộc phải sử dụng SSH key. Tạo private key vào insert vào hệ thống.

### 1.1 Cài đặt Swap cho EC2
EC2 không cấp ổ cứng từ ban đầu mà cho phép gắn EBS riêng biệt, và nó ko có swap nên phải add phân vùng rồi thêm.

Cài đặt EBS làm swap theo link: https://aws.amazon.com/vi/premiumsupport/knowledge-center/ec2-memory-partition-hard-drive/
Lưu ý khi tạo partition thì nên nhớ tên là gì để vào hệ thống tạo swap tương ứng phân vùng đó. Một hướng dẫn để cài tool partition để xử lý đĩa:
https://www.cyberciti.biz/faq/debian-ubuntu-reload-partition-table/

## 2. Install ftp sever
Để cho phép user upload source lên sever thì cần cấp tài khoản FTP, để cài cần làm theo link:
https://www.digitalocean.com/community/tutorials/how-to-set-up-vsftpd-for-a-user-s-directory-on-ubuntu-18-04

Mở các port 20, 21, 990, 40000:50000 để cho phép từ bên ngoài internet truy cập vào:
Thực hiện mở trên firewall Ubuntu, EC2.
```bash
    sudo ufw allow 20/tcp
    sudo ufw allow 21/tcp
    sudo ufw allow 990/tcp
    sudo ufw allow 40000:50000/tcp --> port này là dải port mở để truyền dữ liệu, phải đăng kí trong config vsftpd
    sudo ufw status
 ```
Thêm user tương ứng vào để cho phép user được sử dụng ftp.
```bash
    echo "sammy" | sudo tee -a /etc/vsftpd.userlist
 ```
Có thể gắn bash để chặn user nào xài ftp thì ko được quyền xài ssh, nhưng trong hệ thống ko cần sử dụng.

Config chuẩn:

```bash
listen=NO
listen_ipv6=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
chroot_local_user=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
force_dot_files=YES
pasv_min_port=40000
pasv_max_port=50000

#user_sub_token=$USER
#local_root=/home/$USER/public_html
allow_writeable_chroot=YES

ssl_enable=YES
rsa_cert_file=/etc/ssl/private/vsftpd.pem
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1=YES
ssl_sslv2=NO
ssl_sslv3=NO
require_ssl_reuse=NO
ssl_ciphers=HIGH

port_enable=YES
pasv_enable=YES
pasv_address=3.1.142.167
pasv_addr_resolve=YES
check_shell=NO
passwd_chroot_enable=YES
 ```

## 3. Install nginx
Cài đặt nginx để làm proxy sever, và lưu ý khi cài nginx thì sẽ load cấu hình từ /etc/nginx/sites-enable và /etc/nginx/config.d đặt cấu hình config ở đây để hệ thống tự load đúng.

https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04

### Config 'similac'
```bash
server {
    # Listen HTTP
    listen 80;
    server_name similac.vn;

    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    # Listen HTTPS
    listen 443 ssl;
    server_name similac.vn;

    # SSL config
    ssl_certificate /etc/letsencrypt/live/similac.vn/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/similac.vn/privkey.pem; # managed by Certbot

    ## Gzip config
    gzip on;
    client_max_body_size          512M;

    # Static Root
    location / {
        proxy_pass https://similac;
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_pass_request_headers on;
    }

    # Strapi API
    location /api/ {
        rewrite ^/api/?(.*)$ /$1 break;
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_pass_request_headers on;
    }

    # Strapi Dashboard
    location /admin {
        proxy_pass http://backend/admin;
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_pass_request_headers on;
    }

}
 ```

Đừng quên upstream, khai báo những IP hay host nào được dùng làm sever:
```bash
# Strapi server
upstream similac {
    server 127.0.0.1:6400;
}

upstream backend {
    server 127.0.0.1:1339;
}
 ```

## 4. Install netcore
Cài đặt .netcore cho hệ thống theo hướng dẫn:
https://docs.microsoft.com/en-us/dotnet/core/install/linux-ubuntu

cài đặt gói sdk 3.1 và runtime tương ứng. Có thể ko cần mở port netcore.

## 5. Install nodejs && npm
Cài đặt theo hướng dẫn:
https://www.digitalocean.com/community/tutorials/how-to-install-node-js-on-ubuntu-20-04

Là hệ thống đã cài được Nodejs kèm, npm

## Cài gói pm2
Pm2 là một gói cài quản lí tiến trình ngầm bên dưới cho ứng dụng: Nodejs, netcore để thuận tiện quản lý.

https://pm2.keymetrics.io/docs/usage/quick-start/

Khi cấu hình xong thì đừng quên chạy: 
`pm2 startup` --> để chạy cấu hình sau khi khởi động
`pm2 save` --> lưu config pm2
## 6. Install cerbot
Để sử dụng SSL trên domain thì cần cài let's encrypt vào hệ thống, ở đây xài cerbot-nginx

https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-20-04

Cài theo hướng dẫn, đừng quên config để load file pem và cert vào cấu hình nginx

## 7. Config strapi
### Config S3

- Tạo user dành cho S3 ở phần cấu hình AIM user, dùng trong config.
https://strapi.io/documentation/developer-docs/latest/setup-deployment-guides/deployment/hosting-guides/amazon-aws.html#amazon-aws-install-requirements-and-creating-an-iam-non-root-user

- Tạo s3 bucket name và cấu hình ở:
https://strapi.io/documentation/developer-docs/latest/setup-deployment-guides/deployment/hosting-guides/amazon-aws.html#configure-s3-for-image-hosting

Khi chọn cho s3 setting check giống bucket.
- Cấu hình trong strapi
https://strapi.io/documentation/developer-docs/latest/setup-deployment-guides/deployment/hosting-guides/amazon-aws.html#_3-install-the-strapi-aws-s3-upload-provider

```bash
module.exports = ({ env }) => ({
    upload: {
      provider: 'aws-s3',
      providerOptions: {
        accessKeyId: env('AWS_ACCESS_KEY_ID'),
        secretAccessKey: env('AWS_ACCESS_SECRET'),
        region: env('AWS_REGION'),
        params: {
          Bucket: env('AWS_BUCKET_NAME'),
        },
        cdn: env('AWS_CLOUDFRONT'),
        localServer: {
          maxage: 300000
        }
      },
      breakpoints: {
        xlarge: 1920,
        large: 1000,
        small: 500
      }
    }
  });
```
Thêm cdn thì cần add thêm đoạn: `cdn: env('AWS_CLOUDFRONT')`
### Cấu hình env cho strapi
Trong thư mục `config` tạo folder `env` tạo 3 folder tương ứng với 3 môi trường: `development`, `stagging`, `production`. Chép các file ở config và tiến hành sửa cho tương ứng với từng môi trường.

### Cấu hình config strapi
 - Cấu hình `sever.js` cho `production` sử dụng port 1337 chạy chính, khi có 1 website mới thì nên tạo port mới, môi trường production cần thêm url.
```bash
module.exports = ({ env }) => ({
  host: env('HOST', '0.0.0.0'),
  port: env.int('PORT', 1337),
  // charset: 'utf8mb4',
  url: env('APP_HOST', 'https://cms.similac.byte.vn'),
  admin: {
    auth: {
      secret: env('ADMIN_JWT_SECRET', 'ea859d241e64156e80245f109ee3de33'),
    },
  }
});
```
### Sử dụng pm2 để tạo khởi chạy
Tạo file `strapi.conf.js`

```bash
module.exports = {
  apps: [
    {
      name: 'strapi',
      cwd: '/home/strapi/public_html',
      script: 'npm',
      args: 'start',
      env: {
        NODE_ENV: 'production',
        DATABASE_HOST: 'ci-similac.chmew1lwv7ml.ap-southeast-1.rds.amazonaws.com', // database Endpoint under 'Connectivity & Security' tab
        DATABASE_PORT: '3306',
        DATABASE_NAME: 'strapi', // DB name under 'Configuration' tab
        DATABASE_USERNAME: 'admin', // default username
        DATABASE_PASSWORD: 'ciadmin123',
        'DATABASE_SSL': 'true',
        AWS_ACCESS_KEY_ID: 'AKIAVL4SNFVVKZUU3IEX',
        AWS_ACCESS_SECRET: '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', // Find it in Amazon S3 Dashboard
        AWS_REGION: 'ap-southeast-1',
        AWS_BUCKET_NAME: 'ci-strapi-s3',
      },
    },
  ],
};
```

Sử dụng: `pm2 start strapi.config.js strapi -i max` là để tạo 1 config chạy strapi load cấu hình từ file js. Với môi trường production sử dụng những cấu hình ở bên dưới.


Xem thêm:
https://strapi.io/documentation/developer-docs/latest/setup-deployment-guides/deployment/hosting-guides/amazon-aws.html#_6-install-pm2-runtime

### Qui trình thông thường sẽ chạy như sau:
```bash
su strapi --> Đăng nhập user sẽ chạy strapi
cd ~ && cd public_html --> Về thư mục gốc vào thư mục public html
git pull --> Tải code về hoặc upload ftp lên
pm2 stop strapi -> Dừng ứng dụng pm2 chạy strapi
cd plugins/wysiwyg && npm install --> Cài package cho plugins, các lần sau không cần
cd ~ && cd public_html --> Về thư mục piblic_html
npm install --> Cài đặt package cho strapi
NODE_ENV=production npm run build --> Build với môi trường production
pm2 start strapi --> Khởi chạy strapi
```

Sau khi khởi chạy xong check xem đã chạy chưa, port strapi khai phải đúng trên cấu hình nginx, nếu ko sẽ lỗi 502.

### Để xem logs
Sử dụng `pm2 status` để xem tất cả trạng thái pm2
Sử dụng `pm2 logs strapi` xem logs strapi, hoặc có thể `cd ~ && cd .pm2/logs` để xem logs.
Sử dụng `pm2 monit` để xem trạng thái biểu đồ chạy

## 8. Config .netcore
## Thông  thường sẽ cấu hình:
```bash
su netcore
cd ~ && cd sources
git pull -> Kéo source về hay upload ftp, về folder sources
pm2 stop netcore --> Dùng pm2 chạy netcore
dotnet publish /home/netcore/sources/SmartCsmNetCore.Site/SmartCsmNetCore.Site.csproj -r linux-x64 -o /home/netcore/public_html/ --> Sử dụng lệnh này để build source .netcore về folder public html
cd ~ && cd public_html
pm2 start netcore --> Khởi chạy .netcore

```
Trước khi chạy phải cấu hình pm2 trước:
`pm2 start dotnet 'tên file dll' netcore -i max`

Các cấu hình tương tự như strapi.

### File config `appsettings.json`

```bash
{
  "Logging": {
    "LogFilePath": "logs/ci-mvc/ci-mvc.txt",
    "LogFilePathError": "logs/ci-mvc-errors/ci-mvc.txt"
  },
  "AllowedHosts": "*",
  "KeyVault": {
    "Vault": "kv-aiavndev-robo01",
    "ClientId": "b320ca6d-0f08-4c0c-9e59-1f9474cd2855",
    "ClientSecret": "8~ImB-X42w.x_fp0.V25r~dxMSjxNfhj15"
  },
  "Tokens": {
    "Key": "6J1MNCBkG3yc7zj04c",
    "Issuer": "localhost",
    "TokenExpired": 15
  },
  "Urls": [
    "https://localhost:6300"
  ],
  "Website": "http://localhost:6234",
  "CorsSettings": {
    "AllowOrigin": "*",
    "AllowMethod": "*",
    "AllowHeader": "*"
  },
  "AppSettings": {
    "Item1": "Item1 value",
    "SampleSettings": {
      "SampleString": "SampleString value"
    }
  },
  "WebAPI": "https://cms.similac.byte.vn/",
  "TokenAPI": {
    "Key": "123",
    "Issuer": "localhost",
    "TokenExpired": 15
  }
}
```
Trong source cần cấu hình để tiếp nhận proxy header, cấu hình bảo mật + cấu hình để bắt chạy https mặc định.

### Lưu ý:
https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/linux-nginx?view=aspnetcore-5.0
Để build được trên EC2 source của .netcore cần config để build được cho linux:

```bash
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>netcoreapp3.1</TargetFramework>
    <AssemblyName>SmartCsmNetCore.Site</AssemblyName>
    <RootNamespace>SmartCsmNetCore.Site</RootNamespace>
    <UserSecretsId>bb4bb8e8-7f9b-41cb-b4a9-2141b7337bea</UserSecretsId>
  </PropertyGroup>

  <PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Debug|AnyCPU'">
    <PlatformTarget>x64</PlatformTarget>
  </PropertyGroup>
  <PropertyGroup>
    <RuntimeIdentifiers>win10-x64;linux-x64</RuntimeIdentifiers>
  </PropertyGroup>

  <ItemGroup>
    <Content Remove="package.json" />
  </ItemGroup>
```

Trong file `SmartCsmNetCore.Site.csproj` thêm `<RuntimeIdentifiers>win10-x64;linux-x64</RuntimeIdentifiers>` để thêm cấu hình build cho linux.

## 9. Firewall
Trên AWS muốn cho phép ứng dụng từ bên ngoài truy cập vào cần mở firewall, để mở firewall vào ứng dụng tìm Security Group để thêm tương ứng giao thức: TCP/UDP, IP và port.

Ngoài ra trên VPS EC2 còn có 1 firewall nữa, cấu hình theo link

https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-20-04

Danh sách các port cần mở:
```bash
To                         Action      From
--                         ------      ----
Nginx HTTP                 ALLOW       Anywhere (:80)
Nginx Full                 ALLOW       Anywhere (:80 - :443)
OpenSSH                    ALLOW       Anywhere (:22)
40000:50000/tcp            ALLOW       Anywhere --> Port truyền dữ liệu FTP
990/tcp                    ALLOW       Anywhere
6379/tcp                   ALLOW       Anywhere
20:21/tcp                  ALLOW       Anywhere
1338/tcp                   ALLOW       Anywhere
1337/tcp                   ALLOW       Anywhere
Nginx HTTP (v6)            ALLOW       Anywhere (v6)
Nginx Full (v6)            ALLOW       Anywhere (v6)
OpenSSH (v6)               ALLOW       Anywhere (v6)
40000:50000/tcp (v6)       ALLOW       Anywhere (v6) --> Port truyền dữ liệu FTP
990/tcp (v6)               ALLOW       Anywhere (v6)
6379/tcp (v6)              ALLOW       Anywhere (v6)
20:21/tcp (v6)             ALLOW       Anywhere (v6)
1338/tcp (v6)              ALLOW       Anywhere (v6)
1337/tcp (v6)              ALLOW       Anywhere (v6)
```

## 10. Clone S3
Hiện tại có 2 S3 bucket, 1 cái là s3 bucket dùng test và 1 s3 bucket sử dụng chính. Và khi chạy thực tế giờ muốn lấy về test thì cần clone về, thì dùng theo hd bên dưới.

https://aws.amazon.com/premiumsupport/knowledge-center/move-objects-s3-bucket/
https://aws.amazon.com/vi/premiumsupport/knowledge-center/read-access-objects-s3-bucket/
