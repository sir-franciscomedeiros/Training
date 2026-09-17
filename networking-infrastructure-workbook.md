# Networking and Infrastructure Training Workbook

## Audience and Goal
This workbook is for beginner-to-intermediate DevOps engineers who want practical experience with core networking and infrastructure building blocks on Ubuntu Linux. Every module explains the concept, places it in a real production architecture, and then walks through a hands-on lab with verification, troubleshooting, cleanup, and cloud/Kubernetes mapping.

## Lab Conventions
- **Primary OS:** Ubuntu 24.04 LTS or 22.04 LTS
- **Privileges:** Commands use `sudo` where required
- **Working directory:** `/opt/devops-labs`
- **Local test domain examples:** `lab.local`, `app.lab.local`, `proxy.lab.local`
- **Tools used:** Nginx, HAProxy, Squid, Redis, UFW, iptables, Apache2, Docker, curl, ss, netstat-compatible tools, tcpdump

## Suggested Lab Host Sizes
- **Single-node laptop or VM:** 2 vCPU, 4 GB RAM, 25 GB disk
- **Capstone environment:** 4 vCPU, 8 GB RAM, 40 GB disk

## Environment Preparation

### Manual Preparation
```bash
sudo apt update
sudo apt install -y nginx apache2 haproxy squid redis-server ufw iptables curl wget vim net-tools dnsutils tcpdump jq rsyslog
sudo mkdir -p /opt/devops-labs/{forward-proxy,reverse-proxy,load-balancer,firewall,caching,webserver,capstone}
```

### Docker Preparation
```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin curl jq
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
# Log out and back in before running Docker without sudo.
mkdir -p ~/devops-labs-docker && cd ~/devops-labs-docker
```

---

# Module 1: Forward Proxy

## Theory Section
### What it is
A forward proxy sits between clients and external servers. Clients send requests to the proxy, and the proxy makes the outbound request on their behalf.

### Why it exists
Organizations use forward proxies to control outbound internet access, enforce policy, inspect traffic, centralize logging, and optionally cache frequently requested content.

### Common use cases
- Corporate internet access control
- Package repository access through a controlled egress point
- Anonymous browsing from the client perspective
- URL filtering and monitoring

### Advantages
- Centralized outbound policy enforcement
- Better visibility into user traffic
- Optional content caching
- Can hide internal client IPs from external servers

### Disadvantages
- Adds another hop and operational complexity
- Can become a bottleneck or single point of failure
- HTTPS inspection is operationally sensitive
- Misconfiguration can block legitimate traffic

### Real-world examples
- Corporate employees browsing through Squid
- CI runners allowed to reach the internet only via a proxy
- Restricted environments that must log all outbound traffic

### Where it fits in production
A forward proxy is typically placed on the egress path between internal users or workloads and the internet.

## Architecture Diagrams
### Basic forward proxy flow
```text
[Client] ---> [Forward Proxy: Squid] ---> [Internet Website/API]
   |                    |                         |
   |---- HTTP request --|---- outbound request --|
   |<--- response ------|<--- response ----------|
```

### Request lifecycle
```text
1. Client sends request to proxy
2. Proxy validates ACL/policy
3. Proxy optionally caches or logs request
4. Proxy sends request to destination
5. Destination responds to proxy
6. Proxy returns response to client
```

## Hands-On Lab
### Objective
Install and configure Squid as a forward proxy and verify that a client can browse through it.

### Step 1: Install Squid manually
```bash
sudo apt update
sudo apt install -y squid
sudo systemctl enable --now squid
sudo systemctl status squid --no-pager
```

**Expected output:** `active (running)`

### Step 2: Back up the configuration
```bash
sudo cp /etc/squid/squid.conf /etc/squid/squid.conf.bak
```

### Step 3: Configure a simple ACL
Edit `/etc/squid/squid.conf` and add or adjust the access rules instead of replacing the whole file. Keep the Ubuntu defaults, then add or update the following lines near the existing ACL and `http_access` rules:
```conf
http_port 3128
acl localnet src 192.168.0.0/16
acl localnet src 10.0.0.0/8
acl localnet src 172.16.0.0/12
acl localnet src 127.0.0.1/32
http_access allow localnet
http_access deny all
access_log /var/log/squid/access.log
cache_log /var/log/squid/cache.log
```
Place the `http_access allow localnet` rule before the final deny rule so approved clients match first.

### Step 4: Validate and restart
```bash
sudo squid -k parse
sudo systemctl restart squid
```

**Expected output:** no syntax errors from `squid -k parse`

### Step 5: Test with curl
```bash
curl -x http://127.0.0.1:3128 http://example.com -I
```

**Expected output:** an HTTP response such as `HTTP/1.1 200 OK`

### Docker-based alternative
```bash
docker run -d --name squid \
  -p 3128:3128 \
  ubuntu/squid:latest
curl -x http://127.0.0.1:3128 http://example.com -I
```

### Verification steps
```bash
sudo ss -tulpn | grep 3128
sudo tail -f /var/log/squid/access.log
curl -x http://127.0.0.1:3128 http://example.com -I
```

### Cleanup steps
```bash
sudo systemctl stop squid
sudo apt remove -y squid
sudo apt autoremove -y
# Docker alternative
# docker rm -f squid
```

