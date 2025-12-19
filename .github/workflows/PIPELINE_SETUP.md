# GitHub Actions Pipeline Setup Guide

This document explains how to configure the CI/CD pipeline for deploying to AWS EC2.

## Pipeline Overview

The pipeline consists of five main jobs:

1. **Backend Tests** - Runs `npm test` in the backend directory
2. **Backend Build** - Runs `npm run build` in the backend directory
3. **Backend Deploy** - Deploys the backend to AWS EC2 (runs first)
4. **Frontend Build & Deploy** - Builds the React app and deploys it to AWS EC2 (runs after backend)
5. **Pipeline Summary** - Generates a summary of all job statuses

## Required GitHub Secrets

To enable the pipeline, you need to configure the following secrets in your GitHub repository:

### Navigate to: `Settings > Secrets and variables > Actions > New repository secret`

### Required Secrets:

| Secret Name | Description | Example Value |
|-------------|-------------|---------------|
| `EC2_INSTANCE` | Public IP or DNS of your EC2 instance | `ec2-XX-XXX-XXX-XX.compute-1.amazonaws.com` or `12.34.56.78` |
| `EC2_USER` | SSH username for EC2 instance | `ubuntu` or `ec2-user` |
| `EC2_SSH_PRIVATE_KEY` | Private SSH key for EC2 access | Content of your `.pem` file |
| `BACKEND_DEPLOY_PATH` | (Optional) Path where to deploy backend | `/home/ec2-user/backend` (default) |
| `FRONTEND_DEPLOY_PATH` | (Optional) Path where to deploy frontend | `/var/www/html` (default) |

## Detailed Setup Instructions

### 1. EC2 Instance Setup

Ensure your EC2 instance:
- Has a public IP or Elastic IP
- Security group allows SSH (port 22) from GitHub Actions IPs (0.0.0.0/0 for simplicity, or restrict to GitHub Actions IP ranges)
- Security group allows HTTP (port 80) and HTTPS (port 443) for web traffic
- Has web server installed (nginx/apache) for serving frontend
- Has Node.js installed for running backend
- SSH key pair is available

### 2. SSH Key Configuration

```bash
# Your EC2 SSH private key should look like:
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA...
...
-----END RSA PRIVATE KEY-----
```

Copy the **entire content** of your `.pem` file (including BEGIN and END lines) into the `EC2_SSH_PRIVATE_KEY` secret.

### 3. EC2 Server Preparation

On your EC2 instance, prepare the deployment directory. Choose the instructions based on your Linux distribution:

#### For Ubuntu / Debian:

```bash
# Connect to your EC2 instance
ssh -i your-key.pem ubuntu@your-ec2-host

# Install nginx
sudo apt update
sudo apt install nginx -y

# Create deployment directory
sudo mkdir -p /var/www/html
sudo chown -R ubuntu:ubuntu /var/www/html

# Configure nginx to serve from /var/www/html
sudo nano /etc/nginx/sites-available/default
# Update root directive to: root /var/www/html;

# Restart nginx
sudo systemctl restart nginx
sudo systemctl enable nginx
```

#### For Amazon Linux 2:

```bash
# Connect to your EC2 instance
ssh -i your-key.pem ec2-user@your-ec2-host

# Install nginx
sudo yum update -y
sudo yum install nginx1 -y

# Create deployment directory
sudo mkdir -p /var/www/html
sudo chown -R 755 /var/www/html

# Configure nginx to serve from /var/www/html
sudo nano /etc/nginx/nginx.conf
# Find the 'server' block and update root directive to: root /var/www/html;

# Start and enable nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Configure firewall to allow HTTP/HTTPS
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

#### For Amazon Linux 2023 / RHEL 8+ / CentOS Stream:

```bash
# Connect to your EC2 instance
ssh -i your-key.pem ec2-user@your-ec2-host

# Install nginx
sudo dnf update -y
sudo dnf install nginx -y

# Create deployment directory
sudo mkdir -p /var/www/html
sudo chown -R ec2-user:ec2-user /var/www/html

# Configure nginx to serve from /var/www/html
sudo nano /etc/nginx/nginx.conf
# Find the 'server' block and update root directive to: root /var/www/html;

# Start and enable nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Configure firewall to allow HTTP/HTTPS
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

