# Self-Hosted GitHub Copilot Coding Agent

This repository contains example configuration files and scripts to set up a self-hosted GitHub Copilot Coding Agent using Kubernetes, Actions Runner Controller (ARC), and a custom runner image with pre-installed tools.

## 📋 Overview

The self-hosted GitHub Copilot Coding Agent allows you to run GitHub Copilot features on your own infrastructure, providing:

- **Control over compute resources**: Use your own hardware (including GPUs)
- **Custom environment**: Pre-configure tools, languages, and dependencies
- **Security**: Keep sensitive code within your infrastructure
- **Cost management**: Optimize resource usage for your specific needs

## 🏗️ Architecture

The setup consists of several components:

1. **Minikube Cluster**: Local Kubernetes cluster for testing and development
2. **Actions Runner Controller (ARC)**: Manages self-hosted GitHub Actions runners
3. **Custom Runner Image**: Docker container with:
   - GitHub Actions Runner base (required)
   - **CodeQL (v2.23.2)** - **REQUIRED** for GitHub Copilot Coding Agent functionality
   - Node.js 22.x with YAML tools (required)
   - Python 3 (required for basic functionality)
   - AI/ML libraries (PyTorch, Transformers, LangChain, OpenAI) - **optional**, only needed for ML/AI features
   - VS Code CLI - **optional**, for debugging
   - Various development tools (git, jq, yamllint, etc.) - required

4. **Kubernetes Resources**: Helm charts for deploying and managing runners

## 📁 Repository Structure

```
.
├── copilot-agent-sample-runner/
│   ├── Dockerfile                    # Custom runner image definition
│   ├── values.yaml                   # Helm chart configuration (REQUIRES EDITING)
│   ├── build.sh                      # Build Docker image
│   ├── install.sh                    # Install runner scale set
│   ├── upgrade.sh                    # Upgrade existing installation
│   └── install-codeql-bundle.sh      # CodeQL installation script
├── copilot-setup-steps.yml           # Example GitHub Actions workflow (REQUIRES EDITING)
├── install.sh                        # Main installation script
├── reset.sh                          # Reset and reinstall
└── nuke.sh                           # Complete teardown and fresh start
```

## 🔑 Prerequisites

Before getting started, ensure you have:

