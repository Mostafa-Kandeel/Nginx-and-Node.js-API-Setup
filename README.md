# Nginx and Node.js API Setup

This project provides a step-by-step guide to set up an Nginx web server, a simple Node.js API, a reverse proxy, load balancing, caching, rate limiting, and SSL/TLS certificate configuration on a Linux system (Ubuntu/Debian).

## Table of Contents
- [Prerequisites](#prerequisites)
- [Stage 1: Installing and Configuring Nginx](#stage-1-installing-and-configuring-nginx)
- [Stage 2: Simple Node.js API](#stage-2-simple-nodejs-api)
- [Stage 3: Reverse Proxy Server](#stage-3-reverse-proxy-server)
- [Stage 4: Nginx Load Balancing](#stage-4-nginx-load-balancing)
- [Stage 5: Nginx Caching and Rate Limiting](#stage-5-nginx-caching-and-rate-limiting)
- [Stage 6: Setting Up SSL/TLS with Let's Encrypt](#stage-6-setting-up-ssltls-with-lets-encrypt)
- [Testing](#testing)
- [Contributing](#contributing)


## Prerequisites
- Ubuntu/Debian-based system
- `sudo` privileges
- Basic knowledge of Linux commands, Nginx, and Node.js
- Installed tools: `apt`, `nodejs`, `npm`
- A registered domain name (for SSL/TLS, e.g., `test.kandeel.local` must resolve to your server’s IP)
- Port 80 open for Let's Encrypt verification

## Stage 1: Installing and Configuring Nginx
Install and configure Nginx to serve a static website.

1. **Install Nginx**:
   ```bash
   sudo apt update
   sudo apt install nginx -y
   ```
2. **Check Nginx Configuration Files**:
   ```bash
   ls /etc/nginx
   ```
   Expected output: `conf.d/  modules-available/  modules-enabled/  sites-enabled/`

3. **Create a Website**:
   - Create a directory for the website:
     ```bash
     sudo mkdir -p /var/www/mysite
     sudo chown -R $USER:$USER /var/www/mysite
     cd /var/www/mysite
     touch index.html
     ```
   - Edit `index.html` (e.g., using `vim`) and add your HTML content.

4. **Configure Nginx for the Website**:
   - Create a configuration file:
     ```bash
     sudo touch /etc/nginx/sites-available/mysite
     sudo vim /etc/nginx/sites-available/mysite
     ```
   - Add the following configuration:
     ```
     server {
         listen 80;
         server_name mysite.local;
         root /var/www/mysite;
         index index.html;
         location / {
             try_files $uri $uri/ =404;
         }
         error_log /var/log/nginx/mysite_error.log;
         access_log /var/log/nginx/mysite_access.log;
     }
     ```
5. **Link Configuration**:
   - Link `sites-available` to `sites-enabled`:
     ```bash
     sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
     ```
   - Update `/etc/hosts` to map `mysite.local` (for local testing):
     ```bash
     sudo vim /etc/hosts
     ```
     Add:
     ```
     127.0.0.1 mysite.local
     ```
6. **Reload Nginx**:
   ```bash
   sudo systemctl reload nginx.service
   ```

## Stage 2: Simple Node.js API
Set up a Node.js API using either the built-in `http` module or the Express framework.

1. **Install Node.js**:
   ```bash
   sudo apt install -y nodejs
   sudo apt install -y npm
   ```

2. **Create API Directory**:
   ```bash
   sudo mkdir /api
   cd /api
   sudo touch index.js
   sudo npm init -y
   sudo npm install express
   ```

3. **Option 1: Using `http` Module**:
   - Create `index.js`:
     ```bash
     cat <<'EOF' > /api/index.js
     const http = require('http');
     const port = process.env.PORT || 3000;
     http.createServer((req, res) => {
       if (req.url === '/api/') {
         res.writeHead(200, {'Content-Type': 'application/json'});
         res.end(JSON.stringify({ message: 'Hello from API on port ' + port }));
       } else {
         res.writeHead(404);
         res.end();
       }
     }).listen(port, () => console.log(`API listening on ${port}`));
     EOF
     ```

4. **Option 2: Using Express Framework**:
   - Edit `index.js`:
     ```javascript
     const express = require('express');
     const app = express();
     const port = process.env.PORT || 3000;
     app.get('/api/', (req, res) => {
       res.json({ message: 'Hello Eng Mostafa from API' });
     });
     app.listen(port, () => console.log(`Server running on http://localhost:${port}`));
     ```

5. **Run the API**:
   ```bash
   PORT=3000 node /api/index.js &
   ```

## Stage 3: Reverse Proxy Server
Configure Nginx as a reverse proxy to forward requests to the Node.js API.

1. **Edit Nginx Configuration**:
   ```bash
   sudo vim /etc/nginx/sites-available/test.kandeel.local
   ```
   Add or modify:
   ```
   server {
       listen 80;
       server_name test.kandeel.local;
       location /api/ {
           proxy_pass http://127.0.0.1:3000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
       }
       location /api2/ {
           proxy_pass http://127.0.0.1:3001;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
       }
   }
   ```

2. **Update Hosts File** (for local testing):
   ```bash
   sudo vim /etc/hosts
   ```
   Add:
   ```
   127.0.0.1 test.kandeel.local
   ```

3. **Test and Reload Nginx**:
   ```bash
   sudo nginx -t
   sudo systemctl reload nginx.service
   ```

4. **Access the API**:
   - URL: `http://test.kandeel.local/api`

## Stage 4: Nginx Load Balancing
Configure Nginx to distribute traffic across multiple API instances.

1. **Create Upstream Configuration**:
   ```bash
   sudo vim /etc/nginx/conf.d/upstream-lb.conf
   ```
   Add:
   ```
   upstream api_backend {
       server 127.0.0.1:3000 weight=3;
       server 127.0.0.1:3002;
       # server 127.0.0.1:3002 down;  # Mark server as down
       # server 127.0.0.1:3002 backup;  # Backup server
   }
   ```

2. **Update Site Configuration**:
   ```bash
   sudo vim /etc/nginx/sites-available/test.kandeel.local
   ```
   Modify `location /`:
   ```
   location / {
       proxy_pass http://api_backend;
       proxy_set_header Host $host;
       proxy_set_header X-Real-IP $remote_addr;
   }
   ```

3. **Test and Reload Nginx**:
   ```bash
   sudo nginx -t
   sudo systemctl reload nginx.service
   ```

## Stage 5: Nginx Caching and Rate Limiting
Add caching and rate limiting to optimize performance and control traffic.

1. **Create Cache Directory**:
   ```bash
   sudo mkdir -p /var/cache/nginx/demo_cache
   sudo chown www-data:www-data /var/cache/nginx/demo_cache
   ```

2. **Configure Caching and Rate Limiting**:
   - Edit upstream configuration:
     ```bash
     sudo vim /etc/nginx/conf.d/upstream-lb.conf
     ```
     Add:
     ```
     limit_req_zone $binary_remote_addr zone=limit_z:10m rate=2r/s;
     proxy_cache_path /var/cache/nginx/demo_cache levels=1:2 keys_zone=demo_cache:10m inactive=5m use_temp_path=off;
     ```
   - Update site configuration:
     ```bash
     sudo vim /etc/nginx/sites-available/test.kandeel.local
     ```
     Modify `location /`:
     ```
     location / {
         proxy_pass http://api_backend;
         limit_req zone=limit_z burst=5 nodelay;
         proxy_cache demo_cache;
         proxy_cache_valid 200 30s;
         proxy_set_header Host $host;
         proxy_set_header X-Real-IP $remote_addr;
     }
     ```

3. **Test and Reload Nginx**:
   ```bash
   sudo nginx -t
   sudo systemctl reload nginx.service
   ```

## Stage 6: Setting Up SSL/TLS with Let's Encrypt
Secure your Nginx server with a free SSL/TLS certificate from Let's Encrypt using Certbot.

1. **Install Certbot**:
   ```bash
   sudo apt update
   sudo apt install -y certbot python3-certbot-nginx
   ```

2. **Obtain an SSL Certificate**:
   - Run Certbot to automatically configure SSL for your domain (replace `test.kandeel.local` with your actual domain):
     ```bash
     sudo certbot --nginx -d test.kandeel.local
     ```
   - Follow the prompts to provide an email address and agree to the terms of service.
   - Certbot will automatically update your Nginx configuration to use HTTPS (port 443) and redirect HTTP traffic to HTTPS.

3. **Verify Nginx Configuration**:
   - Certbot modifies the Nginx configuration file (e.g., `/etc/nginx/sites-available/test.kandeel.local`). The updated configuration will look like:
     ```
     server {
         listen 80;
         server_name test.kandeel.local;
         return 301 https://$host$request_uri;  # Redirect HTTP to HTTPS
     }
     server {
         listen 443 ssl;
         server_name test.kandeel.local;
         ssl_certificate /etc/letsencrypt/live/test.kandeel.local/fullchain.pem;
         ssl_certificate_key /etc/letsencrypt/live/test.kandeel.local/privkey.pem;
         include /etc/letsencrypt/options-ssl-nginx.conf;
         ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

         location / {
             proxy_pass http://api_backend;
             limit_req zone=limit_z burst=5 nodelay;
             proxy_cache demo_cache;
             proxy_cache_valid 200 30s;
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
         }
         location /api/ {
             proxy_pass http://127.0.0.1:3000;
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
         }
         location /api2/ {
             proxy_pass http://127.0.0.1:3001;
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
         }
     }
     ```

4. **Test and Reload Nginx**:
   ```bash
   sudo nginx -t
   sudo systemctl reload nginx.service
   ```

5. **Automatic Certificate Renewal**:
   - Let's Encrypt certificates are valid for 90 days. Certbot sets up a cron job or systemd timer to renew them automatically.
   - Test the renewal process:
     ```bash
     sudo certbot renew --dry-run
     ```

## Testing
- Access the static site: `https://mysite.local` (after SSL setup)
- Access the API: `https://test.kandeel.local/api`
- Test load balancing by running multiple Node.js instances (e.g., on ports 3000 and 3002).
- Verify caching and rate limiting by sending multiple requests to `https://test.kandeel.local`.
- Verify HTTPS by checking the padlock in your browser and accessing `https://test.kandeel.local`.

## Contributing
Contributions are welcome! Please follow these steps:
1. Fork the repository.
2. Create a new branch (`git checkout -b feature/your-feature-name`).
3. Commit your changes (`git commit -m 'Add some feature'`).
4. Push to the branch (`git push origin feature/your-feature-name`).
5. Open a pull request.
