# homelab

# 1. Add Helm repo if not already
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 2. Create namespace
kubectl create namespace monitoring

# 3. Install Helm chart with your values
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring -f monitoring/values.yaml

# 4. Apply Ingress manifests
kubectl apply -f monitoring/ingress/grafana-ingress.yaml
kubectl apply -f monitoring/ingress/prometheus-ingress.yaml

