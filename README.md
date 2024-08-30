# Project Setup Guide

This guide will walk you through the process of setting up a Virtual Private Cloud (VPC), launching an EC2 instance, configuring NGINX, setting up a React app and a Node.js server, and using pm2 for process management. The guide also includes steps to create and attach multiple volumes in AWS.

## Prerequisites

- AWS account with sufficient permissions to create VPCs, EC2 instances, Elastic IPs, and volumes.
- SSH client installed on your local machine.

## Steps

### 1. Create a VPC

A Virtual Private Cloud (VPC) is required to launch your resources within a private, isolated network.

1. **Log in to AWS Management Console.**
2. **Navigate to the VPC Dashboard.**
3. **Create a VPC:**
   - Choose **Create VPC**.
   - Set a name tag for your VPC.
   - Choose the IPv4 CIDR block, e.g., `10.0.0.0/16`.
   - Choose `No IPv6 CIDR Block`.
   - Set Tenancy to `default`.
   - Click **Create VPC**.

### 2. Launch an EC2 Instance and Create a .pem Key

1. **Navigate to the EC2 Dashboard.**
2. **Launch an Instance:**

   - Click **Launch Instance**.
   - Choose an Amazon Machine Image (AMI) (e.g., Amazon Linux 2 AMI).
   - Choose an instance type (e.g., `t2.micro`).
   - Configure instance details to place it within the created VPC.
   - Add storage (default is usually fine).
   - Add tags as needed.
   - **Configure security group**: Create a new security group with SSH (port 22) and HTTP (port 80) access enabled.
   - **Key pair (login)**: Choose **Create a new key pair**. Download the `.pem` file and keep it safe.

3. **Launch the Instance**.

### 3. Create an Elastic IP

1. **Navigate to the Elastic IPs section in the EC2 Dashboard.**
2. **Allocate a new Elastic IP:**
   - Click **Allocate Elastic IP address**.
   - Allocate from the Amazon pool.
   - Click **Allocate**.
3. **Associate Elastic IP with the EC2 instance:**
   - Select the Elastic IP, then choose **Actions** > **Associate Elastic IP address**.
   - Choose the EC2 instance and network interface, then click **Associate**.

### 4. Create a Volume

1. **Navigate to the Volumes section in the EC2 Dashboard.**
2. **Create a new volume:**
   - Click **Create Volume**.
   - Select the volume type (e.g., `gp2`).
   - Set the size (e.g., 8 GiB).
   - Select the same Availability Zone as your EC2 instance.
   - Click **Create Volume**.
3. **Attach the volume to the EC2 instance:**
   - Select the volume, then choose **Actions** > **Attach Volume**.
   - Choose the instance and device name, then click **Attach**.

### 5. NGINX Setup

1. **Connect to your EC2 instance using SSH:**

   ```bash
   ssh -i "your-key.pem" ec2-user@your-ec2-public-ip
   ```

2. **Install NGINX:**

   ```bash
   sudo amazon-linux-extras install nginx1 -y
   sudo systemctl start nginx
   sudo systemctl enable nginx
   ```

3. **Verify NGINX is running:**

   ```bash
   curl localhost
   ```

### 6. Create a New Volume and Mount it to `/test`

1. **Create a new volume** (Repeat Step 4).
2. **Attach the volume to the EC2 instance**.
3. **Format the volume and mount it to `/test`:**

   ```bash
   sudo mkfs -t ext4 /dev/xvdf  # Replace with your volume's device name
   sudo mkdir /test
   sudo mount /dev/xvdf /test
   ```

4. **Ensure the volume is mounted on reboot:**

   ```bash
   echo '/dev/xvdf /test ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
   ```

### 7. NGINX Default File Setup

1. **Backup the default NGINX configuration file:**

   ```bash
   sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup
   ```

2. **Modify the NGINX default file:**

   ```bash
   sudo nano /etc/nginx/nginx.conf
   ```

3. **Update server block to proxy requests:**

   ```nginx
   server {
       listen 80;
       server_name YOUR_ELASTIC_IP OR DOMAN_NAME;

       # Proxy for React Build
       location /admin/ {
           root /var/www/web;
           try_files $uri /index.html;
       }

       # Proxy for React (port 3000)
       location /client/ {
           proxy_pass http://localhost:3000/;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }

        # Proxy for Node.js server (port 8000)
        location /api/ {
           proxy_pass http://localhost:8000/;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
        }
   }
   ```

4. **Restart NGINX:**

   ```bash
   sudo systemctl restart nginx
   ```

### 8. Create React App at `/var/www/web`

1. **Install Node.js and npm:**

   ```bash
   curl -sL https://rpm.nodesource.com/setup_14.x | sudo bash -
   sudo yum install -y nodejs
   ```

2. **Create a React app:**

   ```bash
   sudo mkdir -p /var/www/web
   cd /var/www/web
   sudo npx create-react-app my-app
   ```

### 9. Create Node Server at `/var/www/server`

1. **Create a directory for the Node.js server:**

   ```bash
   sudo mkdir -p /var/www/server
   cd /var/www/server
   ```

2. **Initialize a new Node.js project:**

   ```bash
   sudo npm init -y
   ```

3. **Install Express.js:**

   ```bash
   sudo npm install express
   ```

4. **Create the server file:**

   ```bash
   sudo nano index.js
   ```

5. **Add the following server code to `index.js`:**

   ```javascript
   const express = require("express");
   const app = express();
   const PORT = 3000;

   app.get("/api", (req, res) => {
     res.send("Hello from the Node.js server!");
   });

   app.listen(PORT, () => {
     console.log(`Server is running on http://localhost:${PORT}`);
   });
   ```

### 10. Configure NGINX Default File to Serve React and Node.js

This step was already covered in Step 7 under the NGINX configuration. Ensure the correct paths are set in the NGINX configuration file.

### 11. Setup PM2 in EC2

1. **Install pm2 globally:**

   ```bash
   sudo npm install pm2 -g
   ```

2. **Start the Node.js server using pm2:**

   ```bash
   cd /var/www/server
   pm2 start index.js
   ```

3. **Save the pm2 process list and resurrect pm2 on reboot:**

   ```bash
   pm2 save
   pm2 startup
   ```

# Basic and Advance Commands of Linux

### 1. Connect to instance

```bash
ssh -i "your-key.pem" your-instancename@user-public-ip
```

### 2. Run command as root user

If you dont wont to use sudo in any of command then go as the root user this will help you to run the command without using sudo

```bash
sudo -i
```
