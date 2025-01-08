# Deploying a Flask application on EKS
---
### **Prerequisites**

- An AWS account with EKS permissions.
- AWS CLI installed and configured.
- kubectl installed.
- Helm installed.
- Docker (for building the Flask app Docker image).
- GitHub account (for code versioning).

---

### **Step 1: Set Up Your AWS EC2 Instance**

#### 1.1 Launch a t2.micro Instance

1. Go to the **AWS Management Console**.
2. Navigate to **EC2 > Instances > Launch Instance**.
3. Select the **Amazon Linux 2** or **Ubuntu 22.04 LTS** AMI.
4. Choose the **t2.micro** instance type.
5. Configure security groups:
   - Allow **SSH (Port 22)** for your IP.
   - Allow **HTTP (Port 80)** and **HTTPS (Port 443)** for testing.
6. Launch the instance and download the key pair (`.pem` file).

#### 1.2 Connect to the Instance

1. Open a terminal and connect via SSH:
   ```bash
   ssh -i path/to/your-key.pem ubuntu@<EC2-Instance-Public-IP>
   ```

   Replace `path/to/your-key.pem` with the path to your downloaded key and `<EC2-Instance-Public-IP>` with your instance's public IP.

#### 1.3 Update and Install Essentials

Run these commands to update the instance and install necessary tools:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install unzip

# Installing AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Installing Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

# Installing EKSCTL
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin

# Installing Docker
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Installing Docker packages
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Installing kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
```

#### 1.4 Start Docker and Add User Permissions

```bash
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
```

Log out and log back in to apply the changes.

---

### **Step 2: Install `eksctl` and Create EKS Cluster**

#### 2.1 Install `eksctl`

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

#### 2.2 Configure AWS CLI (if not done already)

```bash
aws configure
```
Enter your **AWS Access Key ID**, **Secret Access Key**, **Default Region**, and output format (`json`).

#### 2.3 Create the EKS Cluster

Run this command to create the cluster with **t2.micro** nodes:

```bash
eksctl create cluster \
  --name <cluster-name> \
  --version 1.27 \
  --region <aws-region> \
  --nodegroup-name <nodegroup-name> \
  --nodes 1 \
  --node-type t2.micro \
  --managed
```
This will take ~10-15 minutes.

![EKS](https://github.com/user-attachments/assets/ce85409e-76d9-4fb2-91bf-fdb72d714da2)

---

### **Step 3: Verify EKS Cluster**

1. **Update kubeconfig to connect to the cluster:**

   ```bash
   aws eks --region <aws-region> update-kubeconfig --name <cluster-name>
   ```

2. **Verify the nodes are running:**

   ```bash
   kubectl get nodes
   ```

You should see output like this:

```
NAME                                           STATUS   ROLES    AGE   VERSION
ip-192-168-XX-XX.<aws-region>.compute.internal    Ready    <none>   5m    v1.27.x
```

---

### **Step 4: Dockerize the Flask Application**

#### 4.1 Flask Application Code (`app.py`)

Create a simple Flask app (`app.py`):

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World from Flask on Kubernetes!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
```

#### 4.2 Dockerfile

Create a `Dockerfile`:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt /app/
RUN pip install --no-cache-dir -r requirements.txt

COPY . /app

CMD ["python", "app.py"]
```

#### 4.3 Build and Push Docker Image to Docker Hub

Build the image:

```bash
docker build -t <your-dockerhub-username>/flask-app:v1 .
```

Push it to Docker Hub:

```bash
docker push <your-dockerhub-username>/flask-app:v1
```

---

### **Step 5: Deploy the Flask Application on EKS using Helm**

#### 5.1 Install Helm

Make sure Helm is installed:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

#### 5.2 Create Helm Chart for Flask Application

Generate a Helm chart:

```bash
helm create flask-app
```

Edit the `values.yaml` file to use your custom Docker image:

```yaml
image:
  repository: <your-dockerhub-username>/flask-app
  tag: v1
