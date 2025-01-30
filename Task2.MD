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