### Required Software
- [Minikube](https://minikube.sigs.k8s.io/docs/start/) (v1.30+)
- [Docker](https://docs.docker.com/get-docker/) (v20.10+)
- [Helm](https://helm.sh/docs/intro/install/) (v3.0+)
- [kubectl](https://kubernetes.io/docs/tasks/tools/) (v1.25+)

### System Requirements
- **CPU**: Minimum 4 cores (8+ cores recommended for better performance)
- **Memory**: Minimum 8GB (16GB+ recommended for better performance)
- **Disk**: Minimum 50GB free space (100GB+ recommended)
- **GPU** (optional): NVIDIA GPU with CUDA 13.0 support - only needed if using ML/AI features

### GitHub Requirements
- GitHub Personal Access Token (PAT) with the following scopes:
  - `repo` (Full control of private repositories)
  - `workflow` (Update GitHub Action workflows)
  - `admin:org` (if using organization-level runners)
- Repository or organization where runners will be registered
- GitHub Copilot subscription (Business or Enterprise)

## ⚙️ Configuration

### Required Variables to Set

Before installation, you **must** update the following configuration files:

#### 1. `copilot-agent-sample-runner/values.yaml`

Replace these placeholders:

```yaml
githubConfigUrl: "https://github.com/REPO_URL_WITHOUT_DOT_GIT"
# Replace with your repository or organization URL
# Examples:
#   - Repository: "https://github.com/myorg/myrepo"
#   - Organization: "https://github.com/myorg"

githubConfigSecret:
  github_token: "GITHUB_PERSONAL_TOKEN"
# Replace with your GitHub Personal Access Token
# Generate at: https://github.com/settings/tokens
```

#### 2. `copilot-setup-steps.yml`

This is an example workflow file that demonstrates a **workaround** for using the pre-installed CodeQL binary. This bootstrap step is necessary because the GitHub Copilot Coding Agent requires CodeQL to function properly, and this approach avoids downloading CodeQL on every workflow run.

To use it:

1. Copy it to your repository's `.github/workflows/` directory
2. Replace `REPO_NAME` in line 26 with your actual repository name
3. Update the `runs-on` label (line 14) if you change the runner installation name

```yaml
# Line 14: Update runner label if needed
runs-on: arc-runner-set-copilot-coding-agent

# Line 26-27: Replace REPO_NAME with your repository name
mkdir -p /home/runner/_work/REPO_NAME/.codeql-scratch
ln -s /opt/codeql /home/runner/_work/REPO_NAME/.codeql-scratch
```

## 🚀 Quick Start

See [QUICK_START.md](QUICK_START.md) for detailed step-by-step installation instructions.

## 🔧 Maintenance Scripts

### `install.sh`
Main installation script that:
- Loads the Docker image into Minikube
- Installs ARC controller
- Deploys the runner scale set

### `reset.sh`
Quick reset that:
- Runs `nuke.sh` to clean up
- Runs `install.sh` to reinstall

### `nuke.sh`
Complete cleanup that:
- Stops and deletes Minikube cluster
- Restarts with fresh configuration
- Re-enables GPU support (if configured)

### `copilot-agent-sample-runner/upgrade.sh`
Updates an existing runner installation with new configuration changes.

## 📦 Docker Image Details

The custom runner image (`copilot-agent-sample-runner`) includes:

### Base Tools (Required)
- GitHub Actions Runner (latest)
- Node.js 22.x with npm
- Python 3 with pip
- Git, curl, wget, jq, tree, vim
- yamllint

### Security Tools (Required)
- **CodeQL 2.23.2** (pre-installed at `/opt/codeql`) - **REQUIRED** for GitHub Copilot Coding Agent functionality

### AI/ML Tools (Optional)
- PyTorch with CUDA 13.0 support (optional - only needed for ML/AI features)
- Transformers, LangChain, OpenAI (optional - for AI/ML workflows)
- NumPy, Pandas, Matplotlib (optional - for data science workflows)
- Jupyter, DeepSpeed, Pytest (optional - for development and testing)

### Development Tools (Optional)
- VS Code CLI (optional - for debugging purposes)
- YAML parsing tools

## 🔒 Security Considerations

1. **Token Security**: Never commit your GitHub Personal Access Token to the repository
2. **Network Security**: Consider network policies for production deployments
3. **Resource Limits**: Set appropriate CPU/memory limits in `values.yaml`
4. **Image Security**: Regularly update the base images and dependencies
5. **CodeQL**: CodeQL is **required** for the GitHub Copilot Coding Agent to function properly and is pre-installed for security scanning

## 🐛 Troubleshooting

### Minikube won't start
```bash
# Check Docker is running
docker ps

# Check available resources
minikube config view

# Try deleting and recreating
minikube delete
minikube start --cpus=8 --memory=16g --disk-size=100g
```

### Runners not connecting to GitHub
1. Verify your GitHub token has correct permissions
2. Check the `githubConfigUrl` is correct (no `.git` suffix)
3. View runner logs:
```bash
kubectl logs -n arc-runners -l app.kubernetes.io/component=runner-scale-set-listener
```

### Docker image not found
```bash
# Rebuild and load the image
cd copilot-agent-sample-runner
./build.sh
minikube image load copilot-agent-sample-runner:latest
```

### Helm installation fails
```bash
# Check Helm charts are accessible
helm repo update

# Verify namespace exists
kubectl get namespace arc-systems
kubectl get namespace arc-runners

# Check existing installations
helm list -A
```

## 📚 Additional Resources

- [GitHub Actions Runner Controller Documentation](https://github.com/actions/actions-runner-controller)
- [Minikube Documentation](https://minikube.sigs.k8s.io/docs/)
- [GitHub Copilot Documentation](https://docs.github.com/en/copilot)
- [CodeQL Documentation](https://codeql.github.com/docs/)

## 🤝 Contributing

This is a sample configuration repository. Feel free to customize it for your specific needs:

- Modify the `Dockerfile` to add/remove tools
- Adjust resource limits in `values.yaml`
- Add additional workflows to `.github/workflows/`
- Update Minikube configuration in `nuke.sh`

## 📄 License

This project is provided as-is for educational and reference purposes. Please ensure compliance with GitHub's terms of service and licensing requirements for all included tools and dependencies.

## ⚠️ Important Notes

- This setup is designed for **development and testing**. For production use, consider:
  - Using a real Kubernetes cluster (GKE, EKS, AKS)
  - Implementing proper secrets management (e.g., Kubernetes Secrets, HashiCorp Vault)
  - Setting up monitoring and logging
  - Implementing backup and disaster recovery
  - Adding network security policies
  
- The GPU configuration in `nuke.sh` requires NVIDIA GPU support in your system and Docker
- Resource requirements are significant - ensure your system meets the minimum specifications
