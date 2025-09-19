# 使用 OpenSSL 生成自签名 SSL 证书并配置 Nginx HTTPS

## 步骤 1：使用 OpenSSL 生成自签名证书

OpenSSL 是一个开源工具，广泛用于生成 SSL/TLS 证书。以下是生成自签名证书的命令，证书有效期设为 99 年（即 365 * 99 = 36035 天）。

1. **安装 OpenSSL**（如果尚未安装）
   在大多数 Linux 发行版上，OpenSSL 通常已预装。如果没有，可以通过以下命令安装：

   ```bash
   # Ubuntu/Debian
   sudo apt update
   sudo apt install openssl
   
   # CentOS/RHEL
   sudo yum install openssl
   ```

2. **生成私钥和自签名证书**
   运行以下命令生成私钥和证书，证书有效期设为 99 年：

   ```bash
   openssl req -x509 -nodes -days 36035 -newkey rsa:2048 -keyout /etc/ssl/private/selfsigned.key -out /etc/ssl/certs/selfsigned.crt
   ```

   **说明**：

   - `-x509`：生成自签名证书。
   - `-nodes`：不加密私钥（避免每次重启 Nginx 时需要输入密码）。
   - `-days 36035`：证书有效期为 99 年。
   - `-newkey rsa:2048`：生成 2048 位 RSA 私钥。
   - `-keyout /etc/ssl/private/selfsigned.key`：私钥保存路径。
   - `-out /etc/ssl/certs/selfsigned.crt`：证书保存路径。

   **运行时交互**：
   执行上述命令时，OpenSSL 会提示输入一些信息，例如国家、组织等。示例输入如下：

   ```
   Country Name (2 letter code) [AU]: CN
   State or Province Name (full name) [Some-State]: Beijing
   Locality Name (eg, city) []: Beijing
   Organization Name (eg, company) [Internet Widgits Pty Ltd]: YourCompany
   Organizational Unit Name (eg, section) []: IT
   Common Name (e.g. server FQDN or YOUR name) []: example.com
   Email Address []: admin@example.com
   ```

   - **Common Name (CN)**：应设置为你的域名（例如 `example.com`）或服务器 IP 地址。如果是内网使用，可以用 IP 或任意名称。

3. **验证证书**
   生成后，可以检查证书信息：

   ```bash
   openssl x509 -in /etc/ssl/certs/selfsigned.crt -text -noout
   ```

   确认有效期是否为 99 年（检查 `Not After` 字段）。

## 步骤 2：配置 Nginx 使用自签名证书

1. **确保 Nginx 已安装**
   如果尚未安装 Nginx，可以通过以下命令安装：

   ```bash
   # Ubuntu/Debian
   sudo apt install nginx
   
   # CentOS/RHEL
   sudo yum install nginx
   ```

2. **修改 Nginx 配置文件**
   编辑 Nginx 的配置文件（通常位于 `/etc/nginx/nginx.conf` 或 `/etc/nginx/conf.d/` 目录下）。以下是一个示例配置，启用 HTTPS 并使用自签名证书：

   ```nginx
   server {
       listen 443 ssl;
       server_name example.com;
   
       ssl_certificate /etc/ssl/certs/selfsigned.crt;
       ssl_certificate_key /etc/ssl/private/selfsigned.key;
   
       # 提高安全性（可选）
       ssl_protocols TLSv1.2 TLSv1.3;
       ssl_prefer_server_ciphers on;
       ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
   
       # 网站根目录（根据实际需求调整）
       root /var/www/html;
       index index.html;
   
       location / {
           try_files $uri $uri/ /index.html;
       }
   }
   
   # 可选：将 HTTP 请求重定向到 HTTPS
   server {
       listen 80;
       server_name example.com;
       return 301 https://$host$request_uri;
   }
   ```

   **说明**：

   - `listen 443 ssl`：监听 443 端口并启用 SSL。
   - `ssl_certificate` 和 `ssl_certificate_key`：指定证书和私钥的路径。
   - `ssl_protocols` 和 `ssl_ciphers`：增强 TLS 安全性（可选）。
   - `server_name`：替换为你的域名或 IP 地址。
   - HTTP 到 HTTPS 的重定向配置（第二个 `server` 块）是可选的，但推荐使用。

3. **保存配置文件并测试**
   保存配置文件后，检查语法是否正确：

   ```bash
   sudo nginx -t
   ```

   如果没有错误，重启 Nginx 应用配置：

   ```bash
   sudo systemctl restart nginx
   ```

## 步骤 3：测试 HTTPS 访问

1. **访问网站**
   在浏览器中访问 `https://example.com`（替换为你的域名或 IP）。由于是自签名证书，浏览器会显示“证书不受信任”的警告。
2. **处理证书警告**  
   - **内网使用**：可以手动信任证书（在浏览器中选择“继续”或“接受风险”）。
   - **分发证书**：将 `selfsigned.crt` 分发给客户端，并让用户手动导入到浏览器的信任证书列表中。
   - **长期解决方案**：考虑使用免费的 Let's Encrypt 证书（如果有公网域名），它也是开源的，且受浏览器信任。

## 注意事项

- **自签名证书的局限性**：自签名证书不被浏览器或客户端默认信任，适合内网或测试环境。如果需要公网使用，建议使用 Let's Encrypt 的免费证书。

- **权限管理**：确保私钥文件（`selfsigned.key`）权限严格，只允许 Nginx 和 root 用户访问：

  ```bash
  sudo chmod 600 /etc/ssl/private/selfsigned.key
  sudo chown root:root /etc/ssl/private/selfsigned.key
  ```

- **证书续期**：虽然证书设为 99 年有效期，但建议定期（例如每 5-10 年）检查和更新证书，以符合最新的安全标准。