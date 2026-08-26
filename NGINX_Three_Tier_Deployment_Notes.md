# NGINX Three-Tier Application Deployment Notes

## 1. What is NGINX?

NGINX is an open-source web server that also works as a reverse proxy,
load balancer, SSL termination server, and API gateway.

**Common Uses** - Host static websites (HTML, React, Angular, Vue). -
Reverse proxy for backend applications. - Load balance traffic across
multiple backend servers. - Enable HTTPS using SSL certificates.

------------------------------------------------------------------------

## 2. Three-Tier Architecture

``` text
                Internet
                    |
                    | HTTPS (443)
                    v
              NGINX Server (EC2)
                    |
          +---------+---------+
          |                   |
          |                   |
     React Frontend       Backend API
   /usr/share/nginx/html   Node.js (EC2)
                                |
                                |
                            MongoDB Atlas
```

### Three Tiers

  Tier                Component         Purpose
  ------------------- ----------------- -----------------------
  Presentation Tier   React Frontend    User Interface
  Application Tier    Node.js Backend   Business Logic & APIs
  Data Tier           MongoDB Atlas     Database

NGINX sits in front of the application and routes traffic.

------------------------------------------------------------------------

## 3. Reverse Proxy

A reverse proxy receives requests from users and forwards them to
backend servers.

### Flow

``` text
Browser
   |
   | https://kuldeep.website/api/AdminReg
   v
NGINX
   |
   | proxy_pass
   v
Backend EC2 (98.91.225.124:5000)
   |
   v
Node.js
```

### NGINX Reverse Proxy Example

``` nginx
location /api/ {
    proxy_pass http://98.91.225.124:5000;
}
```

### Why Reverse Proxy?

-   Hide backend IP.
-   Users access only one domain.
-   Handle SSL in one place.
-   Central routing for frontend and backend.

------------------------------------------------------------------------

## 4. Load Balancer in NGINX

NGINX distributes traffic between multiple backend servers.

### Upstream Block

``` nginx
upstream backend_servers {
    server 54.224.56.116:80;
    server 54.172.138.220:80 backup;
}
```

### How it Works

-   Primary backend: `54.224.56.116`
-   Backup backend: `54.172.138.220`
-   If the primary server is unavailable, NGINX forwards traffic to the
    backup server.

### Request Flow

``` text
Client
   |
   v
NGINX
   |
   +----------------------+
   |                      |
Primary Backend      Backup Backend
54.224.56.116        54.172.138.220
```

### Using Both Servers (Round Robin)

``` nginx
upstream backend_servers {
    server 54.224.56.116;
    server 54.172.138.220;
}
```

NGINX sends requests alternately between both servers.

------------------------------------------------------------------------

## 5. SSL (Secure Sockets Layer)

**SSL Full Form:** Secure Sockets Layer

SSL encrypts communication between the browser and server.

### HTTP vs HTTPS

  HTTP            HTTPS
  --------------- -----------
  Port 80         Port 443
  Not encrypted   Encrypted
  Less secure     Secure

### SSL Certificate Location

``` nginx
ssl_certificate /etc/letsencrypt/live/kuldeep.website/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/kuldeep.website/privkey.pem;
ssl_protocols TLSv1.2 TLSv1.3;
```

### Meaning of Files

  File              Purpose
  ----------------- --------------------------------------------
  fullchain.pem     Public SSL certificate + certificate chain
  privkey.pem       Private key for SSL
  TLSv1.2 TLSv1.3   Allowed TLS versions

------------------------------------------------------------------------

## 6. Complete NGINX Configuration Example

``` nginx
events {}

http {

    upstream backend_servers {
        server 54.224.56.116:80;
        server 54.172.138.220:80 backup;
    }

    server {
        listen 443 ssl http2;

        root /usr/share/nginx/html;

        ssl_certificate /etc/letsencrypt/live/kuldeep.website/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/kuldeep.website/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;

        access_log /var/log/nginx/exaple.com.access.log;
        error_log /var/log/nginx/exaple.com.error.log;

        error_page 404 /404.html;

        location / {
            try_files $uri $uri/ /index.html;
        }

        location /api/ {
            proxy_pass http://98.91.225.124:5000;
        }
    }
}
```

------------------------------------------------------------------------

