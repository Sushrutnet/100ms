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

