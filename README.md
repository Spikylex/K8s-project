# WordPress on Kubernetes (Helm Chart)

Deploys WordPress + MariaDB on Kubernetes, migrated from the official
[docker/awesome-compose WordPress example](https://github.com/docker/awesome-compose/blob/master/official-documentation-samples/wordpress/README.md),
fronted by an NGINX Ingress Controller, with Grafana/Prometheus monitoring
for container uptime.

## Prerequisites
Before installing this chart, you need a working cluster and toolchain:

**A Kubernetes cluster** — this project was built and tested against [minikube](https://minikube.sigs.k8s.io/docs/start/) running on an Ubuntu EC2 instance.
**Docker Engine** (if running minikube with the docker driver):
 - sudo apt update
 - sudo apt install docker.io docker-compose-v2 -y
 - sudo systemctl enable --now docker
 - sudo usermod -aG docker $USER
 - newgrp docker
   
**kubectl** — [install instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
**minikube** — [install instructions](https://minikube.sigs.k8s.io/docs/start/)
  - minikube start --driver=docker
  
**Helm 3+** — [install instructions](https://helm.sh/docs/intro/install/)
**NGINX Ingress Controller**, installed via Helm (not part of this chart, since it's shared cluster infrastructure rather than app-specific):
  - helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  - helm repo update
  - helm install ingress-nginx ingress-nginx/ingress-nginx \
  -  --namespace ingress-nginx --create-namespace
    
**A container registry** with the "wordpress" and "mariadb" images pushed to it (this project used Amazon ECR). Update values.yaml with your own image URIs before installing.
**(Optional) kube-prometheus-stack**, for the Grafana uptime panel:
 - helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
 - helm repo update
 - helm install kube-prom-stack prometheus-community/kube-prometheus-stack \
 -   --namespace monitoring --create-namespace
  
**On low-memory environments (e.g. EC2 "t3.small" /"t3.micro"):** minikube plus the full app plus monitoring can exceed 2GB RAM. Adding swap is strongly recommended so the app will run and don't overkill the memory:
- sudo fallocate -l 2G /swapfile
- sudo chmod 600 /swapfile
- sudo mkswap /swapfile
- sudo swapon /swapfile
- echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
- ```

## Configuration
Edit "wordpress-chart/values.yaml" before installing:
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

If pulling images from a private registry, create an image pull secret named "ecr-secret" (or update "templates/wordpress-deployment.yaml" to reference your own secret name):
 - kubectl create secret docker-registry ecr-secret \
 - --docker-server=<account-id>.dkr.ecr.<region>.amazonaws.com \
 - --docker-username=AWS \
 - --docker-password=$(aws ecr get-login-password --region <region>)

## Running the Helm chart
**Install** (first time):
- helm install wordpress ./wordpress-chart

**Upgrade** (after changing "values.yaml" or any template):
- helm upgrade wordpress ./wordpress-chart

**Preview changes before applying them** (always do this before an upgrade on a live environment):
- helm diff upgrade wordpress ./wordpress-chart   # requires the helm-diff plugin
# or, without the plugin:
- helm template wordpress ./wordpress-chart
  
**Uninstall** (removes everything the chart created — Deployment, StatefulSet, Services, Ingress, Secret. Does NOT delete the PVC by default, so your database data survives):
- helm uninstall wordpress

**List installed releases:**
- helm list

**Check status of what's running:**
- kubectl get pods
- kubectl get svc
- kubectl get statefulsets
- kubectl get pvc

## Access the app
### On the EC2 instance itself
Point the Ingress hostname at your cluster's IP:
- echo "$(minikube ip) wordpress.local" | sudo tee -a /etc/hosts

Find the ingress controller's NodePort:
- kubectl get svc -n ingress-nginx ingress-nginx-controller

Then test with curl:
- curl -H "Host: wordpress.local" http://$(minikube ip):<nodeport>

### From your local machine's browser (via SSH tunnel)
Since minikube runs inside the EC2 instance, you can't reach it directly from your laptop's browser without tunneling the port through SSH first.

**1. Find the NodePort on the EC2 instance:**
- kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.spec.ports[0].nodePort}'
**2. Get minikube's internal IP on the EC2 instance:**
- minikube ip
**3. From your local machine (not the EC2 instance), open a terminal and run:**
- ssh -i /path/to/your-key.pem -L 8080:<minikube-ip>:<nodeport> ubuntu@<ec2-public-ip>

- "-i /path/to/your-key.pem" — your EC2 SSH key (Windows users: point this at your ".pem" file's actual path)
- "-L 8080:<minikube-ip>:<nodeport>" — forwards your local port "8080" to minikube's IP+port *inside* the EC2 instance
- Leave this terminal open — the tunnel only works while the SSH session is active

**4. Open in your browser:**
http://<ec2-public-ip>:8080

### Quick access without a hosts file edit (bypasses Ingress)
If you just want to view the site quickly, port-forward directly to the WordPress Service instead of going through Ingress:
**On the EC2 instance:**
- kubectl port-forward svc/wordpress 8080:80

(use your actual Service name — check with "kubectl get svc" if unsure)
**On your local machine:**
- ssh -i /path/to/your-key.pem -L 8080:localhost:8080 ubuntu@<ec2-public-ip>

**In your browser:**
http://localhost:8080
Note: this bypasses the Ingress Controller entirely, so it doesn't exercise the full Ingress → Service → Pod path — useful for quick checks.

## Monitoring (Grafana)
Grafana isn't exposed externally by default — the same SSH tunnel pattern applies, but through "kubectl port-forward" first since Grafana's Service is "ClusterIP", not "NodePort".

**On the EC2 instance**, start a port-forward (leave running in its own terminal):
- kubectl port-forward -n monitoring svc/kube-prom-stack-grafana 3000:80

**From your local machine**, open a second SSH tunnel:
- ssh -i /path/to/your-key.pem -L 3000:localhost:3000 ubuntu@<ec2-public-ip>

**Then open in your browser:**
http://localhost:3000

**Get the auto-generated admin credentials (run on the EC2 instance):**
- kubectl get secret -n monitoring kube-prom-stack-grafana -o jsonpath="{.data.admin-user}" | base64 -d; echo
- kubectl get secret -n monitoring kube-prom-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
A Grafana panel tracks container uptime using the "kube_pod_container_status_running" Prometheus metric, visualized as a State Timeline.

## Architecture
- **Deployment** ("wordpress") — 2 replicas, stateless, serves the WordPress app
- **StatefulSet** ("db") — 1 replica MariaDB, stable identity + dedicated storage
- **PVC** — persists "/var/lib/mysql" across restarts
- **Secret** — holds DB credentials, referenced by both the Deployment and StatefulSet
- **Services** — a ClusterIP Service load-balances WordPress traffic; a headless Service gives the StatefulSet stable DNS
- **Ingress** — routes external HTTP traffic through the NGINX Ingress Controller to the WordPress Service

## Notes
- Database credentials in "values.yaml" are for demo purposes. In production, use a secrets manager (e.g. AWS Secrets Manager, Sealed Secrets, External Secrets Operator) rather than committing plaintext values.
- Ingress-NGINX (the community controller) is used here, not F5's commercial NGINX Ingress Controller — the two share a similar name but are separate projects with different installation and configuration.