```

#### 5.3 Install the Application using Helm

Deploy your Flask application:

```bash
helm install flask-app ./flask-app
```

---

### **Step 6: Expose the Flask Application via LoadBalancer**

#### 6.1 Create Service

Create a Kubernetes service to expose the Flask app via a **LoadBalancer**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-app
spec:
  selector:
    app: flask-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

Apply the service definition:

```bash
kubectl apply -f flask-app-service.yaml
```

#### 6.2 Get the LoadBalancer URL

Once the service is deployed, it will be assigned an external IP or DNS name via the AWS Load Balancer. Run:

```bash
kubectl get svc flask-app
```

You’ll see something like this:

```bash
NAME        TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)        AGE
flask-app   LoadBalancer   10.100.158.150   <external-ip>   80:32223/TCP   12m
```

Copy the `EXTERNAL-IP` and use it to access the Flask app.

---

### **Step 7: Troubleshooting**

If the pod status shows `CrashLoopBackOff` or another failure, follow these steps:

#### 7.1 Check Pod Logs

```bash
kubectl logs flask-app-<pod-id>
```

#### 7.2 Describe the Pod

```bash
kubectl describe pod flask-app-<pod-id>
```

---

### **Step 8: Access the Flask Application**

Once the service is deployed and the LoadBalancer URL is available, access the Flask app by navigating to the external IP or DNS.

Example:

```bash
http://<loadbalancer-url>:80
```

You should see:

<img width="465" alt="Webpage" src="https://github.com/user-attachments/assets/accc996d-40a6-4539-a65b-edeb1ff339b4" />

---

### **Step 9: Cleaning Up**

To clean up resources in AWS EKS, delete the Helm release:

```bash
helm uninstall flask-app
```

Then, follow the comprehensive guide below to delete the AWS resources associated with your project.

---

### **AWS Resources Cleanup**

Here’s a comprehensive guide for cleaning up AWS resources associated with your project:

1. **EKS Cluster & Node Groups Cleanup**  
   Delete EKS cluster and node groups:  
   ```bash
   aws eks delete-cluster --name <cluster-name>
   aws eks delete-nodegroup --cluster-name <cluster-name> --nodegroup-name <nodegroup-name>
   ```

2. **EC2 Instances Cleanup**  
   List and terminate EC2 instances used for your cluster:  
   ```bash
   aws ec2 describe-instances
   aws ec2 terminate-instances --instance-ids <instance-id>
   ```

3. **Load Balancer Cleanup**  
   List and delete Elastic Load Balancers:  
   ```bash
   aws elb describe-load-balancers
   aws elb delete-load-balancer --load-balancer-name <load-balancer-name>
   ```

4. **IAM Roles Cleanup**  
   List and delete IAM roles and policies:  
   ```bash
   aws iam list-roles
   aws iam delete-role --role-name <role-name>
   ```

5. **Security Groups Cleanup**  
   List and delete unused security groups:  
   ```bash
   aws ec2 describe-security-groups
   aws ec2 delete-security-group --group-id <security-group-id>
   ```

6. **VPC Cleanup**  
   Delete VPC and associated resources:  
   ```bash
   aws ec2 describe-vpcs
   aws ec2 delete-vpc --vpc-id <vpc-id>
   ```

7. **S3 Buckets Cleanup**  
   List and delete unused S3 buckets:  
   ```bash
   aws s3 ls
   aws s3 rb s3://<bucket-name> --force
   ```

8. **CloudWatch Logs Cleanup**  
   List and delete log groups:  
   ```bash
   aws logs describe-log-groups
   aws logs delete-log-group --log-group-name <log-group-name>
   ```

9. **Elastic IP Cleanup**  
   List and release unused Elastic IPs:  
   ```bash
   aws ec2 describe-addresses
   aws ec2 release-address --allocation-id <allocation-id>
   ```

10. **ECR Cleanup**  
    List and delete ECR repositories:  
    ```bash
    aws ecr describe-repositories
    aws ecr delete-repository --repository-name <repository-name> --force
    ```

11. **CloudFormation Cleanup**  
    List and delete CloudFormation stacks:  
    ```bash
    aws cloudformation describe-stacks
    aws cloudformation delete-stack --stack-name <stack-name>
    ```

---

**Final Verification**  
- Check the **AWS Console** to ensure that all resources have been deleted.
- Verify in **AWS Billing Dashboard** that no unexpected charges are occurring.

This process should help clean up the AWS resources and prevent any future costs.
