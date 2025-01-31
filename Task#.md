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
3. Outcome:
The Flask app is successfully deployed on AWS EKS and accessible via a public endpoint.
Monitoring is set up using CloudWatch, ensuring that the app’s health and resource usage are continuously tracked.
Technologies Used:
AWS EKS
Terraform
Docker
Kubernetes
AWS CloudWatch
This deployment showcases my experience with cloud infrastructure, containerization, and monitoring in production environments, ensuring scalability and high availability.

Please let me know if you need any further details or would like to discuss my work in more depth.
