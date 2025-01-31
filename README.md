# Task 1: Kubernetes and Python

## **Part 1: Python Script for Kubernetes Pod Monitoring**

### **Objective:**

This task involves creating a Python script that connects to a Kubernetes cluster, lists all running pods in a specific namespace, and checks their status. If a pod is not in the "Running" state, the script fetches the logs of that pod and saves them to a file.

### **Python Script:**

```python
from kubernetes import client, config
import os

# Load Kubernetes configuration (from local kubeconfig)
config.load_kube_config()

# Define the namespace to monitor
namespace = "default"

def list_pods_and_check_status():
    # Connect to the Kubernetes API
    v1 = client.CoreV1Api()

    print(f"Fetching pod information from namespace: {namespace}\n")
    
    # List pods in the specified namespace
    pods = v1.list_namespaced_pod(namespace=namespace)

    for pod in pods.items:
        pod_name = pod.metadata.name
        pod_status = pod.status.phase

        print(f"Pod Name: {pod_name}, Status: {pod_status}")
        
        # Check if the pod is not in the "Running" state
        if pod_status != "Running":
            print(f"\nFetching logs for pod {pod_name} (Status: {pod_status})")
            try:
                # Fetch logs of the pod
                logs = v1.read_namespaced_pod_log(name=pod_name, namespace=namespace)
                
                # Save logs to a file
                log_filename = f"{pod_name}-logs.txt"
                with open(log_filename, "w") as log_file:
                    log_file.write(logs)
                print(f"Logs saved to {log_filename}\n")
            except Exception as e:
                print(f"Error fetching logs for pod {pod_name}: {e}\n")

if __name__ == "__main__":
    list_pods_and_check_status()
```

### **Instructions to Run the Script:**

1. Install the required Kubernetes Python client:
   ```bash
   pip install kubernetes
   ```
2. Ensure that your local Kubernetes configuration (`~/.kube/config`) is set up to access the desired cluster.
3. Execute the script:
   ```bash
   python k8s_pod_monitor.py
   ```

---

## **Part 2: Deploying a Simple Flask Application on Kubernetes**

### **Objective:**

Deploy a simple Flask application on the Kubernetes cluster and verify its status using the Python script.

### **Step 1: Create a Flask Application**

Create a file named `app.py`:

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello_world():
    return "Hello, World!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### **Step 2: Create a Docker Image**

1. Create a `Dockerfile`:

   ```Dockerfile
   FROM python:3.8-slim

   WORKDIR /app

   COPY requirements.txt requirements.txt
   RUN pip install -r requirements.txt

   COPY . .

   CMD ["python", "app.py"]
   ```

2. Create a `requirements.txt` file:

   ```
   Flask
   ```

3. Build and tag the Docker image:

   ```bash
   docker build -t flask-app:latest .
   ```

4. Push the image to a container registry (if needed).

### **Step 3: Kubernetes YAML Deployment**

Create a file named `flask-app-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-app
        image: flask-app:latest
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: flask-app-service
spec:
  type: NodePort
  selector:
    app: flask-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
    nodePort: 30007
```

### **Step 4: Deploy to Kubernetes**

1. Apply the YAML file:
   ```bash
   kubectl apply -f flask-app-deployment.yaml
   ```
2. Verify the pod status:
   ```bash
   kubectl get pods
   ```

### **Step 5: Verify with the Python Script**

Run the Python script to check the status of the deployed Flask app pod:

```bash
python k8s_pod_monitor.py
```

If the pod is not running, the script will fetch and save the logs.

### **Step 6: Access the Application**

Access the Flask app in your browser at:

```
http://<node-ip>:30007
```

Replace `<node-ip>` with the IP of your Kubernetes node.









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

# Part 2: Linux Shell Script

## Objective
Create a shell script to:
- Automatically set up a local Kubernetes cluster using Minikube.
- Install kubectl and configure it to connect to the cluster.
- Deploy the Flask app using the YAML file from Task 1.

```bash
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

# Step 8: Deploy the Flask App Using YAML
echo "Deploying Flask app..."
kubectl apply -f flask_app.yaml

echo "Minikube setup complete!"
```

## Instructions
1. Save the script as `setup_minikube.sh`.
2. Ensure the Flask app YAML configuration file (`flask_app.yaml`) is available in the same directory.
3. Run the script with:
   ```bash
   chmod +x setup_minikube.sh
   ./setup_minikube.sh
   ```

## Example Output
```bash
Minikube not installed. Installing Minikube...
Docker already installed.
Adding user to the Docker group...
kubectl already installed.
Starting Minikube with Docker as the driver...
Setting kubectl to use Minikube context...
Verifying Kubernetes cluster...
Deploying Flask app...
Minikube setup complete!
```

### Troubleshooting
- **Permission Issues with Docker:**
  Run the following command and log in again:
  ```bash
  sudo usermod -aG docker $USER
  newgrp docker
  ```
- **Minikube Startup Errors:**
  Check Docker installation and ensure it's running.
- **kubectl Not Found:**
  Verify the PATH variable includes `/usr/local/bin/`.

### Additional Notes
- This script uses the `set -e` flag to exit on any errors.
- Make sure your system meets the [Minikube system requirements](https://minikube.sigs.k8s.io/docs/start/).

1. Infrastructure Setup:
Cloud Platform: AWS
Service Used: Elastic Kubernetes Service (EKS)
Provisioning Tool: Terraform
App Deployment: Flask app containerized with Docker
Steps:

I used Terraform to provision the AWS infrastructure, including the EKS cluster.
Configured kubectl to connect to the cluster and deploy the Flask app using Kubernetes.
Exposed the Flask app through a LoadBalancer service, ensuring public accessibility via a public endpoint.
2. Monitoring Setup:
Tool Used: AWS CloudWatch
Steps:

Set up CloudWatch monitoring to track key metrics (CPU, memory usage) for the EKS cluster.
Created a custom CloudWatch dashboard to monitor the health of the Flask app, CPU, and memory usage.
Integrated the monitoring with the Flask app to ensure smooth operations and visibility.
CloudWatch Dashboard:
Click here to view the CloudWatch dashboard

3. Outcome:
The Flask app is successfully deployed on AWS EKS and accessible via a public endpoint.
Monitoring is set up using CloudWatch, ensuring that the app’s health and resource usage are continuously tracked.
Live App:
Click here to access the live Flask app

Technologies Used:
AWS EKS
Terraform
Docker
Kubernetes
AWS CloudWatch
You can also find the source code and Terraform configuration files in the GitHub repository:
GitHub Repository
