## Troubleshooting Section
### Common failures
- Proxy port not listening
- ACL denies the client source IP
- Syntax errors in `squid.conf`
- Firewall blocks port 3128

### Diagnostic commands
```bash
sudo squid -k parse
sudo systemctl status squid --no-pager
sudo ss -tulpn | grep 3128
sudo tail -n 50 /var/log/squid/access.log
sudo tail -n 50 /var/log/squid/cache.log
curl -v -x http://127.0.0.1:3128 http://example.com
```

### Log locations
- `/var/log/squid/access.log`
- `/var/log/squid/cache.log`

### How to validate configuration
```bash
sudo squid -k parse
```

## DevOps Perspective
- **AWS:** Squid on EC2, NAT Gateway for egress control, Network Firewall for policy
- **Azure:** Squid on VM Scale Sets, Azure Firewall for central egress control
- **GCP:** Squid on Compute Engine, Cloud NAT and VPC firewall integration
- **Kubernetes equivalent:** Egress proxy pod, service mesh egress gateway, outbound traffic policy via NetworkPolicy and gateway layers

---

# Module 2: Reverse Proxy

## Theory Section
### What it is
A reverse proxy sits in front of backend applications and receives client requests before forwarding them to internal services.

### Why it exists
It simplifies routing, TLS termination, header management, security controls, and exposure of internal services behind one public entry point.

### Common use cases
- HTTPS termination
- Routing `/api` to one app and `/app` to another
- Hiding backend server details
- Rate limiting and access control

### Advantages
- Centralized entry point
- Easier TLS certificate management
- Can add security and caching controls
- Improves backend isolation

### Disadvantages
- Another component to manage
- Misconfiguration can break all access
- Poor tuning can add latency

### Real-world examples
- Nginx in front of Node.js or Python apps
- Cloud ingress controller routing to multiple services
- API gateway patterns in microservices

### Where it fits in production
It is placed on the inbound path, usually behind a load balancer or CDN and in front of application servers.

## Architecture Diagrams
### Basic reverse proxy flow
```text
[User Browser] ---> [Reverse Proxy: Nginx] ---> [Backend App]
      |                     |                      |
      |--- HTTPS request ---|--- HTTP request ---->|
      |<-- HTTPS response --|<-- HTTP response ----|
```

### Request lifecycle
```text
1. Client reaches reverse proxy
2. Proxy terminates TLS or accepts HTTP
3. Proxy matches host/path rules
4. Proxy forwards request upstream
5. Backend responds
6. Proxy returns response to client
```

## Hands-On Lab
### Objective
Use Nginx as a reverse proxy in front of Apache.

### Step 1: Ensure Apache is running on port 8080
```bash
sudo apt install -y apache2 nginx
sudo cp /etc/apache2/ports.conf /etc/apache2/ports.conf.bak
sudo sed -i 's/Listen 80/Listen 8080/' /etc/apache2/ports.conf
sudo sed -i 's/<VirtualHost \*:80>/<VirtualHost *:8080>/' /etc/apache2/sites-available/000-default.conf
echo '<h1>Backend Apache on 8080</h1>' | sudo tee /var/www/html/index.html
sudo systemctl restart apache2
curl http://127.0.0.1:8080
```

**Expected output:** `Backend Apache on 8080`

### Step 2: Configure Nginx reverse proxy
Create `/etc/nginx/sites-available/reverse-proxy.conf`:
```conf
server {
    listen 80;
    server_name app.lab.local;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable it:
```bash
sudo ln -s /etc/nginx/sites-available/reverse-proxy.conf /etc/nginx/sites-enabled/reverse-proxy.conf
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

### Step 3: Test locally
```bash
curl -H 'Host: app.lab.local' http://127.0.0.1
```

**Expected output:** `Backend Apache on 8080`

### Docker-based alternative
```bash
docker network create proxy-lab
docker run -d --name apache-backend --network proxy-lab httpd:2.4
cat > nginx.conf <<'CONF'
events {}
http {
  server {
    listen 80;
    location / {
      proxy_pass http://apache-backend:80;
    }
  }
}
CONF
docker run -d --name nginx-rp --network proxy-lab -p 8081:80 -v $PWD/nginx.conf:/etc/nginx/nginx.conf:ro nginx:stable
curl http://127.0.0.1:8081
```

### Verification steps
```bash
sudo nginx -t
curl -I -H 'Host: app.lab.local' http://127.0.0.1
sudo tail -n 50 /var/log/nginx/access.log
```

### Cleanup steps
```bash
sudo rm -f /etc/nginx/sites-enabled/reverse-proxy.conf
sudo rm -f /etc/nginx/sites-available/reverse-proxy.conf
sudo nginx -t && sudo systemctl restart nginx
# Docker alternative
# docker rm -f nginx-rp apache-backend
# docker network rm proxy-lab
```

## Troubleshooting Section
### Common failures
- Apache still listening on port 80 instead of 8080
- Nginx syntax error
- Proxy upstream not reachable
- Wrong Host header during testing

