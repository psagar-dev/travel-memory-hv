## ✈️ TravelMemory – Fullstack Deployment on AWS

This project deploys a MERN stack Travel Memory application using AWS services such as EC2, Application Load Balancer (ALB), Auto Scaling Groups (ASG), Target Groups, and Cloudflare for enhanced security and DNS management.

---

#### 📁 Project Overview

TravelMemory is a full-stack application that consists of:
- **Backend**: Node.js/Express REST API connected to MongoDB Atlas
- **Frontend**: React application built
- **Deployment**: AWS infrastructure with high availability and scalability features

---

#### 📁 Repository Structure

```
TravelMemory/
│
├── backend/              # Node.js Express backend
├── frontend/             # React frontend
├── setup-backend.sh      # Backend provisioning script (Auto Scaling)
|── setup-frontend.sh     # Frontend EC2 static deployment script
├── images/               # Documentation images (AWS console screenshots)
│   └── all screenshort
```

---

#### 🏗️ Architecture

The application is deployed with the following AWS architecture:
- **Frontend**: EC2 instance running Nginx to serve static React files
- **Backend**: Auto Scaling Group of EC2 instances behind Application Load Balancer
- **Database**: MongoDB Atlas cloud database
- **DNS & Security**: Cloudflare for DNS management and DDoS protection

---

#### ⚙️ Prerequisites

Before deploying this application, ensure the following:

- ✅ A valid AWS account with permissions to create EC2, ALB, ASG, and Target Groups
- ✅ Domain access via Cloudflare (e.g., `travelmemory.example.com` and `tmapi.example.com`)
- ✅ MongoDB Atlas cluster (URI is configured in backend `.env`)
- ✅ Security group rules allowing HTTP (80), HTTPS (443), and backend port (3000)
- ✅ Ubuntu-based EC2 AMI with internet access for installing dependencies

---

#### 🛠 Deployment Instructions

##### 📦 Required Services & Dependencies

The deployment scripts automatically install these dependencies:
- **Nginx** – Reverse proxy for both backend and frontend
- **Node.js v22** – JavaScript runtime environment
- **PM2** – Process manager for running the backend as a service
- **Git** – For cloning the application repository

#### 🧱 Deployment Architecture Diagram
![Security Group 1](/images/travelmemory-diagram.svg)
---
## 🚀 Backend Deployment


### 🔒 AWS Security Group Configuration

This document details the security group configuration used for the TravelMemory application deployment on AWS. Security groups function as virtual firewalls controlling inbound and outbound traffic to your AWS resources.

![Security Group 1](/images/security-group.png)

#### 📋 Security Group Details

| Property | Value |
|----------|-------|
| **Name** | tm-sagar-sg |
| **Description** | Travel Memory Security Group |
| **VPC** | vpc-456afd9442d7f8ba (default-vpc-1aeb1c-f10) |

---

#### 🔐 Inbound Rules Configuration

Security groups control which traffic is allowed to reach your EC2 instances. Below are the inbound rules configured for the TravelMemory application:

| Protocol | Port Range | Source | Description |
|----------|------------|--------|-------------|
| HTTP | 80 | 0.0.0.0/0 | Allow HTTP traffic from anywhere |
| TCP Custom | 3000 | 0.0.0.0/0 | Allow traffic |
| HTTPS | 443 | 0.0.0.0/0 | Allow HTTPS traffic from anywhere |
| SSH | 22 | 0.0.0.0/0 | Allow SSH access for management |

> ⚠️ **Security Note**: Rules with source of 0.0.0.0/0 allow all IP addresses to access your instance. We recommend limiting access to known IP addresses only.

---

#### 📤 Outbound Rules Configuration

Outbound rules control the traffic that's allowed to leave your instances:

| Type | Protocol | Port Range | Destination | Description |
|------|----------|------------|-------------|-------------|
| All traffic | All | All | 0.0.0.0/0 | Allow all outbound traffic |

> ⚠️ **Security Note**: Rules with destination of 0.0.0.0/0 allow your instances to send traffic to any IP address. For production environments, consider implementing more restrictive outbound rules.

---
### ✈️ AWS Launch Template for TravelMemory Application

