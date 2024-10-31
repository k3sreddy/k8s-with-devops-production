# k8s-with-devops-production
k8s with devops on-prem production

Pre-requisites:
2 Rocky Linux 9.4 servers with 4 CPU cores, 16GB RAM, and 50GB SSD for haproxy
3 Rocky Linux 9.4 servers with 4 CPU cores, 16GB RAM, and 50GB SSD for kubernetes master
4 Rocky Linux 9.4 servers with 4 CPU cores, 16GB RAM, and 50GB SSD for kubernetes workernode
1 Rocky Linux 9.4 servers with 4 CPU cores, 16GB RAM, and 50GB SSD for devops and devsecops tools
A domain name for my cluster (e.g., icsmobile.local)
IP address for the nodes starts with 172.16.10.131 and end with 172.16.10.141

# ICS Mobile Infrastructure Setup

## Table of Contents
1. [Kubernetes Cluster Setup](#kubernetes-cluster-setup)
2. [Storage Configuration](#storage-configuration)
3. [Networking Setup](#networking-setup)
4. [Application Deployment and Security](#application-deployment-and-security)
5. [Monitoring and Logging](#monitoring-and-logging)
6. [Backup and Disaster Recovery](#backup-and-disaster-recovery)
7. [CI/CD Pipeline Setup](#cicd-pipeline-setup)
8. [Security Hardening](#security-hardening)
9. [Detailed Kubernetes Configurations](#detailed-kubernetes-configurations)
10. [Advanced Monitoring and Alerting](#advanced-monitoring-and-alerting)
11. [Advanced Security Configurations](#advanced-security-configurations)
12. [Backup and Disaster Recovery Enhancements](#backup-and-disaster-recovery-enhancements)
13. [CI/CD Pipeline Enhancements](#cicd-pipeline-enhancements)

1. Kubernetes Cluster Setup
1.1 Prepare All Nodes
Perform these steps on all nodes (HAProxy, Masters, Workers, and DevOps):

# Update the system
sudo dnf update -y

# Install necessary packages
sudo dnf install -y wget curl vim

# Disable SELinux
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Enable IP forwarding
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Load necessary modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
overlay
EOF
sudo modprobe br_netfilter
sudo modprobe overlay

# Configure firewall
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --reload

# Install containerd
sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y containerd.io
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable contained

1.2 Set up HAProxy Load Balancer
On the HAProxy nodes:

# Install HAProxy
sudo dnf install -y haproxy

# Configure HAProxy
sudo tee /etc/haproxy/haproxy.cfg <<EOF
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend kubernetes
    bind *:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server k8s-master-0 172.16.10.133:6443 check fall 3 rise 2
    server k8s-master-1 172.16.10.134:6443 check fall 3 rise 2
    server k8s-master-2 172.16.10.135:6443 check fall 3 rise 2
EOF

# Start and enable HAProxy
sudo systemctl start haproxy
sudo systemctl enable haproxy

1.3 Install Kubernetes Components
On all Kubernetes nodes (Masters and Workers):

# Add Kubernetes repository
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Install Kubernetes components
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet

1.4 Initialize the Kubernetes Cluster
On the first master node:

# Initialize the cluster
sudo kubeadm init --control-plane-endpoint="172.16.10.131:6443" --upload-certs --apiserver-advertise-address=172.16.10.133 --pod-network-cidr=10.244.0.0/16

# Set up kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Calico network plugin
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

1.5 Join Other Master Nodes
Run the join command provided by kubeadm init on the other master nodes.

1.6 Join Worker Nodes
Run the join command provided by kubeadm init on all worker nodes.

1.7 Verify Cluster Status

kubectl get nodes

2. Storage Configuration

   We'll set up Rook-Ceph for distributed storage:

# On the master node
git clone --single-branch --branch release-1.8 https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
kubectl create -f cluster.yaml

# Wait for all pods to be running
kubectl -n rook-ceph get pod

# Create a storage class
kubectl create -f csi/rbd/storageclass.yaml

3. Networking Setup
   Install and configure Ingress-NGINX:

# Install Ingress-NGINX
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.0/deploy/static/provider/cloud/deploy.yaml

# Create a basic network policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

4. Application Deployment and Security
Deploy the Python Gunicorn application:

# Create a deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gunicorn-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gunicorn-app
  template:
    metadata:
      labels:
        app: gunicorn-app
    spec:
      containers:
      - name: gunicorn-app
        image: your-private-registry/gunicorn-app:latest
        ports:
        - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: gunicorn-app
spec:
  selector:
    app: gunicorn-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gunicorn-app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - gunicorn.icsmobile.local
    secretName: gunicorn-tls
  rules:
  - host: gunicorn.icsmobile.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gunicorn-app
            port: 
              number: 80
EOF

Deploy the internal Python script as a CronJob:

cat <<EOF | kubectl apply -f -
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: report-generator
spec:
  schedule: "0 1 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: report-generator
            image: your-private-registry/report-generator:latest
            command: ["python", "/app/generate_report.py"]
          restartPolicy: OnFailure
EOF

5. Monitoring and Logging
Install Prometheus and Grafana:

# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack

# Access Grafana (default credentials: admin/prom-operator)
kubectl port-forward service/prometheus-grafana 3000:80

Install ELK stack:


# Add Elastic Helm repo
helm repo add elastic https://helm.elastic.co
helm repo update

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch

# Install Kibana
helm install kibana elastic/kibana

# Install Filebeat
helm install filebeat elastic/filebeat

6. Backup and Disaster Recovery
Install and configure Velero:

# Install Velero CLI
wget https://github.com/vmware-tanzu/velero/releases/download/v1.6.0/velero-v1.6.0-linux-amd64.tar.gz
tar -xvf velero-v1.6.0-linux-amd64.tar.gz
sudo mv velero-v1.6.0-linux-amd64/velero /usr/local/bin/

# Install Velero in the cluster (adjust for your specific backup storage)
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.2.0 \
    --bucket velero-backups \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000

# Create a backup
velero backup create my-backup --include-namespaces=default,kube-system

7. CI/CD Pipeline Setup
Install and configure GitLab:

# Add GitLab Helm repo
helm repo add gitlab https://charts.gitlab.io/
helm repo update

# Install GitLab
helm install gitlab gitlab/gitlab \
  --set global.hosts.domain=icsmobile.local \
  --set certmanager-issuer.email=your-email@example.com

Install Jenkins:

# Add Jenkins Helm repo
helm repo add jenkins https://charts.jenkins.io
helm repo update

# Install Jenkins
helm install jenkins jenkins/jenkins

Install SonarQube:

# Add SonarQube Helm repo
helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm repo update

# Install SonarQube
helm install sonarqube sonarqube/sonarqube

Install Trivy:

# Install Trivy
sudo rpm -ivh https://github.com/aquasecurity/trivy/releases/download/v0.19.2/trivy_0.19.2_Linux-64bit.rpm

Install ArgoCD:

# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

8. Security Hardening
Implement Pod Security Policies:

cat <<EOF | kubectl apply -f -
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: RunAsAny
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'persistentVolumeClaim'

EOF

Implement Network Policies:

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

Configure RBAC:

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
roleRef:
  name: pod-reader
  kind: Role
subjects:
- kind: User
  name: your-username
  apiGroup: rbac.authorization.k8s.io
EOF