### Diagnostic commands
```bash
sudo ss -tulpn | grep -E ':80|:8080'
sudo nginx -t
curl -v -H 'Host: app.lab.local' http://127.0.0.1
sudo journalctl -u nginx -n 50 --no-pager
sudo journalctl -u apache2 -n 50 --no-pager
```

### Log locations
- `/var/log/nginx/access.log`
- `/var/log/nginx/error.log`
- `/var/log/apache2/access.log`
- `/var/log/apache2/error.log`

### How to validate configuration
```bash
sudo nginx -t
apache2ctl configtest
```

## DevOps Perspective
- **AWS:** Application Load Balancer plus Nginx/Envoy, or ALB directly routing to targets
- **Azure:** Application Gateway, Front Door, Nginx Ingress on AKS
- **GCP:** HTTP(S) Load Balancer, Ingress for GKE
- **Kubernetes equivalent:** Ingress controller or API gateway

---

# Module 3: Load Balancer

## Theory Section
### What it is
A load balancer distributes requests across multiple backend servers.

### Why it exists
It improves availability, throughput, and fault tolerance while reducing pressure on individual application servers.

### Common use cases
- Multiple web servers behind one VIP
- Blue/green deployments
- Health-check-driven failover
- Scaling stateless applications

### Advantages
- High availability
- Better performance distribution
- Health checks and automatic failover
- Easier horizontal scaling

### Disadvantages
- More complexity
- Session persistence may be needed for stateful apps
- Bad health checks can remove good nodes or keep bad ones

### Real-world examples
- HAProxy distributing traffic to web nodes
- Cloud load balancers in front of autoscaling groups
- Kubernetes Services or Ingress balancing across pods

### Where it fits in production
Usually between the firewall edge and reverse proxy or directly in front of application nodes.

## Architecture Diagrams
### Round-robin balancing
```text
                +--> [Web1]
[Client] --> [HAProxy]
                +--> [Web2]
                +--> [Web3]
```

### Request lifecycle
```text
1. Client connects to load balancer
2. Load balancer selects healthy backend
3. Request forwarded to chosen server
4. Response returned through load balancer
5. Health checks continue in background
```

## Hands-On Lab
### Objective
Use HAProxy to distribute traffic between two Nginx backend servers.

### Step 1: Run two lightweight backends with Docker
```bash
docker network create lb-lab
docker run -d --name web1 --network lb-lab -p 8081:80 nginxdemos/hello
docker run -d --name web2 --network lb-lab -p 8082:80 nginxdemos/hello
curl http://127.0.0.1:8081 | head
curl http://127.0.0.1:8082 | head
```

### Step 2: Install and configure HAProxy
```bash
sudo apt install -y haproxy
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
```

Replace `/etc/haproxy/haproxy.cfg` with:
```conf
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    log global
    mode http
    option httplog
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend http_front
    bind *:9000
    default_backend web_pool

backend web_pool
    balance roundrobin
    option httpchk GET /
    server web1 127.0.0.1:8081 check
    server web2 127.0.0.1:8082 check
```

### Step 3: Validate and start
```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl enable --now haproxy
sudo systemctl restart haproxy
```

### Step 4: Test balancing
```bash
for i in {1..6}; do curl -s http://127.0.0.1:9000 | grep -E 'Server address|Hostname'; done
```

**Expected output:** alternating responses from both backends.

### Manual backend alternative
Use two Apache or Nginx instances on different ports and update the backend entries accordingly.

### Verification steps
```bash
sudo ss -tulpn | grep 9000
curl -I http://127.0.0.1:9000
sudo journalctl -u haproxy -n 50 --no-pager
```

### Cleanup steps
```bash
sudo systemctl stop haproxy
# Docker cleanup
# docker rm -f web1 web2
# docker network rm lb-lab
```

## Troubleshooting Section
### Common failures
- HAProxy cannot bind the frontend port
- Backends are down or incorrect
- Health checks fail due to wrong path
- Docker containers are not running

### Diagnostic commands
```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
curl -v http://127.0.0.1:9000
docker ps
sudo ss -tulpn | grep haproxy
sudo journalctl -u haproxy -n 100 --no-pager
```

### Log locations
- `/var/log/haproxy.log` if rsyslog is configured for HAProxy
- `journalctl -u haproxy`

### How to validate configuration
```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

## DevOps Perspective
- **AWS:** Application Load Balancer, Network Load Balancer
- **Azure:** Azure Load Balancer, Application Gateway
- **GCP:** Cloud Load Balancing
- **Kubernetes equivalent:** Service, Ingress, Gateway API, service mesh load balancing

---

# Module 4: Firewall

## Theory Section
### What it is
A firewall controls which network traffic is allowed or denied based on rules.

### Why it exists
It reduces attack surface by permitting only expected traffic between users, systems, and services.

### Common use cases
- Only allow SSH from admin IPs
- Expose web traffic but block direct backend access
- Restrict east-west traffic between subnets
- Control egress to required destinations only

### Advantages
- Strong access control
- Reduces unauthorized exposure
- Supports segmentation and defense in depth

### Disadvantages
- Misrules can cause outages
- Rule sprawl becomes hard to maintain
- Stateless filtering may be insufficient for complex applications

### Real-world examples
- UFW on Ubuntu hosts
- iptables rules on Linux gateways
- Security groups and NSGs in cloud environments

### Where it fits in production
At the host, subnet, VPC/VNet, and edge layers.

## Architecture Diagrams
### Firewall in front of services
```text
[Users] ---> [Firewall] ---> [Load Balancer] ---> [Apps]
              | allow 80/443 |
              | deny 22 from internet |