This document describes the AWS Launch Template configuration used for the TravelMemory application's backend deployment. The launch template provides a reusable configuration for creating EC2 instances with consistent settings and automated provisioning.

![Create Template 1](/images/create-template-1.png)
![Create Template 2](/images/create-template-2.png)

#### 📝 Launch Template Details

| Property | Value |
|----------|-------|
| **Template Name** | tm-sagar-template |
| **Description** | Launch template for Travel Memory backend instances |
| **Version** | v1 |
| **Auto Scaling Guidance** | Compatible with EC2 Auto Scaling |

#### 💻 Instance Configuration

| Setting | Value |
|---------|-------|
| **AMI** | Ubuntu Server 24.04 LTS (HVM), SSD Volume Type |
| **AMI ID** | ami-0e35dadd555f55f67 |
| **Architecture** | 64-bit (x86) |
| **Instance Type** | t2.micro (Free tier eligible) |
| **Key Pair** | sagar-b-10 |

#### 🔒 Security Configuration

The launch template uses the security group `tm-sagar-sg`

#### 💾 Storage Configuration

| Property | Value |
|----------|-------|
| **Volume Type** | EBS |
| **Device Name** | /dev/sda1 |
| **Size** | 8 GB |
| **Volume Type** | gp3 |
| **IOPS** | 2000 (default) |
| **Delete on Termination** | Yes |
| **Encrypted** | No |

#### 📜 User Data Script

The following bootstrapping script runs automatically when instances are launched:
**Note:** Please add your `MONGO_URI`
```bash
#!/bin/bash
# === System Update ===
apt-get update

# === Install Node.js v22 ===
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
apt-get install -y nodejs

# === Install Nginx ===
apt-get install -y nginx

# === Set permission ===
cd /var/www/
chmod -R 775 /var/www/

# === Clone Backend Repo to /var/www ===
git clone https://github.com/psagar-dev/travel-memory.git
cd travel-memory/backend

# === Create .env File & put data ===
echo "PORT=3000" > .env
echo "MONGO_URI='Your Mongo URl'" >> .env

# === Install App Dependencies & PM2 ===
npm install
npm install pm2 -g
pm2 start index.js --name "travel-memory-backend"
pm2 startup
pm2 save

# === Configure Nginx ===
rm /etc/nginx/sites-enabled/default

bash -c 'cat > /etc/nginx/sites-available/travel-memory << EOF
server {
    listen 80;
    server_name _;
    root /var/www/html;
    index index.html index.nginx-debian.html;
    location / {
        try_files \$uri \$uri/ /index.html;
    }
    location /api/ {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host \$host;
        proxy_cache_bypass \$http_upgrade;
    }
}
EOF'

ln -s /etc/nginx/sites-available/travel-memory /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
systemctl restart nginx
```
---
### 🎯 AWS Target Group Configuration

#### 📋 Overview
This document describes the process of creating a target group in AWS, which allows you to route traffic to registered targets and perform health checks on those targets.

![Create Template 2](/images/target-groups.png)

#### Target Group Configuration Steps

##### 🔰 1. Basic Configuration
- **Target Type Selection**: "Instances" is currently selected
  - Supports load balancing to instances within a specific VPC
  - Facilitates the use of Amazon EC2 Auto Scaling and EC2 capacity

##### 🎯 2. Target Group Details
| Field             | Value           |
|-------------------|------------------|
| **Name**          | `tm-sagar-tg`     |
| **Protocol/Port** | HTTP on port 80   |
| **IP Address Type** | IPv4            |


##### 🌐 3. VPC Configuration
- Selected VPC: default-vpc-batch-10 (vpc-05c6b6be327f01ea)
- Only VPCs that support the IP address type selected will appear in the list

##### 📡 4. Protocol Version
- HTTP/1.1 is selected
- Other options include:
  - HTTP/2
  - gRPC

##### 🏥 5. Health Check Settings
| Field                 | Value                                                      |
|-----------------------|------------------------------------------------------------|
| **Protocol**          | HTTP                                                       |
| **Path**              | "/" (default path for health checks)                       |
| **Port**              | Traffic port (uses the same port as the target group)      |
| **Healthy threshold** | 5 consecutive successful checks required                   |
| **Unhealthy threshold** | 2 consecutive failed checks required                    |
| **Timeout**           | 5 seconds before considering a health check failed         |
| **Interval**          | 30 seconds between health checks                           |
| **Success codes**     | 200 (HTTP codes to use when checking for a successful response) |

