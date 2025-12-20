## Summary
DevOps project delivering a Task Manager web app with CRUD, MariaDB storage, automated tests, CI, local VM provisioning (Vagrant + Ansible), Docker images, and Kubernetes manifests for Minikube. No public cloud deployment is required.

## Repository Setup
Use this repository’s remote URL as the main repository URL.

Clone with submodules:
```bash
git clone --recurse-submodules <this-repository-URL>
# To get the URL: git remote get-url origin
```

If already cloned:
```bash
git submodule update --init --recursive
```

## Apps as submodules
The app components are **Git submodules**. List them:
```bash
git submodule status
```