**Note:** Remember to update your GitHub secrets with the correct `EC2_USER`:
- Ubuntu/Debian: use `ubuntu`
- Amazon Linux: use `ec2-user`
- RHEL: use `ec2-user`

**To check your Linux distribution:**
```bash
cat /etc/os-release
# or
uname -a
```

### 4. Install Node.js on EC2 (for Backend)

The backend requires Node.js to run. Install it on your EC2 instance:

#### For Amazon Linux 2:
```bash
# Install Node.js 18.x using nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install 18
nvm use 18
nvm alias default 18

# Verify installation
node --version
npm --version
```

#### For Ubuntu:
```bash
# Install Node.js 18.x
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify installation
node --version
npm --version
```

### 5. Optional: Setup Process Manager (PM2)

To keep your backend running continuously:

```bash
# Install PM2 globally
npm install -g pm2

# PM2 will be used to manage your backend application
# The pipeline can restart it automatically after deployment
```

### 6. Test the Pipeline

After configuring all secrets:
1. Push changes to `main` or `develop` branch
2. Go to `Actions` tab in your GitHub repository
3. Watch the pipeline execution
4. Check the summary for detailed results

## Pipeline Triggers

The pipeline runs on:
- **Push** to `main` or `develop` branches
- **Pull Request** to `main` or `develop` branches

**Note:** Deployment to EC2 (both backend and frontend) only happens on pushes to the `main` branch.

## Deployment Order

The pipeline deploys in the following order:
1. **Backend** is deployed first to `/home/ec2-user/backend` (or custom path)
2. **Frontend** is deployed second to `/var/www/html` (or custom path)

This ensures the API is available before the frontend that depends on it.

## Troubleshooting

### Common Issues:

1. **SSH Connection Failed**
   - Verify EC2 security group allows SSH from GitHub Actions IPs
   - Check if `EC2_INSTANCE` and `EC2_USER` secrets are correct
   - Ensure `EC2_SSH_PRIVATE_KEY` is complete with no extra spaces or line breaks
   - Verify the key format includes BEGIN and END lines

2. **Permission Denied on EC2**
   - Run: `sudo chown -R $USER:$USER /var/www/html` on EC2
   - Check file permissions: `ls -la /var/www/html`

3. **Build Artifacts Not Found**
   - Ensure `npm run build` completes successfully
   - Check `frontend/build` directory exists after build

4. **Package Manager Not Found (apt/yum/dnf)**
   - Check your EC2 instance Linux distribution: `cat /etc/os-release`
   - Use the appropriate package manager for your distro (see section 4 above)
   - Ensure you're using the correct username for your distro

## Optional Enhancements

### Enable Backend Auto-Restart with PM2:

If you want to automatically restart your backend after deployment, uncomment these lines in the pipeline (backend-deploy job):

```yaml
# pm2 restart backend || pm2 start dist/index.js --name backend
```

To set this up on your EC2 instance:
```bash
# Install PM2
npm install -g pm2

# Start your backend for the first time
cd /home/ec2-user/backend
pm2 start dist/index.js --name backend

# Save PM2 process list
pm2 save

# Setup PM2 to start on system boot
pm2 startup
# Follow the command it provides
```

### Enable Nginx Restart:

If you want to restart nginx after frontend deployment, uncomment this line in the pipeline (frontend-deploy job):

```yaml
# ssh -i deploy_key.pem -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST "sudo systemctl restart nginx"
```

Configure passwordless sudo for nginx:
```bash
# On EC2 instance
sudo visudo
# Add this line (replace ec2-user with your username):
ec2-user ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx
```

### Custom Deploy Paths:

- Set `BACKEND_DEPLOY_PATH` secret for backend deployment (default: `/home/ec2-user/backend`)
- Set `FRONTEND_DEPLOY_PATH` secret for frontend deployment (default: `/var/www/html`)

## Security Best Practices

1. ✅ Keep SSH keys secure and never commit them to repository
2. ✅ Rotate SSH keys regularly
3. ✅ Use separate EC2 instances for production and development
4. ✅ Restrict EC2 security group SSH access to GitHub Actions IP ranges when possible
5. ✅ Use environment-specific branches (main for production, develop for staging)
6. ✅ Enable EC2 instance monitoring and CloudWatch logs
7. ✅ Keep your EC2 instance and packages up to date
8. ✅ Use HTTPS/SSL certificates for production deployments

## Support

For issues or questions about the pipeline, please open an issue in the repository.