```

### Request lifecycle
```text
1. Packet reaches firewall
2. Firewall compares packet to rules
3. Rule matches allow or deny
4. Allowed traffic moves to target service
5. Denied traffic is dropped or rejected
```

## Hands-On Lab
### Objective
Use UFW and iptables to allow HTTP and block direct backend ports.

### Step 1: Check current status
```bash
sudo ufw status verbose
sudo iptables -L -n -v
```

### Step 2: Allow required access with UFW
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status numbered
```

### Step 3: Start a temporary backend listener
```bash
sudo nohup python3 -m http.server 8080 --bind 0.0.0.0 >/tmp/firewall-test.log 2>&1 &
sudo ss -tulpn | grep 8080
```

### Step 4: Block the test backend port with iptables
```bash
sudo iptables -A INPUT -p tcp --dport 8080 -j DROP
sudo iptables -L INPUT -n --line-numbers
```

### Step 5: Verify behavior
```bash
# Replace the example below with the externally reachable IP of this Ubuntu host.
HOST_IP=<HOST_EXTERNAL_IP>
curl -I http://127.0.0.1
# Run this from a second VM, workstation, or server on the same network:
curl -sS -o /dev/null -w '%{http_code}\n' http://$HOST_IP:8080 --max-time 3
```

**Expected output:** port 80 works; the second command returns `000` or times out because port 8080 is blocked **when tested from a different host on the network**.
Loopback traffic to `127.0.0.1` or locally generated traffic on the same Ubuntu host is not a valid firewall test for this INPUT-chain rule.

### Docker-based alternative
Apply published-port restrictions at the host firewall while containers expose services internally on a Docker network.

### Verification steps
```bash
sudo ufw status verbose
sudo iptables -L -n -v
sudo ss -tulpn
```

### Cleanup steps
```bash
sudo iptables -L INPUT -n --line-numbers
# delete the matching DROP rule number shown above, for example:
# sudo iptables -D INPUT 1
sudo pkill -f 'python3 -m http.server 8080' || true
sudo ufw disable
```

## Troubleshooting Section
### Common failures
- SSH locked out due to rule order or missing allow
- UFW inactive when expected active
- iptables rule added to wrong chain
- Cloud security group still blocks traffic

### Diagnostic commands
```bash
sudo ufw status numbered
sudo iptables -S
sudo iptables -L -n -v
sudo ss -tulpn
sudo tcpdump -ni any port 80 or port 8080
```

### Log locations
- `/var/log/ufw.log` when logging is enabled
- `journalctl` for service-related evidence

### How to validate configuration
```bash
sudo ufw status verbose
sudo iptables -S
```

## DevOps Perspective
- **AWS:** Security Groups, NACLs, AWS Network Firewall
- **Azure:** NSGs, Azure Firewall
- **GCP:** VPC firewall rules, Cloud Armor for L7 protection
- **Kubernetes equivalent:** NetworkPolicy, Calico policies, Cilium policies, ingress restrictions

---

# Module 5: Caching Server

## Theory Section
### What it is
A caching server stores frequently used data closer to consumers so repeated requests can be served faster.

### Why it exists
It reduces latency and offloads repeated work from applications and databases.

### Common use cases
- Session storage
- API response caching
- Database query result caching
- Page fragment caching

### Advantages
- Faster response times
- Lower backend load
- Better scalability

### Disadvantages
- Cache invalidation complexity
- Risk of stale data
- Memory sizing and eviction tuning required

### Real-world examples
- Redis for app cache and sessions
- Nginx proxy cache for static or semi-static pages
- CDN edge caching for static assets

### Where it fits in production
Between application and database for object cache, or between users and applications for HTTP content cache.

## Architecture Diagrams
### Redis cache flow
```text
[User] -> [App] -> [Redis Cache]
               \-> [Database]

Cache hit: App returns Redis data
Cache miss: App queries DB, stores result in Redis, returns response
```

### Request lifecycle
```text
1. User request reaches app
2. App checks Redis for cached object
3. If hit, return cached data
4. If miss, query database
5. Store result in cache with TTL
6. Return response
```

## Hands-On Lab
### Objective
Run Redis, write sample keys, and validate cache behavior.

### Step 1: Install Redis
```bash
sudo apt install -y redis-server
sudo systemctl enable --now redis-server
sudo systemctl status redis-server --no-pager
```

### Step 2: Test Redis CLI
```bash
sudo redis-cli ping
sudo redis-cli set lab:message "hello cache"
sudo redis-cli get lab:message
sudo redis-cli expire lab:message 60
sudo redis-cli ttl lab:message
```

**Expected output:** `PONG`, then `hello cache`, then a positive TTL.

### Step 3: Simulate cache hit and miss
```bash
sudo redis-cli del product:100
sudo redis-cli get product:100
sudo redis-cli setex product:100 120 '{"id":100,"name":"Demo Product"}'
sudo redis-cli get product:100
```