#### Next Steps
After configuring the target group, you'll proceed to register targets to this group in the next step.

![Target Group](/images/target-group-detail.png)
---

### ⚖️ AWS Application Load Balancer Configuration

![Target Group](/images/alb-1.png)

#### 🔧 Basic configuration

| Field                 | Value               |
|-----------------------|---------------------|
| **Load balancer name** | tm-alb-sagar-backend |
| **Scheme**            | Internet-facing     |
| **IP address type**   | IPv4                |


#### 📋 Network mapping

*   **VPC:** default-vpc-batch-10 (vpc-0056d809452f9f8ea)
*   **Availability Zones and subnets:**
    *   ap-south-1a: subnet-0d08b704a1b1af340
    *   ap-south-1b: subnet-0c1842aca7dc1b6b0
    *   ap-south-1c: subnet-066ec2d046612e19b

#### 🛡️ Security groups

*   **Selected security group:** tm-sagar-sg (sg-0d58f1b05fe6632a5)

#### 🎯 Listeners and routing

*   **Listener:** HTTP:80
    *   **Default action:** Forward to `tm-sagar-tg` (Target group)

![Target Group](/images/alb-detail.png)
![Target Group](/images/alb-resource.png)

---

### 🔄 AWS Auto Scaling Group Configuration Review

![Target Group](/images/asg-review.png)
#### 📝 Step 1: Choose launch template

*   **Group details:**
    *   Auto Scaling group name: `tm-sagar-asg`
*   **Launch template:**
    *   Launch template: `tm-sagar-template` (lt-064702ee8b710b2eb)
    *   Version: Default
    *   Description: v1

#### 🌐 Step 2: Choose instance launch options

*   **Network:**
    *   VPC: `vpc-0056d809452f9f8ea`
    *   **Availability Zones and subnets:**
        *   ap-south-1a: `subnet-0a69df11f8db6250d` (172.31.48.0/20)
        *   ap-south-1a: `subnet-0a08b704a1b1af340` (172.31.32.0/20)
        *   ap-south-1a: `subnet-0cb24de117fbe1893` (172.31.112.0/20)
        *   ap-south-1a: `subnet-083a8c50c53d0402d` (172.31.64.0/20)
        *   ap-south-1a: `subnet-03003e7cd7eed6630` (172.31.96.0/20)
    *   Availability Zone distribution: Balanced best effort
*   **Instance type requirements:**
    *   This Auto Scaling group will adhere to the launch template.

#### 🔗 Step 3: Integrate with other services

*   **Load balancing:**
    *   Load balancer 1:
        *   Name: `tm-alb-sagar-backend`
        *   Type: Application/HTTP
        *   Target group: `tm-sagar-tg`
*   **VPC Lattice integration options:**
    *   VPC Lattice target groups: (None specified)
*   **Application Recovery Controller (ARC) zonal shift:**
    *   ARC zonal shift: Disabled
*   **Health checks:**
    *   Health check type: EC2, ELB
    *   Health check grace period: 300 seconds

#### ⚖️ Step 4: Configure group size and scaling policies
##### Group Size

| Field                   | Value                           |
|-------------------------|---------------------------------|
| Desired capacity        | 2                               |
| Desired capacity type   | Units (number of instances)     |

##### Scaling

| Field                     | Value             |
|---------------------------|-------------------|
| Minimum desired capacity  | 1                 |
| Maximum desired capacity  | 3                 |
| Target tracking policy    | (None specified)  |

##### Instance Maintenance Policy

| Field                    | Value       |
|--------------------------|-------------|
| Replacement behavior     | No policy   |
| Min healthy percentage   | -           |
| Max healthy percentage   | -           |

##### Additional Settings

| Field                          | Value     |
|--------------------------------|-----------|
| Instance scale-in protection   | Disabled  |
| Monitoring                     | Disabled  |
| Default instance warmup        | Disabled  |

##### Capacity Reservation Preference

| Field                          | Value     |
|--------------------------------|-----------|
| Preference                     | Default   |
| Capacity Reservation IDs       | -         |
| Resource Groups                | -         |

