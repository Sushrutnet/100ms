# Task 2: GitHub Actions and Linux

## **Part 1: GitHub Actions Workflow**

### **Objective:**
Create a GitHub Actions workflow that:
- Automatically builds and pushes the Docker image of the Flask app to Docker Hub whenever code is pushed to the main branch.
- Deploys the Flask app to the Kubernetes cluster using the YAML file from Task 1.

### **GitHub Actions Workflow File (`.github/workflows/deploy-flask-app.yml`)**

```yaml
name: Deploy Flask App to Minikube

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: self-hosted

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      # Install dependencies (Docker, Minikube, kubectl)
      - name: Set up dependencies
        run: |
          # Install Minikube
          if ! command -v minikube; then
            curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
            sudo mv minikube-linux-amd64 /usr/local/bin/minikube
            sudo chmod +x /usr/local/bin/minikube
          fi

          # Install Docker
          if ! command -v docker; then
            sudo apt-get update
            sudo apt-get install -y ca-certificates curl
            sudo apt-get install -y docker.io
            sudo systemctl start docker
          fi

          # Install kubectl
          if ! command -v kubectl; then
            curl -LO https://dl.k8s.io/release/v1.26.0/bin/linux/amd64/kubectl
            sudo mv kubectl /usr/local/bin/
            sudo chmod +x /usr/local/bin/kubectl
          fi

      # Ensure Docker is running
      - name: Ensure Docker is running
        run: |
          sudo systemctl start docker
          sudo systemctl enable docker

      # Start Minikube
      - name: Start Minikube
        run: |
          minikube start --driver=docker || minikube start --driver=docker

      # Update kubectl context to Minikube
      - name: Update kubectl context
        run: |
          minikube update-context

      # Set kubectl context explicitly
      - name: Ensure kubectl context is properly set
        run: |
          kubectl config use-context minikube

      # Apply Kubernetes Deployment for Flask App
      - name: Deploy Flask app to Kubernetes
        run: |
          kubectl apply -f flask-deployment.yaml --validate=false

      # Check deployment status
      - name: Wait for deployment to complete
        run: |
          kubectl rollout status deployment flask-app-deployment

      # Verify that the Flask app is running
      - name: Verify the Flask app is running
        run: |
          kubectl get pods
```

### **Key Features:**
- **Dependency Setup:** Installs Minikube, Docker, and kubectl if not already installed.
- **Cluster Management:** Starts Minikube and sets the kubectl context.
- **Deployment:** Applies the Kubernetes YAML file and waits for the deployment to complete.
- **Verification:** Checks the deployment status and verifies the running pods.

---

## **Part 2: Linux Shell Script**

### **Objective:**
Create a shell script to:
- Automatically set up a local Kubernetes cluster using Minikube.
- Install kubectl and configure it to connect to the cluster.
- Deploy the Flask app using the YAML file from Task 1.




#!/bin/bash

set -e  # Exit immediately if a command exits with a non-zero status

# Step 1: Install Minikube
if ! command -v minikube &>/dev/null; then
    echo "Minikube not installed. Installing Minikube..."
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    sudo mv minikube-linux-amd64 /usr/local/bin/minikube
    sudo chmod +x /usr/local/bin/minikube
else
    echo "Minikube already installed."
fi

# Step 2: Install Docker
if ! command -v docker &>/dev/null; then
    echo "Docker not installed. Installing Docker..."
    sudo apt-get update

    # Install necessary packages
    sudo apt-get install -y ca-certificates curl

    # Create the keyrings directory if it doesn't exist
    sudo install -m 0755 -d /etc/apt/keyrings

    # Add Docker's official GPG key
    if [ ! -f /etc/apt/keyrings/docker.asc ]; then
        sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
        sudo chmod a+r /etc/apt/keyrings/docker.asc
    else
        echo "Docker GPG key already exists; skipping key addition."
    fi

    # Add the Docker repository
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    # Update the package index
    sudo apt-get update

    # Install Docker CE
    sudo apt-get install -y docker-ce
else
    echo "Docker already installed."
fi

# Step 3: Add the current user to the Docker group to avoid permission issues
if ! groups "$USER" | grep -q '\bdocker\b'; then
    echo "Adding user to the Docker group..."
    sudo usermod -aG docker "$USER"
    newgrp docker || true  # Skip error if newgrp fails (not interactive)
else
    echo "User already in the Docker group."
fi

# Step 4: Install kubectl if not already installed
if ! command -v kubectl &>/dev/null; then
    echo "kubectl not found. Installing kubectl..."
    curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
    sudo mv kubectl /usr/local/bin/kubectl
    sudo chmod +x /usr/local/bin/kubectl
else
    echo "kubectl already installed."
fi

# Step 5: Start Minikube
echo "Starting Minikube with Docker as the driver..."
minikube start --driver=docker

# Step 6: Set kubectl to use the Minikube context
echo "Setting kubectl to use Minikube context..."
kubectl config use-context minikube

# Step 7: Verify the Kubernetes cluster status
echo "Verifying Kubernetes cluster..."
kubectl get all

echo "Minikube setup complete!"