### Docker-based alternative
```bash
docker run -d --name redis-lab -p 6380:6379 redis:7
sudo redis-cli -h 127.0.0.1 -p 6380 ping
```

### Verification steps
```bash
sudo ss -tulpn | grep 6379
sudo redis-cli info memory | head
sudo redis-cli monitor
```

### Cleanup steps
```bash
sudo redis-cli del lab:message product:100
sudo systemctl stop redis-server
# Docker alternative
# docker rm -f redis-lab
```

## Troubleshooting Section
### Common failures
- Redis only listening on loopback when remote testing is expected
- Protected mode blocking access
- Memory pressure causing evictions
- Wrong TTL assumptions

### Diagnostic commands
```bash
sudo redis-cli ping
sudo redis-cli info server
sudo redis-cli info memory
sudo journalctl -u redis-server -n 50 --no-pager
sudo ss -tulpn | grep 6379
```

### Log locations
- `journalctl -u redis-server`
- `/var/log/redis/redis-server.log` on many Ubuntu installs

### How to validate configuration
```bash
sudo redis-cli CONFIG GET bind
sudo redis-cli CONFIG GET maxmemory-policy
```

## DevOps Perspective
- **AWS:** ElastiCache for Redis
- **Azure:** Azure Cache for Redis
- **GCP:** Memorystore for Redis
- **Kubernetes equivalent:** Redis deployment/statefulset with persistent or ephemeral design based on use case

---

# Module 6: Web Server

## Theory Section
### What it is
A web server serves web content over HTTP or HTTPS. It can deliver static files directly and may also pass dynamic requests to application runtimes.

### Why it exists
It provides the standard interface for browsers, APIs, and automation clients to access content and services.

### Common use cases
- Static site hosting
- Serving frontend assets
- TLS termination
- Acting as an origin for a reverse proxy or load balancer

### Advantages
- Efficient static content delivery
- Mature logging and access control features
- Easy integration with proxies and app runtimes

### Disadvantages
- Needs careful TLS and directory configuration
- Can expose sensitive files if misconfigured
- Performance differs based on tuning and workload type

### Real-world examples
- Apache hosting internal portals
- Nginx serving frontend assets for SPAs
- Mixed stacks where Apache serves PHP and Nginx fronts static content

### Where it fits in production
As an application origin, static asset server, or backend behind reverse proxies and load balancers.

## Architecture Diagrams
### Simple web server flow
```text
[Browser] ---> [Web Server: Apache/Nginx] ---> [HTML/CSS/JS files]
```

### Request lifecycle
```text
1. Browser requests a page
2. Web server matches host and path
3. Static file is served or dynamic request forwarded
4. Response headers and body returned
5. Access logged for audit and operations
```

## Hands-On Lab
### Objective
Deploy Apache as a simple web server and validate content delivery.

### Step 1: Install Apache
```bash
sudo apt install -y apache2
sudo systemctl enable --now apache2
sudo systemctl status apache2 --no-pager
```

### Step 2: Publish a test page
```bash
cat <<'HTML' | sudo tee /var/www/html/index.html
<!DOCTYPE html>
<html>
<head><title>DevOps Lab</title></head>
<body>
<h1>Ubuntu Web Server Lab</h1>
<p>This page is served by Apache.</p>
</body>
</html>
HTML
curl http://127.0.0.1
```

### Step 3: Enable a name-based virtual host
Create `/etc/apache2/sites-available/app.lab.local.conf`:
```conf
<VirtualHost *:80>
    ServerName app.lab.local
    DocumentRoot /var/www/html

    ErrorLog ${APACHE_LOG_DIR}/app-error.log
    CustomLog ${APACHE_LOG_DIR}/app-access.log combined
</VirtualHost>
```

Enable it:
```bash
sudo a2ensite app.lab.local.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
curl -H 'Host: app.lab.local' http://127.0.0.1
```

### Docker-based alternative
```bash
docker run -d --name apache-web -p 8085:80 httpd:2.4
curl http://127.0.0.1:8085
```

### Verification steps
```bash
sudo apache2ctl configtest
curl -I http://127.0.0.1
sudo tail -n 50 /var/log/apache2/access.log
```

### Cleanup steps
```bash
sudo a2dissite app.lab.local.conf
sudo systemctl reload apache2
# Docker alternative
# docker rm -f apache-web
```

## Troubleshooting Section
### Common failures
- Service not started
- Syntax issue in virtual host file
- DocumentRoot permissions or missing file
- Another service already using port 80

### Diagnostic commands
```bash
sudo apache2ctl configtest
sudo ss -tulpn | grep :80
curl -v http://127.0.0.1
sudo journalctl -u apache2 -n 50 --no-pager
```

### Log locations
- `/var/log/apache2/access.log`
- `/var/log/apache2/error.log`
- `/var/log/apache2/app-access.log`
- `/var/log/apache2/app-error.log`

### How to validate configuration
```bash
sudo apache2ctl configtest
```

## DevOps Perspective
- **AWS:** EC2 + Apache/Nginx, S3 + CloudFront for static sites
- **Azure:** VM Scale Sets, App Service, Front Door for edge delivery
- **GCP:** Compute Engine, Cloud Run, Cloud Storage + CDN for static sites
- **Kubernetes equivalent:** Pod serving HTTP, typically exposed through Service + Ingress

