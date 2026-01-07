# MuMo - Multi-Modal Monitoring System

MuMo is a comprehensive monitoring and data visualization platform that provides real-time sensor data collection, user management, and interactive dashboards for monitoring various environmental and operational metrics.

## Architecture Overview

MuMo consists of several interconnected components orchestrated with Docker Compose:

- **Dashboard** (PHP/Apache): Web interface for data visualization and system management
- **Pipeline** (Node.js/RDF): Data processing pipeline using RDF Connect framework  
- **Database** (MySQL): Data storage for users, sensors, and measurements
- **Nginx**: Reverse proxy and SSL termination

```
Browser → Nginx → (PHP | Adminer | Solid | Pipeline)
                    ↓
                  MySQL
```

## Project Structure

```
mumo-full/
├── dashboard/           # PHP web application
│   ├── pages/          # PHP pages and functionalities
│   ├── assets/         # CSS, JS, images
│   ├── handlers/        # API endpoints
│   └── readme.md       # Detailed dashboard documentation
├── pipeline/           # RDF data processing pipeline
│   ├── pipeline/       # TTL pipeline definitions
│   ├── proc/          # Custom processors
│   └── README.md      # Pipeline technical documentation
├── data/              # LDES data storage
├── nginx.conf         # Nginx configuration
├── docker-compose.yml # Container orchestration
└── settings.php       # Configuration file
```

## Services Overview

### nginx
- **Role:** Reverse proxy and SSL termination
- **Ports:** 80:80, 443:443 (SSL)
- **Configuration:** Routes requests to backend services

