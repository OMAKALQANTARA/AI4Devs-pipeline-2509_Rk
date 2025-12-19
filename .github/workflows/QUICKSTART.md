# Pipeline Quick Start Guide

## 🚀 What Changed

### ✅ Fixed Issues:
1. **Removed AWS Credentials Error** - No longer using `aws-actions/configure-aws-credentials` (not needed for SSH deployment)
2. **Backend Deploys First** - Backend is deployed before frontend
3. **Separate Deploy Paths** - Backend and frontend have independent deployment directories

### 📋 New Pipeline Flow:

```
1. Backend Tests (npm test)
          ↓
2. Backend Build (npm run build)
          ↓
3. Backend Deploy to EC2 ✨ NEW
          ↓
4. Frontend Build & Deploy to EC2
          ↓
5. Pipeline Summary
```

## ⚡ Quick Setup (5 minutes)

### Step 1: Configure GitHub Secrets

Go to your repository: `Settings > Secrets and variables > Actions > New repository secret`

Add these **3 required secrets**:

| Secret | Value | Where to find it |
|--------|-------|------------------|
| `EC2_INSTANCE` | `12.34.56.78` | Your EC2 public IP or DNS |
| `EC2_USER` | `ec2-user` | Use `ec2-user` for Amazon Linux, `ubuntu` for Ubuntu |
| `EC2_SSH_PRIVATE_KEY` | `-----BEGIN RSA...` | Full content of your `.pem` file |

**Optional secrets** (use defaults if not set):
- `BACKEND_DEPLOY_PATH` - Default: `/home/ec2-user/backend`
- `FRONTEND_DEPLOY_PATH` - Default: `/var/www/html`

### Step 2: Prepare Your EC2 Instance

Connect to your EC2:
```bash
ssh -i your-key.pem ec2-user@your-ec2-ip
```

Run this one-time setup:

#### For Amazon Linux 2:
```bash
# Install Node.js
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install 18 && nvm use 18 && nvm alias default 18

# Install nginx
sudo yum update -y
sudo yum install nginx -y

# Create directories
mkdir -p /home/ec2-user/backend
sudo mkdir -p /var/www/html
sudo chown -R ec2-user:ec2-user /var/www/html

# Start nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Install PM2 (for backend process management)
npm install -g pm2
```

#### For Ubuntu:
```bash
# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install nginx
sudo apt update
sudo apt install nginx -y

# Create directories
mkdir -p /home/ubuntu/backend
sudo mkdir -p /var/www/html
sudo chown -R ubuntu:ubuntu /var/www/html

# Start nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Install PM2
npm install -g pm2
```

### Step 3: Update EC2 Security Group

In AWS Console, ensure your EC2 security group allows:
- **Port 22** (SSH) - For deployment
- **Port 80** (HTTP) - For web traffic
- **Port 443** (HTTPS) - For secure web traffic (if using SSL)
- **Port 3000** (or your backend port) - For backend API

### Step 4: Push to Main Branch

```bash
git add .
git commit -m "Setup CI/CD pipeline"
git push origin main
```

Watch the pipeline run in: `Actions` tab in your GitHub repository

## 🎯 What Gets Deployed Where

| Component | Default Path | Port | Purpose |
|-----------|-------------|------|---------|
| **Backend** | `/home/ec2-user/backend` | 3000 | Node.js API server |
| **Frontend** | `/var/www/html` | 80 | React static files served by nginx |

## 🔧 Optional: Enable Auto-Restart

### Backend (using PM2):

On your EC2 instance:
```bash
cd /home/ec2-user/backend
pm2 start dist/index.js --name backend
pm2 save
pm2 startup
```

Then uncomment this line in `.github/workflows/pipeline.yml` (line ~163):
```yaml
pm2 restart backend || pm2 start dist/index.js --name backend
```

### Frontend (restart nginx):

On your EC2 instance:
```bash
sudo visudo
# Add this line (adjust username):
ec2-user ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx
```

Then uncomment this line in `.github/workflows/pipeline.yml` (line ~224):
```yaml
ssh -i deploy_key.pem -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST "sudo systemctl restart nginx"
```

## 📊 Monitoring Your Deployment

After pushing to main:
1. Go to **Actions** tab in GitHub
2. Click on the latest workflow run
3. Watch each job complete:
   - ✅ Backend Tests
   - ✅ Backend Build
   - ✅ Backend Deploy
   - ✅ Frontend Deploy
   - ✅ Pipeline Summary

## 🐛 Troubleshooting

### "Permission denied (publickey)"
- Check `EC2_SSH_PRIVATE_KEY` secret includes BEGIN and END lines
- Verify `EC2_USER` matches your instance type
- Ensure your EC2 key pair matches the secret

### "Connection refused"
- Check EC2 security group allows SSH from 0.0.0.0/0
- Verify EC2 instance is running
- Confirm `EC2_INSTANCE` IP/DNS is correct

### Backend not starting
- SSH to EC2 and check: `cd ~/backend && npm start`
- Check Node.js is installed: `node --version`
- Check logs: `pm2 logs backend`

### Frontend not loading
- Check nginx status: `sudo systemctl status nginx`
- Verify files deployed: `ls -la /var/www/html`
- Check nginx config: `sudo nginx -t`

## 🎉 Success!

Your application should now be accessible at:
- **Frontend**: `http://your-ec2-ip`
- **Backend API**: `http://your-ec2-ip:3000` (or your configured port)

For detailed setup instructions, see: [PIPELINE_SETUP.md](./PIPELINE_SETUP.md)

