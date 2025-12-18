# GitHub Actions Pipeline Setup Guide

This document explains how to configure the CI/CD pipeline for deploying to AWS EC2.

## Pipeline Overview

The pipeline consists of three main jobs:

1. **Backend Tests** - Runs `npm test` in the backend directory
2. **Backend Build** - Runs `npm run build` in the backend directory
3. **Frontend Deploy** - Builds the React app and deploys it to AWS EC2

## Required GitHub Secrets

To enable the pipeline, you need to configure the following secrets in your GitHub repository:

### Navigate to: `Settings > Secrets and variables > Actions > New repository secret`

### Required Secrets:

| Secret Name | Description | Example Value |
|-------------|-------------|---------------|
| `AWS_ACCESS_KEY_ID` | AWS IAM user access key ID | `AKIAIOSFODNN7EXAMPLE` |
| `AWS_ACCESS_KEY` | AWS IAM user secret access key | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| `AWS_REGION` | AWS region where your EC2 instance is located | `us-east-1` |
| `EC2_INSTANCE` | Public IP or DNS of your EC2 instance | `ec2-XX-XXX-XXX-XX.compute-1.amazonaws.com` |
| `EC2_USER` | SSH username for EC2 instance | `ubuntu` or `ec2-user` |
| `EC2_SSH_PRIVATE_KEY` | Private SSH key for EC2 access | Content of your `.pem` file |
| `DEPLOY_PATH` | (Optional) Path where to deploy on EC2 | `/var/www/html` (default) |

## Detailed Setup Instructions

### 1. AWS Credentials Setup

Create an IAM user with programmatic access:
1. Go to AWS Console > IAM > Users > Add User
2. Enable "Programmatic access"
3. Attach policies: `AmazonEC2FullAccess` (or create custom policy with minimal permissions)
4. Save the Access Key ID and Secret Access Key

### 2. EC2 Instance Setup

Ensure your EC2 instance:
- Has a public IP or Elastic IP
- Security group allows SSH (port 22) from GitHub Actions IPs
- Has web server installed (nginx/apache) if serving static files
- SSH key pair is available

### 3. SSH Key Configuration

```bash
# Your EC2 SSH private key should look like:
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA...
...
-----END RSA PRIVATE KEY-----
```

Copy the **entire content** of your `.pem` file (including BEGIN and END lines) into the `EC2_SSH_PRIVATE_KEY` secret.

### 4. EC2 Server Preparation

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

### 5. Test the Pipeline

After configuring all secrets:
1. Push changes to `main` or `develop` branch
2. Go to `Actions` tab in your GitHub repository
3. Watch the pipeline execution
4. Check the summary for detailed results

## Pipeline Triggers

The pipeline runs on:
- **Push** to `main` or `develop` branches
- **Pull Request** to `main` or `develop` branches

**Note:** Deployment to EC2 only happens on pushes to the `main` branch.

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

4. **AWS Credentials Invalid**
   - Verify IAM user has necessary permissions
   - Check if `AWS_ACCESS_KEY_ID` and `AWS_ACCESS_KEY` are correct
   - Ensure credentials are active and not expired

5. **Package Manager Not Found (apt/yum/dnf)**
   - Check your EC2 instance Linux distribution: `cat /etc/os-release`
   - Use the appropriate package manager for your distro (see section 4 above)
   - Ensure you're using the correct username for your distro

## Optional Enhancements

### Enable Nginx Restart (Uncomment in pipeline):
If you want to restart nginx after deployment, uncomment this line in the pipeline:
```yaml
# ssh -i deploy_key.pem -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST "sudo systemctl restart nginx"
```

And configure passwordless sudo for nginx:
```bash
# On EC2 instance
sudo visudo
# Add: ubuntu ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx
```

### Custom Deploy Path:
Set `DEPLOY_PATH` secret to deploy to a custom directory (defaults to `/var/www/html`)

## Security Best Practices

1. ✅ Use IAM roles with minimal required permissions
2. ✅ Rotate AWS access keys regularly
3. ✅ Keep SSH keys secure and never commit them to repository
4. ✅ Use separate AWS accounts/users for production and development
5. ✅ Enable MFA on AWS IAM users
6. ✅ Restrict EC2 security group SSH access to specific IP ranges

## Support

For issues or questions about the pipeline, please open an issue in the repository.