---

# Capstone Project: Production-Style Multi-Tier Environment

## Scenario
Build a small production-style platform on Ubuntu that demonstrates how security, traffic control, scaling, and observability work together.

## Target Architecture
```text
                    Internet / Users
                           |
                           v
                    [Firewall: UFW/iptables]
                           |
                           v
                 [Load Balancer: HAProxy]
                           |
                           v
                [Reverse Proxy: Nginx Layer]
                     /                   \
                    v                     v
            [Web Server 1]          [Web Server 2]
                    \                  /
                     \                /
                      v              v
                       [Redis Cache Layer]
                              |
                              v
                           [Database]
```

## Real-World Placement of Each Component
- **Firewall:** restricts exposed ports and trusted source ranges
- **Load balancer:** spreads requests across proxy or app nodes
- **Reverse proxy:** central TLS termination and routing policy
- **Web servers:** serve the application or static assets
- **Redis cache:** reduces repeated database work
- **Database:** source of truth for durable data

## Environment Setup
### Option A: Single Ubuntu host with multiple ports
- Firewall on host
- HAProxy on port 80/443 or 9000 for lab
- Nginx reverse proxy on 8088
- Apache/Nginx web servers on 8081 and 8082
- Redis on 6379
- Database placeholder on 5432 or 3306

### Option B: Docker Compose style topology
- `firewall` enforced at host
- `haproxy` container
- `nginx-rp` container
- `web1` and `web2` containers
- `redis` container
- `db` container

## Network Design
- Public ingress only to HAProxy
- Reverse proxy reachable only from HAProxy
- Web servers reachable only from reverse proxy
- Redis reachable only from web servers
- Database reachable only from app or cache tier as required

### Traffic Flow Example
```text
Client -> TCP/443 -> Firewall -> HAProxy -> Nginx Reverse Proxy -> Web1/Web2
                                                   |
                                                   +-> Redis cache lookup
                                                   |
                                                   +-> Database on cache miss
```

## Security Considerations
- Allow only required inbound ports
- Restrict management SSH by source IP
- Deny direct access to backend ports from users
- Run services as non-root where possible
- Keep packages updated
- Protect Redis and database from public exposure
- Use TLS in real production deployments

## High Availability Considerations
- Two or more web servers
- Health checks at HAProxy
- At least two reverse proxy nodes in real deployments
- Redis replication or managed cache in production
- Database backups and HA strategy

## Scaling Considerations
- Horizontal scale web servers
- Add more HAProxy or use managed load balancer
- Offload static assets to object storage/CDN
- Use Redis for hot objects and session sharing
- Separate read and write workloads in database tier

## Monitoring Considerations
- Service uptime checks
- HTTP success/error rates
- HAProxy backend health
- Nginx request latency
- Redis memory and hit rate
- Host CPU, memory, disk, and network utilization

## Logging Strategy
- `journalctl` for services
- Nginx access/error logs
- Apache access/error logs
- HAProxy logs via rsyslog/journal
- Redis logs and slowlog
- Centralize to ELK, Loki, Splunk, or cloud-native logging later

## Capstone Hands-On Build
### Step 1: Start web servers
Deploy two web servers on ports 8081 and 8082, each serving a unique page.

Example page content:
```bash
mkdir -p ~/capstone/web1 ~/capstone/web2
echo '<h1>Web Server 1</h1>' > ~/capstone/web1/index.html
echo '<h1>Web Server 2</h1>' > ~/capstone/web2/index.html
```
Each backend should return a unique `index.html` so requests to `/` clearly show which server answered.

### Step 2: Configure reverse proxy
Forward `/` to the web tier and preserve forwarding headers.

### Step 3: Configure HAProxy
Balance across reverse proxy nodes or a single reverse proxy endpoint for a smaller lab.

### Step 4: Enable firewall policy
Only expose `22`, `80`, and `443` or a lab test port such as `9000`.

### Step 5: Deploy Redis
Use Redis for a test object such as `site:banner` or `session:user123`.

### Step 6: Simulate database tier
Use a lightweight database container or document it as a protected backend dependency.

### Step 7: Validate end-to-end flow
```bash
curl -I http://127.0.0.1:9000
curl -H 'Host: app.lab.local' http://127.0.0.1:9000
redis-cli get site:banner
sudo tcpdump -ni any port 9000 or port 8088 or port 8081 or port 8082
```

## Capstone Sample Configurations
### Sample HAProxy configuration
```conf
global
    log /dev/log local0
    daemon

defaults
    mode http
    log global
    option httplog
    timeout connect 5s
    timeout client 30s
    timeout server 30s

frontend public_http
    bind *:9000
    default_backend reverse_proxy_pool

backend reverse_proxy_pool
    balance roundrobin
    option httpchk GET /
    server nginxrp1 127.0.0.1:8088 check
```