### solid
- **Role:** RDF/Linked Data pipeline with authentication
- **Technology:** RDF-Connect framework
- **Pipeline:** `./pipeline/pipeline/auth-pipeline.ttl`
- **Components:** Community Solid Server, ACL generator, Identity Provider
- **Documentation:** See [pipeline/README.md](pipeline/README.md#authentication--solid-server)

### pipeline  
- **Role:** Main data-processing RDF pipeline
- **Technology:** RDF-Connect framework
- **Pipeline:** `./pipeline/pipeline/pipeline.ttl`
- **Documentation:** See [pipeline/README.md](pipeline/README.md) for detailed architecture

### db (MySQL)
- **Role:** Relational database
- **Image:** mysql:8.0
- **Initialization:** `./dashboard/database_withDeviceChannelIndex.sql`

### php
- **Role:** PHP web backend
- **Built from:** `./dashboard/`
- **Documentation:** See [dashboard/readme.md](dashboard/readme.md) for complete setup guide

### adminer
- **Role:** Database management UI
- **Access:** Internal only (port 8080)

## Quick Start

### Prerequisites
- Docker and Docker Compose
- Git submodules support

### Installation

1. **Clone with submodules:**
```bash
git clone --recursive git@github.com:MuseumMonitoring/mumo-platform.git
cd mumo-full
git submodule update --init --recursive
```

2. **Configure settings:**
```bash
# Copy and edit settings configuration
cp settings.php.example settings.php
# See dashboard/readme.md for detailed configuration
```

3. **Start the application:**
```bash
docker-compose up --build
```

4. **Access the application:**
- Dashboard: http://localhost
- Adminer: http://localhost:8080

## Configuration

### Main Configuration
- **`settings.php`**: Database connection, domain settings, cluster name
- **`docker-compose.yml`**: Service definitions and environment variables
- **`nginx.conf`**: Reverse proxy configuration

### Detailed Setup Guides
- **Dashboard Configuration**: See [dashboard/readme.md](dashboard/readme.md#settingsphp) for complete settings.php setup
- **Pipeline Configuration**: See [pipeline/README.md](pipeline/README.md#configuration) for environment variables
- **Database Setup**: See [dashboard/readme.md](dashboard/readme.md#database-setup) for initial database configuration

## Production Deployment

This section guides you through deploying MuMo in a production environment with SSL encryption and a custom domain. The examples use `mumo.faro.be` as the domain - replace this with your actual domain throughout all configurations.

### Prerequisites

- Ubuntu/Debian server with Docker and Docker Compose installed
- Domain name pointing to your server's IP address
- Root or sudo access for SSL certificate installation

### Step 1: Install SSL Certificates with Certbot

1. **Install Certbot:**
```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
```

2. **Obtain SSL certificate** (replace with your domain):
```bash
sudo certbot --nginx -d your-domain.com
```

3. **Set up automatic renewal:**
```bash
sudo crontab -e
# Add this line for daily certificate renewal check:
0 12 * * * /usr/bin/certbot renew --quiet
```

### Step 2: Update docker-compose.yml

Modify your `docker-compose.yml` file with the following changes:

**Nginx Service - Add SSL support:**
```yaml
nginx:
  image: nginx:alpine
  ports:
    - "80:80"
    - "443:443"        # Add SSL port
  expose:
    - "443"
    - "80"
  volumes:
    - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    # Mount SSL certificates (update paths with your domain)
    - /etc/letsencrypt/live/your-domain.com/fullchain.pem:/etc/letsencrypt/live/your-domain.com/fullchain.pem
    - /etc/letsencrypt/live/your-domain.com/privkey.pem:/etc/letsencrypt/live/your-domain.com/privkey.pem
    - /etc/letsencrypt/options-ssl-nginx.conf:/etc/letsencrypt/options-ssl-nginx.conf
    - /etc/letsencrypt/ssl-dhparams.pem:/etc/letsencrypt/ssl-dhparams.pem
  networks:
    public:
      aliases:
        - api.local
        - your-domain.com    # Add your domain
```

**Solid Service - Use HTTPS baseUrl:**
```yaml
solid:
  build: ./pipeline/
  command: /app/pipeline/auth-pipeline.ttl
  environment:
    DEBUG: rdfc
    groupHistory: http://php/history.php?groups
    userHistory: http://php/history.php?users
    sensorHistory: http://php/history.php?sensors
    baseUrl: "https://your-domain.com/"  # Important: Use HTTPS for internal requests
  volumes:
    - ./data/:/app/pipeline/ldes/
    - ./pipeline/pipeline/:/app/pipeline/
```

**Adminer Service - Remove external port exposure:**
```yaml
adminer:
  image: adminer
  expose:
    - "8080"
  # Remove ports mapping for security - only accessible internally
```

### Step 3: Update nginx.conf

Replace the entire `nginx.conf` file with this SSL-enabled configuration:

```nginx
# Docker network detection - redirects internal traffic to HTTP, external to HTTPS
map $remote_addr $from_docker {
    default         1;
    172.18.0.0/16   0;
}

server {
  listen 80;
  
  # Redirect external requests to HTTPS
  if ($from_docker) {
         return 301 https://$host$request_uri;
         break;
  }
  
  location = / {
    proxy_pass http://php:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location ~ \.php$ {
    proxy_pass http://php:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location /assets {
    proxy_pass http://php:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location / {
    proxy_pass http://solid:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}

# HTTPS server block with SSL configuration
server {
  listen 443 ssl;
  # Update these paths with your domain
  ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
  include /etc/letsencrypt/options-ssl-nginx.conf;
  ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
  
  location = / {
    proxy_pass http://php:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location ~ \.php$ {
    proxy_pass http://php:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location /assets {
    proxy_pass http://php:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location / {
    proxy_pass http://solid:3000;
    proxy_set_header    Host                $http_host;
    proxy_set_header    X-Real-IP           $remote_addr;
    proxy_set_header    X-Forwarded-Ssl     on;
    proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
    proxy_set_header    X-Forwarded-Proto   $scheme;
  }
}
```

### Step 4: Update settings.php

Modify `settings.php` to use your domain and HTTPS:

```php
<?php session_start ();
if($_SERVER['HTTP_HOST'] != "db"){
  // Production settings
  $con = mysqli_connect("db", "user", "userpass", "mumo_test", 3306);
  $url = "https://your-domain.com";     // Use HTTPS
  $domain = "your-domain.com";         // Your domain
  $clustername = "Mumo";
  $logo = "assets\images\logo\mumoLogo.png";
  $logo_w = "assets\images\logo\mumoLogoW.png";
}else{
  // Development settings (can keep local or use same domain)
  $con = mysqli_connect("db", "user", "userpass", "mumo_test", 3306);
  $url = "https://your-domain.com";     # Keep consistent with production
  $domain = "your-domain.com";
  $clustername = "Mumo";
  $logo = "assets\images\logo\Mumo final.png";
  $logo_w = "assets\images\logo\Mumo final_w.png";
}
?>
```

### Step 5: Deploy the Application

1. **Build and start the containers:**
```bash
docker-compose down
docker-compose up --build -d
```

2. **Verify the deployment:**
- Access your dashboard at: `https://your-domain.com`
- Check that HTTP redirects to HTTPS
- Verify all services are running: `docker-compose ps`

### Important Notes

**Why HTTPS in docker-compose?**
The Solid container needs to make requests to itself using the public endpoint. Since the public endpoint uses HTTPS, the `baseUrl` environment variable must use `https://` even for internal Docker communications.

**Security Considerations:**
- Change default MySQL passwords in production
- Adminer is only accessible internally (good for security)
- All external traffic is forced to use HTTPS
- SSL certificates auto-renew with the cron job

**Troubleshooting:**
- If SSL certificates aren't found, verify Certbot installation and domain ownership
- Check container logs: `docker-compose logs nginx solid php`
- Ensure your domain DNS points to the server IP before running Certbot

### Security Considerations

- **Database**: Change default MySQL passwords in production
- **SSL**: Always use HTTPS in production environments
- **Firewall**: Configure firewall to allow only necessary ports (80, 443)
- **Adminer**: Consider removing or securing database admin interface in production
- **Backups**: Implement regular database and file backups

## Development

### Local Development
For development without SSL:
1. Use the default `docker-compose.yml` configuration
2. Access via http://localhost
3. Adminer available at http://localhost:8080

### Component Development
- **Dashboard Development**: See [dashboard/readme.md](dashboard/readme.md) for PHP application development
- **Pipeline Development**: See [pipeline/README.md](pipeline/README.md#usage) for RDF pipeline modifications
- **Database Schema**: See [dashboard/database.sql](dashboard/database.sql) for complete schema

### API Endpoints
The dashboard provides these endpoints used by the pipeline:
- `/history.php?users`: User history data
- `/history.php?groups`: Group history data  
- `/history.php?sensors`: Sensor history data
- `/history.php?data`: Measurement data
- `/endpoint.php`: TTN webhook endpoint
- `/export.php`: Data export functionality

## Component Interactions

All services communicate via a shared Docker network:
- **Internal URLs**: `http://php/`, `http://db:3306`, `http://solid:3000`
- **Architecture Overview**: See [Overview.drawio.svg](Overview.drawio.svg)

## Troubleshooting

### Common Issues
1. **Submodule issues**: `git submodule update --init --recursive`
2. **Database connection**: Check MySQL logs and credentials
3. **Pipeline failures**: See [pipeline/README.md](pipeline/README.md) for debugging
4. **SSL errors**: Verify certificate paths and domain configuration

### Debug Commands
```bash
docker-compose logs nginx      # Nginx/proxy issues
docker-compose logs php        # PHP application errors  
docker-compose logs db         # Database issues
docker-compose logs solid      # Pipeline problems
```

## Commands

Start the Stack:
```bash
docker compose up --build
```

Stop the stack:
```bash
docker compose down
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with `docker-compose up --build`
5. Submit a pull request

## License

This project is licensed under the terms specified in the LICENSE file.

## Support

For support and questions:
- Check the troubleshooting section
- Review container logs
- Examine the documentation in the `dashboard/documentation/` directory
