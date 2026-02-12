# Jenkins Job Setup Guide

## Prerequisites

Before creating the Jenkins job, ensure you have:

- [ ] Jenkins installed and running
- [ ] Docker installed on Jenkins server
- [ ] Python 3.9 installed on Jenkins server
- [ ] Xvfb installed for E2E tests: `sudo apt-get install xvfb firefox geckodriver`
- [ ] GitHub repository URL
- [ ] DockerHub account (for pushing images)

## Step 1: Install Required Jenkins Plugins

Navigate to: **Jenkins Dashboard → Manage Jenkins → Manage Plugins → Available**

Install these plugins:
- [x] **GitHub Integration Plugin** - For webhook triggers
- [x] **Pipeline** - For Jenkinsfile support
- [x] **Docker Pipeline** - For Docker commands
- [x] **Email Extension Plugin** - For email notifications
- [x] **HTML Publisher Plugin** - For test reports
- [x] **JUnit Plugin** - For test results
- [x] **Cobertura Plugin** or **Coverage Plugin** - For code coverage
- [x] **Credentials Binding Plugin** - For secure credential management

After installation, restart Jenkins:
```bash
sudo systemctl restart jenkins
```

## Step 2: Configure Jenkins Credentials

Navigate to: **Jenkins Dashboard → Manage Jenkins → Manage Credentials → (global) → Add Credentials**

### 2.1 DockerHub Credentials
- **Kind**: Username with password
- **Scope**: Global
- **Username**: `your-dockerhub-username`
- **Password**: `your-dockerhub-password`
- **ID**: `DOCKER_HUB_CREDENTIALS`
- **Description**: DockerHub Login

### 2.2 GitHub Credentials (for version increment pushback)
- **Kind**: Username with password
- **Scope**: Global
- **Username**: `your-github-username`
- **Password**: `your-github-personal-access-token`
  - Create token at: https://github.com/settings/tokens
  - Required scopes: `repo` (full control)
- **ID**: `GITHUB_CREDENTIALS`
- **Description**: GitHub Access Token

### 2.3 JIRA Credentials (Optional)
- **Kind**: Username with password
- **Scope**: Global
- **Username**: `your-jira-email`
- **Password**: `your-jira-api-token`
  - Create token at: https://id.atlassian.com/manage-profile/security/api-tokens
- **ID**: `JIRA_CREDENTIALS`
- **Description**: JIRA API Access

## Step 3: Configure Email Notifications

Navigate to: **Manage Jenkins → Configure System → Extended E-mail Notification**

### SMTP Configuration
- **SMTP server**: `smtp.gmail.com` (or your mail server)
- **SMTP Port**: `465` (SSL) or `587` (TLS)
- **Credentials**: Add new credentials
  - Username: your-email@gmail.com
  - Password: App-specific password (for Gmail)
- **Use SSL**: ✓ (check)
- **Default Content Type**: HTML (text/html)
- **Default Recipients**: your-email@example.com

### Test Email
Click "Test configuration by sending test e-mail" to verify setup.

## Step 4: Create Jenkins Pipeline Job

### 4.1 Create New Job
1. **Jenkins Dashboard → New Item**
2. **Enter job name**: `devops-ci-cd-pipeline`
3. **Select**: Pipeline
4. **Click**: OK

### 4.2 General Configuration

**Description**:
```
DevOps CI/CD Pipeline for testing, building, and deploying the application
Triggers on merge to main branch
```

**GitHub project** (check):
- **Project URL**: `https://github.com/your-username/Devops-ci-cd-exercise/`

**Build Triggers**:
- [x] **GitHub hook trigger for GITScm polling**

**Advanced Project Options**:
- **Display Name**: `DevOps CI/CD Pipeline`

### 4.3 Pipeline Configuration

**Definition**: Pipeline script from SCM

**SCM**: Git
- **Repository URL**: `https://github.com/your-username/Devops-ci-cd-exercise.git`
- **Credentials**: Select your GitHub credentials
- **Branch Specifier**: `*/main`

**Script Path**: `jenkins/Jenkinsfile`

**Lightweight checkout**: ✓ (uncheck for initial setup)

### 4.4 Environment Variables

Scroll to **Pipeline** section, then add these in your Jenkinsfile environment block or configure in:
**Jenkins Dashboard → Job → Configure → Pipeline → Environment variables**

Add:
- `DOCKER_REGISTRY` = `your-dockerhub-username/devops-testing-app`
- `GIT_REPO` = `Devops-ci-cd-exercise`
- `JIRA_URL` = `https://your-company.atlassian.net` (optional)
- `JIRA_PROJECT_KEY` = `DEV` (optional)

## Step 5: Configure GitHub Webhook

### 5.1 In GitHub Repository
1. Go to: **Your Repo → Settings → Webhooks → Add webhook**