### Sample Nginx reverse proxy configuration
```conf
upstream web_tier {
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}

server {
    listen 8088;
    server_name app.lab.local;

    location / {
        proxy_pass http://web_tier;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Sample Docker Compose alternative
Create `web1/index.html` and `web2/index.html` with different page content, save the following as `compose.yaml`, store database credentials in a local `.env` file that is not committed, then run `docker compose up -d` from the same directory:
```yaml
services:
  haproxy:
    image: haproxy:2.9
    ports:
      - "9000:9000"
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    depends_on:
      - nginx-rp

  nginx-rp:
    image: nginx:stable
    ports:
      - "8088:8088"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - web1
      - web2

  web1:
    image: httpd:2.4
    volumes:
      - ./web1/index.html:/usr/local/apache2/htdocs/index.html:ro

  web2:
    image: httpd:2.4
    volumes:
      - ./web2/index.html:/usr/local/apache2/htdocs/index.html:ro

  redis:
    image: redis:7

  db:
    image: postgres:16
    env_file:
      - .env
```
Example local `.env` content on the lab host:
```text
POSTGRES_USER=labuser
POSTGRES_PASSWORD=<set-locally-on-the-lab-host>
```

## Failure Injection Exercises
### Exercise 1: Break reverse proxy routing
- Introduce a bad upstream port
- Observe `502 Bad Gateway`
- Fix the upstream and reload Nginx

### Exercise 2: Break HAProxy health checks
- Change `option httpchk GET /health` without creating `/health`
- Watch backends go DOWN
- Add a valid endpoint or revert the check path

### Exercise 3: Block app traffic with firewall
- Add an iptables drop rule on the reverse proxy port
- Confirm failure with `curl` and `tcpdump`
- Remove the rule and validate recovery

### Exercise 4: Flush Redis unexpectedly
- Delete a cache key
- Observe cache miss workflow
- Recreate the key and confirm faster retrieval

## Restore Service Exercises
- Re-enable stopped services with `systemctl`
- Validate configs with `nginx -t`, `apache2ctl configtest`, `haproxy -c`, `squid -k parse`
- Re-run `curl` tests end to end
- Confirm listening ports with `ss -tulpn`

## Validate Traffic Flow
```text
1. Confirm public entry reaches HAProxy
2. Confirm HAProxy reaches reverse proxy
3. Confirm reverse proxy reaches web nodes
4. Confirm app/web tier can reach Redis
5. Confirm only approved ports are exposed externally
```

---

# Challenge Exercises

## Beginner
1. Install Apache and publish a custom page.
2. Configure Nginx as a reverse proxy to Apache.
3. Verify Apache is no longer directly exposed publicly.

## Intermediate
1. Add HAProxy in front of two web servers.
2. Configure UFW to allow only required ports.
3. Validate load balancing and backend failover.

## Advanced
1. Add Redis and document a cache hit and miss flow.
2. Restrict Redis so only the app tier can connect.
3. Create structured troubleshooting steps for a `502` error.

## Expert
1. Build the full capstone stack.
2. Break at least four components intentionally.
3. Trace failures with logs and packet captures.
4. Restore service and document the root cause for each failure.

## Challenge Solutions
### Beginner solution outline
- Apache should serve a custom `index.html`.
- Nginx should listen on the public port and proxy to Apache on a private port such as `8080`.
- Direct user access should be tested only against Nginx, while backend exposure is blocked or unpublished.

### Intermediate solution outline
- HAProxy should listen on a single frontend port such as `9000`.
- Two backends should be registered with health checks enabled.
- UFW should allow only SSH and the chosen web port.
- Stopping one backend should still leave the service reachable through the remaining healthy node.

### Advanced solution outline
- Redis should contain a key with a TTL and demonstrate hit/miss behavior.
- Redis should bind only to internal interfaces or loopback in a single-host lab.
- A `502` should be traced first in Nginx logs, then confirmed by checking upstream reachability and listening ports.

### Expert solution outline
- The full request path should be verifiable hop by hop.
- Failures should be isolated by confirming the last healthy hop in the path.
- Recovery should include service restart, config validation, firewall rollback, and post-fix curl testing.

---

# 50 Relevant Linux Commands Used Throughout the Labs
1. `apt update`
2. `apt install`
3. `apt remove`
4. `apt autoremove`
5. `systemctl enable`
6. `systemctl start`
7. `systemctl stop`
8. `systemctl restart`
9. `systemctl reload`
10. `systemctl status`
11. `journalctl -u`
12. `ss -tulpn`
13. `ip addr`
14. `ip route`
15. `ping`
16. `curl`
17. `wget`
18. `dig`
19. `nslookup`
20. `host`
21. `tcpdump`
22. `traceroute`
23. `tracepath`
24. `netstat` *(legacy but still common)*
25. `lsof -i`
26. `ps aux`
27. `top`
28. `htop`
29. `free -m`
30. `df -h`
31. `du -sh`
32. `cat`
33. `less`
34. `tail -f`
35. `grep`
36. `awk`
37. `sed`
38. `tee`
39. `cp`
40. `mv`
41. `rm`
42. `mkdir -p`
43. `chmod`
44. `chown`
45. `ufw status`
46. `iptables -L -n -v`
47. `nginx -t`
48. `apache2ctl configtest`
49. `haproxy -c -f /etc/haproxy/haproxy.cfg`
50. `redis-cli`

---

# Interview Questions and Answers

1. **What is the difference between a forward proxy and a reverse proxy?**  
   A forward proxy represents the client to the internet, while a reverse proxy represents backend servers to clients.

2. **Why do we use load balancers?**  
   To distribute traffic, improve availability, and support horizontal scaling.

3. **What is the purpose of a firewall?**  
   To permit only approved traffic and reduce attack surface.

4. **Why is Redis commonly used in DevOps environments?**  
   It provides fast in-memory caching, sessions, counters, queues, and ephemeral shared state.

5. **What problem does a reverse proxy solve for TLS?**  
   It centralizes TLS termination so backend services do not all manage certificates independently.

6. **What is a health check in a load balancer?**  
   A periodic test that determines whether a backend should receive traffic.

7. **What is the risk of exposing Redis publicly?**  
   Unauthorized access, data leakage, misuse, and service compromise.

8. **How does caching improve application performance?**  
   By avoiding repeated expensive work such as database queries or content rendering.

9. **What does `502 Bad Gateway` usually indicate?**  
   A proxy or gateway could not get a valid response from its upstream backend.

10. **What is the difference between UFW and iptables?**  
    UFW is a simplified front end for managing firewall rules, while iptables is the lower-level packet filtering tool.

---

# Common DevOps Real-World Scenarios
- Web tier suddenly returns `502` after a backend port change
- HAProxy marks all backends down because health check path is wrong
- A firewall rule blocks SSH after a maintenance change
- Redis fills memory and starts evicting data unexpectedly
- Direct backend access is accidentally exposed to the internet
- One web node serves stale content while the other is updated
- TLS ends at the wrong layer, breaking header-aware redirects

---

# Mini Projects
1. Build a secure outbound proxy for package management.
2. Publish two internal apps behind one reverse proxy using host-based routing.
3. Create a highly available web tier behind HAProxy.
4. Add Redis-based caching to reduce repeated backend requests.
5. Harden a Ubuntu host with UFW and iptables while keeping the web app reachable.

---

# Knowledge Checks

## Quick Questions
1. Which component hides internal servers from external users?  
   **Answer:** Reverse proxy.

2. Which component controls outbound internet usage for clients?  
   **Answer:** Forward proxy.

3. Which component decides allow or deny at packet or flow level?  
   **Answer:** Firewall.

4. Which component spreads traffic across multiple servers?  
   **Answer:** Load balancer.

5. Which component stores hot data in memory for faster access?  
   **Answer:** Cache server.

## Practical Checks
- Show that only ports `22`, `80`, and `443` are reachable.
- Demonstrate a reverse proxy request reaching the backend.
- Prove HAProxy rotates or fails over to a healthy backend.
- Show a Redis key TTL decreasing over time.
- Identify the exact log that confirms a `403` or `502` event.

---

# Final Assessment

## Part 1: Build
Create a working environment on Ubuntu with:
- one firewall policy
- one HAProxy instance
- one Nginx reverse proxy
- two web servers
- one Redis cache
- one protected database tier placeholder

## Part 2: Explain
For each component, explain:
- what it does
- where it sits in the request path
- why it is needed
- what breaks if it fails

## Part 3: Troubleshoot
Intentionally introduce these failures:
- wrong backend port in reverse proxy
- blocked reverse proxy port in firewall
- failed HAProxy health check path
- deleted Redis cache key

For each failure, provide:
- symptom
- diagnostic command
- root cause
- fix
- validation after recovery

## Part 4: Architecture Review
Draw an ASCII diagram of your final environment and explain:
- ingress path
- egress path
- east-west traffic
- monitoring points
- logging points

## Part 5: Production Readiness Reflection
Answer the following:
1. How would you make this highly available across multiple hosts?
2. What would you move to managed cloud services first?
3. Which metrics and alerts are mandatory before go-live?
4. How would you secure secrets and TLS certificates?

---

# Suggested Solutions Outline
- Forward proxy should show successful proxy-mediated outbound requests in Squid logs.
- Reverse proxy should return backend content through Nginx while the backend is not directly exposed to users.
- Load balancer should distribute traffic across at least two healthy backends and fail over when one stops.
- Firewall should allow only approved ports and block direct backend access.
- Cache lab should show a Redis `PONG`, readable keys, and valid TTL behavior.
- Capstone should demonstrate successful end-to-end traffic flow from firewall through load balancer, reverse proxy, web tier, cache, and protected data tier.

## Troubleshooting Solution Pattern
1. Reproduce the problem with `curl`, browser access, or service checks.
2. Confirm the process is running with `systemctl status`, `docker ps`, or `ps aux`.
3. Confirm the service is listening on the expected port with `ss -tulpn`.
4. Validate the configuration with the native tool for that service.
5. Read the most relevant log file or `journalctl` output.
6. Trace packet flow with `tcpdump` if the symptom suggests a network block.
7. Apply the smallest corrective change.
8. Re-test from the user side and from each internal hop.

## Next Steps for Students
- Repeat all labs using hostnames instead of raw IPs
- Add TLS certificates with Let's Encrypt in a safe lab domain
- Add Prometheus and Grafana for monitoring
- Convert the capstone into Docker Compose or Kubernetes manifests
