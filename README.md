WordPress on Kubernetes (Helm Chart)

Deploys WordPress + MariaDB on Kubernetes, migrated from the official docker/awesome-compose WordPress example, fronted by an NGINX Ingress Controller, with Grafana/Prometheus monitoring for container uptime.

Prerequisites

Before installing this chart, you need a working cluster and toolchain:

A Kubernetes cluster — this project was built and tested against minikube running on an Ubuntu EC2 instance.
Docker Engine (if running minikube with the docker driver):
bash
  sudo apt update
  sudo apt install docker.io docker-compose-v2 -y
  sudo systemctl enable --now docker
  sudo usermod -aG docker $USER
  newgrp docker
kubectl — install instructions
minikube — install instructions
bash
  minikube start --driver=docker
Helm 3+ — install instructions
NGINX Ingress Controller, installed via Helm (not part of this chart, since it's shared cluster infrastructure rather than app-specific):
bash
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  helm repo update
  helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx --create-namespace
A container registry with the wordpress and mariadb images pushed to it (this project used Amazon ECR). Update values.yaml with your own image URIs before installing.
(Optional) kube-prometheus-stack, for the Grafana uptime panel:
bash
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm repo update
  helm install kube-prom-stack prometheus-community/kube-prometheus-stack \
    --namespace monitoring --create-namespace

Note on low-memory environments (e.g. EC2 t3.small/t3.micro): minikube plus the full app plus monitoring can exceed 2GB RAM. Adding swap is strongly recommended:

bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
Configuration

Edit wordpress-chart/values.yaml before installing:

yaml
db:
  rootPassword: <your-root-password>
  name: wordpress
  user: wordpress
  password: <your-db-password>
  storage: 2Gi
  image: <your-registry>/<repo>:mariadb

wordpress:
  image: <your-registry>/<repo>:wordpress
  replicas: 2
  host: wordpress.local

If pulling images from a private registry, create an image pull secret named ecr-secret (or update templates/wordpress-deployment.yaml to reference your own secret name):

bash
kubectl create secret docker-registry ecr-secret \
  --docker-server=<account-id>.dkr.ecr.<region>.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region <region>)
Install
bash
helm install wordpress ./wordpress-chart

Check status:

bash
kubectl get pods
kubectl get svc
kubectl get statefulsets
kubectl get pvc
Access the app

Point the Ingress hostname at your cluster's IP:

bash
echo "$(minikube ip) wordpress.local" | sudo tee -a /etc/hosts

Find the ingress controller's NodePort:

bash
kubectl get svc -n ingress-nginx ingress-nginx-controller

Then visit http://wordpress.local:<nodeport> in a browser, or test with curl:

bash
curl -H "Host: wordpress.local" http://$(minikube ip):<nodeport>
Monitoring

A Grafana panel tracks container uptime using the kube_pod_container_status_running Prometheus metric, visualized as a State Timeline.

To access Grafana:

bash
kubectl port-forward -n monitoring svc/kube-prom-stack-grafana 3000:80

Then open http://localhost:3000. Get the auto-generated admin credentials:

bash
kubectl get secret -n monitoring kube-prom-stack-grafana -o jsonpath="{.data.admin-user}" | base64 -d; echo
kubectl get secret -n monitoring kube-prom-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
Architecture
Deployment (wordpress) — 2 replicas, stateless, serves the WordPress app
StatefulSet (db) — 1 replica MariaDB, stable identity + dedicated storage
PVC — persists /var/lib/mysql across restarts
Secret — holds DB credentials, referenced by both the Deployment and StatefulSet
Services — a ClusterIP Service load-balances WordPress traffic; a headless Service gives the StatefulSet stable DNS
Ingress — routes external HTTP traffic through the NGINX Ingress Controller to the WordPress Service
Notes
Database credentials in values.yaml are for demo purposes. In production, use a secrets manager (e.g. AWS Secrets Manager, Sealed Secrets, External Secrets Operator) rather than committing plaintext values.
Ingress-NGINX (the community controller) is used here, not F5's commercial NGINX Ingress Controller — the two share a similar name but are separate projects with different installation and configuration.