#### 🔔 Step 5: Add notifications
*   **Notifications:** No notifications

#### 🏷️ Step 6: Add tags
*   **Tags (0):** No tags

#### 📊 Monitoring & Management
- ASG Instance Management Dashboard
![Auto Scaling Group Management](/images/asg-instance-management.png)

- Auto Scaling Group Activity History
![Auto Scaling Activity](/images/asg-activity.png)


#### 🌐 DNS Configuration
##### Cloudflare CNAME Setup
Configure the API endpoint using CNAME record:
- API subdomain CNAME record in Cloudflare
![Cloudflare CNAME Configuration](/images/cname-cloudflare.png)

---

### 🌐 Frontend Deployment Flow
##### 💻 EC2 Instance Configuration

This document outlines the configuration used to launch the `tm-sagar-frontend` EC2 instance.

##### ⚙️ Instance Settings
| Field                        | Value                                                                                                  |
|------------------------------|--------------------------------------------------------------------------------------------------------|
| **Name**                     | tm-sagar-frontend                                                                                      |
| **AMI**                      | Ubuntu Server 24.04 LTS (ami-0e35ddab05955cf57) - Free tier eligible                                   |
| **Instance Type**            | t2.micro - Free tier eligible                                                                          |
| **Key Pair**                 | sagar-b10                                                                                              |
| **Network**                  | vpc-0056d809452f9f8ea (default-vpc-batch-10)                                                           |
| **Subnet**                   | No preference                                                                                          |
| **Auto-assign Public IP**    | Enabled                                                                                               |
| **Firewall (Security Group)**| tm-sagar-sg (sg-0d58f1b05fe6652a3) - Existing security group selected. Ensure this group allows inbound traffic on port 80 (HTTP) and potentially port 22 (SSH) from your IP. |
| **Storage**                  | 8 GiB gp3 Root Volume - Free tier eligible                                                             |
| **Advanced Details**         | Default settings were mostly used, except for the User Data                                           |


#### User Data Script

The following script was provided in the User Data section to automatically configure the instance upon launch:

```bash
#!/bin/bash
# === System Update ===
apt-get update

# === Install Node.js v22 ===
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
apt-get install -y nodejs

# === Install Nginx ===
apt-get install -y nginx

# === Set permission ===
cd /var/www/
chmod -R 775 /var/www/

# === Clone Backend Repo to /var/www ===
git clone https://github.com/psagar-dev/travel-memory.git
cd travel-memory/frontend

# === Create .env File & put data ===
echo "REACT_APP_BACKEND_URL=http://tmapi.example.com" > .env

# === Install App Dependencies ===
npm install
npm run build

# === Copy Build Output to /var/www/html ===
rm -rf /var/www/html/*
cp -r build/* /var/www/html/

# === Configure Nginx for Static Site ===
rm -f /etc/nginx/sites-enabled/default
bash -c 'cat > /etc/nginx/sites-available/tm.example.com << EOF
server {
    listen 80;
    server_name _;

    root /var/www/html;
    index index.html;

    location ~ /\.(?!well-known).* {
        deny all;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
EOF'

ln -s /etc/nginx/sites-available/tm.patelsagar.com /etc/nginx/sites-enabled/

nginx -t
systemctl reload nginx
systemctl restart nginx
```

#### 🖼️ Frontend Configuration
- Cloudflare DNS settings for the frontend application
![Frontend DNS Configuration](/images/frontend-cname.png)

#### 📸 Application Screenshot
- TravelMemory application running in production environment
![TravelMemory Application Demo](/images/frontend.png)


## 📜 Project Information

### 📄 License Details
This project is released under the MIT License, granting you the freedom to:
- 🔓 Use in commercial projects
- 🔄 Modify and redistribute
- 📚 Use as educational material

## 📞 Contact

📧 Email: [Email Me](securelooper@gmail.com
)
🔗 LinkedIn: [LinkedIn Profile](https://www.linkedin.com/in/sagar-93-patel)  
🐙 GitHub: [GitHub Profile](https://github.com/psagar-dev)  

---

<div align="center">
  <p>Built with ❤️ by Sagar Patel</p>
</div>