# DevOps Automation Stack: Kubernetes-Powered CI/CD Pipeline

**Author:** [Harshwardhan Jadhav]  
**LinkedIn Post:** [(https://www.linkedin.com/posts/jadhavharshwardhan_github-harshwardhanjadhav0full-devops-stack-kubeadm-docker-jenkins-ansible-github-webhooks-aws-activity-7354870452239962115-eqWw?utm_source=share&utm_medium=member_desktop&rcm=ACoAAFQgnCoBgMH2PuaEzuGok5YyLx2BNyWlB8E)]

---

## 📋 Project Summary

This project demonstrates building a production-ready DevOps automation stack by orchestrating Jenkins and Ansible workloads on a self-managed Kubernetes cluster. The implementation showcases containerization, infrastructure automation, and CI/CD pipeline deployment across AWS EC2 instances.

**Core Achievement:** Successfully deployed a scalable automation platform that combines continuous integration capabilities with configuration management, all running on cloud-native Kubernetes infrastructure.

---

## 🎯 Technical Goals Accomplished

• **Container Orchestration** - Deployed Jenkins and Ansible as containerized workloads on Kubernetes  
• **Custom Image Creation** - Built and published Docker images to Docker Hub registry  
• **Infrastructure as Code** - Automated cluster setup and application deployment using YAML manifests  
• **Service Exposure** - Configured NodePort services for external access to Jenkins web interface  
• **Resource Management** - Implemented node selectors and tolerations for workload placement control  
• **Multi-Node Architecture** - Established master-worker node topology for distributed computing

---

## 🏗️ System Architecture

```
    ┌─────────────────────────────────┐
    │        AWS Cloud Environment    │
    │  ┌─────────────┐ ┌─────────────┐ │
    │  │   Master    │ │   Worker    │ │
    │  │   EC2       │ │   EC2       │ │
    │  └──────┬──────┘ └──────┬──────┘ │
    └─────────┼─────────────────┼───────┘
              │                 │
    ┌─────────▼─────────────────▼───────┐
    │     Kubernetes Cluster            │
    │  ┌─────────────┐ ┌─────────────┐  │
    │  │  Jenkins    │ │  Ansible    │  │
    │  │  Container  │ │  Container  │  │
    │  └─────────────┘ └─────────────┘  │
    └───────────────────────────────────┘
                     │
            ┌────────▼────────┐
            │  External Access │
            │  Port: 30080     │
            └──────────────────┘
```

## ⚙️ Infrastructure Provisioning

### Control Plane Node Configuration

**Step 1: System Prerequisites**
```bash
# Update system packages and install container runtime
sudo yum update -y
sudo yum install -y yum-utils device-mapper-persistent-data lvm2 git curl
sudo amazon-linux-extras enable docker
sudo yum install -y docker
sudo systemctl enable docker && sudo systemctl start docker

# Disable swap for Kubernetes compatibility
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

**Step 2: Kubernetes Repository Setup**
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF
```

**Step 3: Cluster Initialization**
```bash
# Install Kubernetes components
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable kubelet

# Initialize cluster with pod network CIDR
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# Configure kubectl access
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Deploy Calico network plugin
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

### Worker Node Integration

**Prerequisites Installation** (Same as control plane steps 1-2)

**Cluster Join Process**
```bash
# Install Kubernetes binaries
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable kubelet

# Join cluster using token from master node
sudo kubeadm join <CONTROL-PLANE-IP>:6443 --token <JOIN-TOKEN> \
    --discovery-token-ca-cert-hash sha256:<CERT-HASH>
```

## 🐳 Container Image Development

### Jenkins Automation Server Image

**Custom Dockerfile Implementation:**
```dockerfile
FROM amazonlinux:2

# Environment configuration
ENV JENKINS_HOME=/var/jenkins_home
ENV JENKINS_VERSION=2.426.1
ENV JENKINS_URL=https://get.jenkins.io/war-stable/${JENKINS_VERSION}/jenkins.war

# Install runtime dependencies
RUN yum update -y && \
    amazon-linux-extras enable java-openjdk11 && \
    yum install -y java-11-openjdk wget git curl unzip && \
    yum clean all

# Create Jenkins user and directories
RUN useradd -m -d $JENKINS_HOME -s /bin/bash jenkins && \
    mkdir -p /opt/jenkins && \
    chown -R jenkins:jenkins $JENKINS_HOME /opt/jenkins

# Download Jenkins WAR file
RUN wget -q -O /opt/jenkins/jenkins.war $JENKINS_URL

USER jenkins
EXPOSE 8080
WORKDIR $JENKINS_HOME
CMD ["java", "-jar", "/opt/jenkins/jenkins.war"]
```

**Image Build and Registry Push:**
```bash
# Build custom Jenkins image
docker build -t <harshj2003/custom-jenkins:latest -f Jenkins-Dockerfile .

# Authenticate with Docker Hub
docker login

# Push to registry
docker push <harshj2003/custom-jenkins:latest
```

### Ansible Configuration Management Image

**Lightweight Dockerfile:**
```dockerfile
FROM amazonlinux:2

# Install Ansible and dependencies
RUN yum update -y && \
    yum install -y python3-pip openssh-clients git && \
    pip3 install ansible

CMD ["/bin/bash"]
```

**Build Process:**
```bash
docker build -t <harshj2003/custom-ansible:latest -f Ansible-Dockerfile .
docker push <harshj2003/custom-ansible:latest
```

## 🚀 Application Deployment on Kubernetes

### Namespace Creation
```bash
# Create dedicated namespace for DevOps tools
kubectl create namespace automation-stack
```

### Jenkins CI/CD Server Deployment

**Deployment Manifest:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins-server
  namespace: default
  labels:
    app: jenkins-ci
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins-ci
  template:
    metadata:
      labels:
        app: jenkins-ci
    spec:
      nodeSelector:
        jenkins-node: "true"
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: jenkins-container
        image: <harshj2003/custom-jenkins:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

**Service Exposure Configuration:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-web-service
  namespace: default
spec:
  type: NodePort
  selector:
    app: jenkins-ci
  ports:
  - name: web-ui
    port: 8080
    targetPort: 8080
    nodePort: 30080
    protocol: TCP
```

**Deploy Jenkins Stack:**
```bash
kubectl apply -f jenkins-deployment.yaml
kubectl apply -f jenkins-service.yaml
```

### Ansible Configuration Pod

**Pod Specification:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ansible-automation
  namespace: default
  labels:
    tool: ansible
spec:
  containers:
  - name: ansible-runner
    image: <harshj2003/custom-ansible:latest
    imagePullPolicy: Always
    command: ["/bin/bash", "-c", "while true; do sleep 3600; done"]
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
```

**Deploy Ansible Workload:**
```bash
kubectl apply -f ansible-pod.yaml
```

## 🌐 Service Access and Management

### Jenkins Web Interface Access

**Check Service Status:**
```bash
# Verify service deployment
kubectl get services -n default
kubectl get pods -n default -l app=jenkins-ci
```

**Access Jenkins Dashboard:**
```
URL: http://<ec2-public-ip>:30080
```

**Retrieve Initial Admin Credentials:**
```bash
# Get pod name
kubectl get pods -n default

# Extract admin password
kubectl exec -n default -it <jenkins-pod-name> -- \
  cat /var/jenkins_home/secrets/initialAdminPassword
```

### Node Management and Scheduling

**Apply Node Labels for Workload Placement:**
```bash
# Label nodes for Jenkins scheduling
kubectl label node <target-node-name> jenkins-node=true

# Verify node labels
kubectl get nodes --show-labels
```

**Dynamic Deployment Updates:**
```bash
# Update deployment with node selector
kubectl patch deployment jenkins-server -n default \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/nodeSelector", "value": {"jenkins-node": "true"}}]'
```

## 🔧 Development Workflow

### Local Development Setup

**File Transfer to EC2:**
```bash
# Transfer Dockerfiles to EC2 instance
scp -i <mykey>.pem Dockerfile ec2-user@<ec2-public-ip>:/home/ec2-user/

# SSH into EC2 for build process
ssh -i <mykey>.pem ec2-user@<ec2-public-ip>
```

**Remote Build Execution:**
```bash
# Navigate to project directory
cd /home/ec2-user/

# Execute Docker build and push
docker build -t <your-registry>/image:tag .
docker push <your-registry>/image:tag
```

## ✅ System Validation

### Health Checks and Monitoring

```bash
# Cluster status verification
kubectl get nodes -o wide
kubectl get pods -n default --show-labels
kubectl get services -n default

# Application logs inspection
kubectl logs -n default -l app=jenkins-ci --tail=50
kubectl logs -n default ansible-automation

# Resource utilization check
kubectl top nodes
kubectl top pods -n default
```

### Troubleshooting Commands

```bash
# Pod debugging
kubectl describe pod <pod-name> -n default
kubectl exec -it <pod-name> -n default -- /bin/bash

# Service connectivity testing
kubectl port-forward service/jenkins-web-service 8080:8080
```

---

## 🎯 Project Outcomes

This implementation successfully demonstrates:

• **Infrastructure Automation** - Automated Kubernetes cluster provisioning on AWS EC2
• **Containerization Strategy** - Custom Docker images for Jenkins and Ansible workloads
• **Service Orchestration** - Kubernetes-native deployment and service management
• **Scalable Architecture** - Multi-node cluster with proper resource allocation
• **DevOps Integration** - Combined CI/CD and configuration management capabilities

The project provides a solid foundation for enterprise-grade DevOps automation, showcasing modern container orchestration and infrastructure management practices.
