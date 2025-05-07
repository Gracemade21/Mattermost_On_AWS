## Mattermost_On_AWS
This guide walks you through setting up a secure, self-hosted Mattermost chat application on AWS EC2. The architecture includes:  A public subnet hosting the Mattermost web application.  A private subnet hosting the MySQL database.  A NAT gateway to allow the private subnet to access the internet securely.
# Deploying Mattermost and MySQL on AWS EC2

This guide walks you through setting up a secure, self-hosted Mattermost chat application on AWS EC2. The architecture includes:

- A **public subnet** hosting the Mattermost web application.
- A **private subnet** hosting the MySQL database.
- A **NAT instance** to allow the private subnet to access the internet securely.

## 📌 Prerequisites

- An active AWS account.
- Basic knowledge of AWS services (VPC, EC2, Subnets, sql database, Security Groups).
- SSH access to EC2 instances.

## 🗺️ Architecture Overview
![image](https://github.com/user-attachments/assets/a4d139de-869c-4c2e-b66d-eab501617489)

The deployment consists of:

- A Virtual Private Cloud (VPC) with CIDR block `10.0.0.0/16`.
- A public subnet (`10.0.1.0/24`) for the Mattermost server and NAT instance.
- A private subnet (`10.0.2.0/24`) for the MySQL database.
- An Internet Gateway attached to the VPC.
- Route tables configured for public and private subnets.
- Security groups to control inbound and outbound traffic.

## 🛠️ Deployment Steps

### 1. Create a VPC

- Navigate to the VPC dashboard in AWS.
- Create a new VPC with CIDR block `10.0.0.0/16`.
- Enable DNS hostnames for the VPC.

### 2. Create Subnets

- **Public Subnet**:
  - CIDR block: `10.0.1.0/24`.
  - Enable auto-assign public IPv4 addresses.

- **Private Subnet**:
  - CIDR block: `10.0.2.0/24`.
  - Do not enable auto-assign public IPv4 addresses.

### 3. Set Up Internet Gateway

- Create an Internet Gateway.
- Attach it to your VPC.

### 4. Configure Route Tables

- **Public Route Table**:
  - Associate it with the public subnet.
  - Add a route: Destination `0.0.0.0/0`, Target: Internet Gateway.

- **Private Route Table**:
  - Associate it with the private subnet.
  - Add a route: Destination `0.0.0.0/0`, Target: NAT instance (to be created).

### 5. Launch EC2 Instances

- **NAT Instance**:
  - Launch in the public subnet.
  - Assign an Elastic IP.
  - Configure as a NAT instance.

- **MySQL Database Instance**:
  - Launch in the private subnet.
  - Use Ubuntu 20.04 LTS AMI.

- **Mattermost Application Instance**:
  - Launch in the public subnet.
  - Use Ubuntu 20.04 LTS AMI.

### 6. Install and Configure MySQL

- SSH into the MySQL instance:
  ```bash
  ssh -i MySQL.pem ubuntu@<private-IP>

- Update the system:
  ```bash
  sudo apt update && sudo apt upgrade -y

- Create the MySQL installation script on EC2:
  ```bash
  vim setup-mysql.sh

- Paste the MySQL installation script on EC2:
  ```bash
  #!/bin/bash

  # Exit immediately if a command exits with a non-zero status
  set -e
  
  # Update package list and install MySQL Server
  apt update -y
  apt install -y mysql-server
  
  echo "[INFO] MySQL installed."
  
  # Start MySQL service and enable it on boot
  systemctl start mysql
  systemctl enable mysql
  sleep 5
  
  echo "[INFO] Configuring MySQL..."
  
  # Secure MySQL and configure user/database
  mysql -u root <<-EOF
  -- Set root password and use native authentication
  ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'StrongPasswordHere';
  
  -- Remove unnecessary users and test databases
  DELETE FROM mysql.user WHERE User='' OR (User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1'));
  DROP DATABASE IF EXISTS test;
  DELETE FROM mysql.db WHERE Db='test' OR Db='test_%';
  FLUSH PRIVILEGES;
  
  -- Create application user and database
  CREATE USER 'mmuser'@'%' IDENTIFIED BY 'mostest';
  CREATE DATABASE mattermost_test CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
  GRANT ALL PRIVILEGES ON mattermost_test.* TO 'mmuser'@'%';
  FLUSH PRIVILEGES;
  EOF
  
  echo "[INFO] MySQL user and database created."
  
  # Update bind-address to allow remote access
  sed -i "s/^bind-address\s*=.*/#bind-address = 127.0.0.1/" /etc/mysql/mysql.conf.d/mysqld.cnf
  
  # Restart MySQL to apply changes
  systemctl restart mysql
  
  echo "[INFO] MySQL configuration complete and service restarted."


- Make the script executable:
  ```bash
  chmod +x setup-mysql.sh

- Run the script with sudo:
  ```bash
  sudo ./setup-mysql.sh

- Verify MySQL is running:
  ```bash
  sudo systemctl status mysql


### 7. Install and Configure Mattermost

- SSH into the Mattermost instance:
  ```bash
  ssh -i Mattermost.pem ubuntu@<public-IP>

- Update the system:
  ```bash
  sudo apt update && sudo apt upgrade -y

- Create the MySQL installation script on EC2:
  ```bash
  vim install_mattermost.sh

- Paste the Mattermost installation script on EC2:
  ```bash
  #!/bin/bash

  # Check if DB host argument is passed
  if [ -z "$1" ]; then
    echo "Usage: $0 <DB_HOST>"
    exit 1
  fi
    
  # Variables
  MM_VERSION=5.19.0
  MM_TAR="mattermost-${MM_VERSION}-linux-amd64.tar.gz"
  MM_DIR="/opt/mattermost"
  DB_HOST=$1
  
  # Download Mattermost
  wget "https://releases.mattermost.com/${MM_VERSION}/${MM_TAR}" || { echo "Download failed"; exit 1; }
  echo "Downloaded Mattermost"
  
  # Extract and install
  tar -xvzf "$MM_TAR" || { echo "Extraction failed"; exit 1; }
  echo "Extracted Mattermost"
  mv mattermost "$MM_DIR"
  mkdir -p "$MM_DIR/data"
  
  # Clean up tarball after extraction
  rm -f "$MM_TAR"
  echo "Cleaned up tarball"
  
  # Create system user
  id mattermost &>/dev/null || useradd --system --user-group mattermost
  echo "Created user"
  
  # Backup and update DB host in config.json
  CONFIG="$MM_DIR/config/config.json"
  cp "$CONFIG" "$CONFIG.bak"
  
  # Replace DB host (assuming default connection string format)
  sed -i "s/localhost:3306/${DB_HOST}:3306/" "$CONFIG"
  echo "Updated config.json with DB host: $DB_HOST"
  
  # Optional: set correct permissions for the entire directory
  chown -R mattermost:mattermost "$MM_DIR"
  chmod -R 775 "$MM_DIR/data" # Ensure that data directory is writable by mattermost user
  echo "Set correct permissions"
  
  # (Optional) Start Mattermost after installation
  # sudo -u mattermost ./bin/mattermost &
  
  echo "Mattermost setup completed."
  
- Make the script executable:
  ```bash
  chmod +x install_mattermost.sh

- ### Run the script with your MySQL host
  Replace your-db-host with the actual hostname or IP address of your MySQL server:
  ```bash
  ./install_mattermost.sh your-db-host

- Example:
  ```bash
  ./install_mattermost.sh 192.168.1.100

- ### What to Expect
  - It will download and extract Mattermost.
  - Move it to /opt/mattermost.
  - Create a mattermost system user.
  - Update config.json to point to the MySQL host you specified.
  - Set correct permissions.


### 8. Access Mattermost Web Application
- Open a web browser and navigate to:

  ```cpp
  http://<Mattermost-public-IP>:8065

- Complete the initial setup through the web interface.

### Conclusion
By following this guide, you've successfully deployed a self-hosted Mattermost application with a MySQL backend on AWS EC2, utilizing a secure VPC architecture with public and private subnets.