2. Configure:
   - **Payload URL**: `http://your-jenkins-url:8080/github-webhook/`
     - Replace `your-jenkins-url` with your Jenkins server IP or domain
     - If using ngrok for local testing: `https://xxxxx.ngrok.io/github-webhook/`
   
   - **Content type**: `application/json`
   
   - **Secret**: (leave empty or add matching secret in Jenkins)
   
   - **SSL verification**: Enable SSL verification (if using HTTPS)
   
   - **Which events would you like to trigger this webhook?**
     - Select: **Just the push event**
   
   - **Active**: ✓ (check)

3. **Click**: Add webhook

4. **Verify**: You should see a green checkmark after first delivery

### 5.2 Test Webhook
Make a small commit to main branch:
```bash
echo "test" >> README.md
git add README.md
git commit -m "test: trigger pipeline"
git push origin main
```

Check Jenkins - build should start automatically.

## Step 6: Initial Build & Verification

### 6.1 Manual Trigger (First Time)
1. **Jenkins Dashboard → Your Job → Build Now**
2. Monitor the build in **Console Output**

### 6.2 Expected Results

**Successful Build Shows**:
- ✅ Setup Environment
- ✅ Lint Code
- ✅ Unit Tests (with coverage report)
- ✅ Integration Tests
- ✅ End-to-End Tests
- ✅ Performance Tests (if ENVIRONMENT=production)
- ✅ Security Scan
- ✅ Build Docker Image
- ✅ Email sent

**View Reports**:
- **Test Results**: Job → Test Results
- **Coverage Report**: Job → Unit Test Coverage Report
- **Performance Report**: Job → Performance Test Report
- **Artifacts**: Job → Last Successful Artifacts

### 6.3 Test Failure Scenario
Introduce a failing test:
```python
# In tests/unit/test_utils.py
def test_failure():
    assert False, "Intentional failure"
```

Commit and push - verify:
- ❌ Build fails at Unit Tests stage
- 📧 Email received with failure details
- 🎫 JIRA ticket created (if configured)

## Step 7: Advanced Configuration (Optional)

### 7.1 Blue Ocean UI
For better visualization:
```bash
# Install Blue Ocean plugin
Jenkins Dashboard → Manage Plugins → Available → Blue Ocean
```

Access at: `http://your-jenkins-url/blue`

### 7.2 Build Retention
**Job → Configure → General → Discard old builds**
- Days to keep builds: 30
- Max # of builds to keep: 50

### 7.3 Concurrent Builds
**Job → Configure → General**
- [ ] Do not allow concurrent builds (recommended for deployment jobs)

### 7.4 Build Parameters
Add build parameters for manual control:
**Job → Configure → This project is parameterized**

Add Boolean Parameter:
- **Name**: `SKIP_TESTS`
- **Default**: false
- **Description**: Skip all tests (emergency deployments only)

## Troubleshooting

### Issue: Webhook not triggering
**Solution**:
- Check GitHub webhook delivery status
- Verify Jenkins is accessible from internet
- Check firewall rules
- Test with: `curl -X POST http://your-jenkins-url/github-webhook/`

### Issue: Docker permission denied
**Solution**:
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### Issue: E2E tests failing
**Solution**:
```bash
# Install required dependencies
sudo apt-get install -y xvfb firefox-geckodriver
```

### Issue: Email not sending
**Solution**:
- Verify SMTP configuration
- Check Extended Email Publisher plugin settings
- For Gmail: Enable "Less secure app access" or use App Password
- Test with Jenkins system configuration

### Issue: Python venv fails
**Solution**:
```bash
sudo apt-get install python3.9 python3.9-venv python3-pip
```

## Quick Reference Commands

### Check Jenkins Logs
```bash
sudo tail -f /var/log/jenkins/jenkins.log
```

### Restart Jenkins
```bash
sudo systemctl restart jenkins
```

### Check Docker Status
```bash
docker ps
docker images | grep devops-testing-app
```

### Manual Version Increment
```bash
cd /path/to/repo
chmod +x scripts/increment-version.sh
./scripts/increment-version.sh
```

## Next Steps

1. ✅ Verify all tests pass
2. ✅ Check test reports are published
3. ✅ Verify Docker image is built
4. ✅ Test email notifications
5. ✅ Test GitHub webhook trigger
6. 📸 Take screenshots for documentation
7. 🚀 Merge to main and watch automation work!

## Screenshots to Capture (for README)

1. **Successful Build**:
   - Jenkins job overview with green checkmark
   - Blue Ocean pipeline visualization
   - Test results dashboard
   - Coverage report

2. **Failed Build**:
   - Jenkins job overview with red X
   - Console output showing failure
   - Email notification
   - JIRA ticket (if configured)

## Support

If you encounter issues:
1. Check Jenkins console output
2. Review GitHub webhook delivery logs
3. Verify all credentials are configured correctly
4. Check Jenkins system logs
5. Consult Jenkins documentation: https://www.jenkins.io/doc/