## 7. Explanation of Each Directive

### `listen 443 ssl http2`

-   Listen on HTTPS port **443**.
-   Enable SSL.
-   Enable HTTP/2.

### `root /usr/share/nginx/html`

Directory containing React build files.

### `location /`

``` nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

Used for React Single Page Applications.

If a file exists, NGINX serves it. Otherwise it serves `index.html`,
allowing React Router to handle routes like `/AdminReg`.

### `location /api/`

``` nginx
location /api/ {
    proxy_pass http://98.91.225.124:5000;
}
```

Forwards API requests to the backend Node.js application.

------------------------------------------------------------------------

## 8. Deploying React Frontend on NGINX

### Step 1 -- Build React

``` bash
npm run build
```

### Step 2 -- Copy Build to VM

``` bash
scp -i ~/nginx.pem -r build/* ubuntu@3.81.214.242:/tmp/build/
```

### Step 3 -- Copy to NGINX Directory

``` bash
sudo rm -rf /usr/share/nginx/html/*
sudo cp -r /tmp/build/* /usr/share/nginx/html/
```

### Step 4 -- Reload NGINX

``` bash
sudo nginx -t
sudo systemctl reload nginx
```

------------------------------------------------------------------------

## 9. Deploying Node.js Backend on EC2

### Copy Backend

``` bash
scp -i ~/nginx.pem -r backend/* ubuntu@BACKEND_EC2_IP:/home/ubuntu/backend/
```

### Install Dependencies

``` bash
cd backend
npm install
```

### Start Backend

``` bash
npm start
```

Backend runs on **Port 5000**.

------------------------------------------------------------------------

## 10. Frontend + Backend Request Flow

``` text
Browser
   |
   | GET /
   v
NGINX
   |
   v
React Frontend

Browser
   |
   | POST /api/AdminReg
   v
NGINX
   |
   | proxy_pass
   v
Backend EC2 (Node.js)
   |
   v
MongoDB Atlas
```

------------------------------------------------------------------------

## 11. Useful NGINX Commands

### Test Configuration

``` bash
sudo nginx -t
```

### Reload NGINX

``` bash
sudo systemctl reload nginx
```

### Restart NGINX

``` bash
sudo systemctl restart nginx
```

### Check Status

``` bash
sudo systemctl status nginx
```

### View Configuration

``` bash
cat /etc/nginx/nginx.conf
```

------------------------------------------------------------------------

## 12. Useful SSL Commands

### Obtain Certificate

``` bash
sudo certbot certonly --standalone -d kuldeep.website
```

### Certificate Location

``` bash
/etc/letsencrypt/live/kuldeep.website/
```

Files:

``` text
fullchain.pem
privkey.pem
cert.pem
chain.pem
```

------------------------------------------------------------------------

## 13. Troubleshooting

### 404 Not Found

**Reason:** React route handled as a static file.

**Fix**

``` nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

### 405 Method Not Allowed

**Reason:** Frontend called the wrong API route or NGINX routed to the
frontend.

**Fix**

Use:

``` text
POST /api/AdminReg
```

and proxy `/api/` to the backend.

### 504 Gateway Timeout

**Reason:** NGINX cannot reach the backend server.

**Check** - Backend Node.js is running. - Backend listens on port
5000. - Security Group allows port 5000 from the NGINX server.

------------------------------------------------------------------------

## 14. Interview Notes

### What is NGINX?

NGINX is a web server that also acts as a reverse proxy, load balancer,
and SSL termination server.

### What is a Reverse Proxy?

A reverse proxy receives client requests and forwards them to backend
servers without exposing backend IP addresses.

### What is Load Balancing?

Load balancing distributes incoming traffic across multiple backend
servers to improve availability and performance.

### What is SSL?

SSL (Secure Sockets Layer) encrypts communication between client and
server using HTTPS on port 443.

### Difference Between Reverse Proxy and Load Balancer

  -----------------------------------------------------------------------
  Reverse Proxy                        Load Balancer
  ------------------------------------ ----------------------------------
  Routes requests to backend servers.  Distributes traffic across
                                       multiple backend servers.

  Hides backend server details.        Improves availability and
                                       scalability.

  Can proxy a single backend.          Works with one or many backend
                                       servers.
  -----------------------------------------------------------------------
