# Quick Start Guide

This guide will walk you through setting up your self-hosted GitHub Copilot Coding Agent from scratch.

## 📋 Prerequisites Checklist

Before you begin, make sure you have:

- [ ] Docker installed and running
- [ ] Minikube installed
- [ ] Helm installed
- [ ] kubectl installed
- [ ] GitHub Personal Access Token created
- [ ] At least 8 CPU cores and 16GB RAM available
- [ ] 100GB free disk space

## 🚀 Step-by-Step Setup

### Step 1: Create GitHub Personal Access Token

1. Go to https://github.com/settings/tokens
2. Click **"Generate new token"** → **"Generate new token (classic)"**
3. Set an expiration date (recommend 90 days for security)
4. Select the following scopes:
   - ✅ `repo` (Full control of private repositories)
   - ✅ `workflow` (Update GitHub Action workflows)
   - ✅ `admin:org` (if setting up organization-level runners)
5. Click **"Generate token"**
6. **IMPORTANT**: Copy the token immediately - you won't be able to see it again!
7. Store it securely (you'll need it in Step 4)

### Step 2: Clone the Repository

```bash
git clone https://github.com/kennethtang4/copilot-agent-self-hosted-sample.git
cd copilot-agent-self-hosted-sample
```

### Step 3: Start Minikube

Start Minikube with appropriate resources:

```bash
# For basic setup (minimum requirements)
minikube start --cpus=8 --memory=16g --disk-size=100g --driver=docker

# For GPU-enabled setup (recommended for ML features)
minikube start \
  --cpus=32 \
  --memory=64g \
  --disk-size=500g \
  --driver=docker \
  --container-runtime=containerd

# Enable GPU support (if you have NVIDIA GPU)
minikube addons enable nvidia-device-plugin
```

Verify Minikube is running:
```bash
minikube status
```

Expected output:
```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

### Step 4: Configure Values

Edit the configuration file with your specific values:

```bash
cd copilot-agent-sample-runner
nano values.yaml  # or use your preferred editor (vim, code, etc.)
```

**Required changes in `values.yaml`:**

1. **Update GitHub URL** (line 1):
   ```yaml
   # For a repository:
   githubConfigUrl: "https://github.com/YOUR-USERNAME/YOUR-REPO"
   
   # OR for an organization (recommended for multiple repos):
   githubConfigUrl: "https://github.com/YOUR-ORG-NAME"
   ```
   
   ⚠️ **Important**: Do NOT include `.git` at the end of the URL

2. **Add your GitHub token** (line 3):
   ```yaml
   githubConfigSecret:
     github_token: "ghp_your_actual_token_here"
   ```
   
   Replace `GITHUB_PERSONAL_TOKEN` with the token you created in Step 1

3. **Optional adjustments**:
   ```yaml
   # Adjust runner scaling (lines 4-5)
   maxRunners: 10  # Maximum number of runners
   minRunners: 0   # Minimum idle runners
   
   # Adjust storage (lines 14, 43)
   storage: 1Gi    # Increase if needed (e.g., 5Gi, 10Gi)
   ```

Save and close the file.

### Step 5: Build the Runner Image

Build the custom Docker image with all required tools:

```bash
# Make sure you're in copilot-agent-sample-runner directory
./build.sh
```

This will take 10-20 minutes as it:
- Downloads the GitHub Actions runner base image
- Installs Node.js, Python, and development tools
- Downloads and installs CodeQL (v2.23.2)
- Installs PyTorch and AI/ML libraries
- Installs VS Code CLI

Monitor the build for any errors. Expected final output:
```
Successfully built [image-id]
Successfully tagged copilot-agent-sample-runner:latest
```

### Step 6: Install the Runner

Run the main installation script:

```bash
cd ..  # Return to root directory
./install.sh
```

This script will:
1. Load the Docker image into Minikube
2. Install the Actions Runner Controller
3. Deploy your runner scale set

Expected output:
```
...
NAME: arc
...
STATUS: deployed
...
NAME: arc-runner-set-copilot-coding-agent
...
STATUS: deployed
```

### Step 7: Verify Installation

Check that everything is running:

```bash
# Check ARC controller
kubectl get pods -n arc-systems

# Check runner pods
kubectl get pods -n arc-runners

# Check runner scale set
helm list -n arc-runners
```

Expected output for runners:
```
NAME                                                            READY   STATUS    RESTARTS   AGE
arc-runner-set-copilot-coding-agent-listener-xxxxx-xxxx        1/1     Running   0          1m
```

### Step 8: Test Your Setup

1. **Create a test workflow** in your repository:

   Create `.github/workflows/test-runner.yml`:
   ```yaml
   name: Test Self-Hosted Runner
   
   on:
     workflow_dispatch:
   
   jobs:
     test:
       runs-on: arc-runner-set-copilot-coding-agent
       steps:
         - name: Check runner environment
           run: |
             echo "Runner is working!"
             echo "Node version: $(node --version)"
             echo "Python version: $(python --version)"
             echo "CodeQL version: $(codeql --version)"
             
         - name: Checkout code
           uses: actions/checkout@v4
         
         - name: Run a simple test
           run: echo "Self-hosted runner is operational!"
   ```

2. **Run the workflow**:
   - Go to your repository on GitHub
   - Click **"Actions"** tab
   - Select **"Test Self-Hosted Runner"**
   - Click **"Run workflow"**
   - Click **"Run workflow"** button

3. **Watch it run**:
   - The workflow should start within 30 seconds
   - A runner pod will be created automatically
   - After completion, the runner pod will be cleaned up

### Step 9: Set Up CodeQL Workflow (Optional)

If you want to use the example CodeQL workflow:

1. Copy the example workflow to your repository:
   ```bash
   mkdir -p .github/workflows
   cp copilot-setup-steps.yml .github/workflows/
   ```

2. Edit `.github/workflows/copilot-setup-steps.yml`:
   - Line 26: Replace `REPO_NAME` with your repository name
   ```yaml
   mkdir -p /home/runner/_work/YOUR-REPO-NAME/.codeql-scratch
   ln -s /opt/codeql /home/runner/_work/YOUR-REPO-NAME/.codeql-scratch
   ```

3. Commit and push:
   ```bash
   git add .github/workflows/copilot-setup-steps.yml
   git commit -m "Add CodeQL workflow for self-hosted runner"
   git push
   ```

## ✅ Success Indicators

You'll know your setup is working when:

- ✅ Minikube is running without errors
- ✅ All pods are in "Running" state
- ✅ Helm installations show "deployed" status
- ✅ Your runner appears in GitHub Settings → Actions → Runners
- ✅ Test workflow completes successfully

## 🔄 Making Changes

### Update Runner Configuration

If you need to change settings in `values.yaml`:

```bash
cd copilot-agent-sample-runner
# Edit values.yaml
nano values.yaml

# Apply changes
./upgrade.sh
```

### Rebuild Runner Image

If you modify the `Dockerfile`:

```bash
cd copilot-agent-sample-runner
./build.sh
minikube image load copilot-agent-sample-runner:latest
./upgrade.sh
```

### Complete Reset

If something goes wrong and you want to start fresh:

```bash
./reset.sh
```

This will nuke and reinstall everything.

## 🐛 Common Issues and Solutions

### Issue: "Minikube start" fails with resource error

**Solution**: Reduce resources or free up system resources
```bash
# Try with minimum specs
minikube start --cpus=4 --memory=8g --disk-size=50g --driver=docker
```

### Issue: Pods stuck in "Pending" state

**Solution**: Check resource availability
```bash
kubectl describe pod <pod-name> -n arc-runners
# Look for resource constraint messages
```

### Issue: Runner not appearing in GitHub

**Solution**: Check your token and URL
```bash
# View runner logs
kubectl logs -n arc-runners -l app.kubernetes.io/component=runner-scale-set-listener

# Common issues:
# - Token expired or insufficient permissions
# - Wrong URL format (includes .git or incorrect)
# - Network connectivity issues
```

### Issue: "ImagePullBackOff" error

**Solution**: Ensure image is loaded in Minikube
```bash
minikube image ls | grep copilot-agent-sample-runner
# If not listed:
cd copilot-agent-sample-runner
./build.sh
minikube image load copilot-agent-sample-runner:latest
./upgrade.sh
```

### Issue: Docker build fails

**Solution**: Check Docker is running and has internet access
```bash
docker ps  # Should show running containers
docker pull ubuntu:latest  # Test internet connectivity
```

### Issue: Out of disk space

**Solution**: Clean up Docker and Minikube
```bash
# Clean Docker
docker system prune -a

# Clean Minikube
minikube delete
# Then start fresh
```

## 🎯 Next Steps

Now that your runner is set up:

1. **Configure GitHub Copilot**: Ensure your organization has GitHub Copilot enabled
2. **Create workflows**: Start building workflows that use your self-hosted runner
3. **Monitor resources**: Keep an eye on CPU, memory, and disk usage
4. **Scale as needed**: Adjust `maxRunners` and `minRunners` based on demand
5. **Security**: Regularly update the runner image and rotate tokens

## 📚 Additional Information

- See [README.md](README.md) for architecture details and advanced configuration
- Check GitHub Actions documentation for workflow syntax
- Review Kubernetes documentation for pod management

## 🆘 Getting Help

If you encounter issues not covered here:

1. Check pod logs: `kubectl logs <pod-name> -n arc-runners`
2. Check Minikube logs: `minikube logs`
3. Review Helm deployment: `helm status arc-runner-set-copilot-coding-agent -n arc-runners`
4. Check GitHub Actions Runner Controller logs
5. Consult the [GitHub Actions Runner Controller documentation](https://github.com/actions/actions-runner-controller)

## 🎉 Congratulations!

You now have a fully functional self-hosted GitHub Copilot Coding Agent! Your workflows can leverage custom compute resources, pre-installed tools, and enhanced security.

---

**Tip**: Bookmark this guide for future reference when setting up additional runners or troubleshooting issues.
