# Canary Deployment Demo

This project demonstrates a Kubernetes canary deployment setup using Helm charts, with Prometheus and Grafana monitoring integration.

## Project Structure

```
canary-demo/
├── app/                      # Main application code
│   ├── app.py               # Flask application with Prometheus metrics
│   └── requirements.txt     # Python dependencies
├── canary-demo/             # Helm chart
│   ├── Chart.yaml
│   ├── values.yaml          # Chart configuration values
│   └── templates/           # Kubernetes manifests
├── tests/                   # Test files
│   └── test_app.py         # Application unit tests
└── answers.yml             # Deployment metrics and observations
```

## Setup Steps

1. **Add Prometheus Helm Repository**
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

2. **Install Prometheus & Grafana Stack**
   ```bash
   helm install prometheus prometheus-community/kube-prometheus-stack -n canary-demo
   ```

3. **Deploy Canary Application**
   ```bash
   helm install canary-demo ./canary-demo -n canary-demo
   ```

   The deployment includes:
   - Main deployment (4 pods, v1)
   - Canary deployment (1 pod, v2)
   - Ingress with 80:20 traffic split

4. **Port Forward Services**
   ```bash
   kubectl port-forward svc/canary-demo 8080:80 -n canary-demo &
   kubectl port-forward svc/canary-demo-canary 8081:80 -n canary-demo &
   kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8082:80 &
   kubectl port-forward svc/prometheus-grafana 3000:80 -n canary-demo &
   kubectl port-forward svc/prometheus-operated 9090:9090 -n canary-demo &
   ```

## Testing Canary Deployment

1. **Test Traffic Distribution**
   ```bash
   for i in {1..20}; do curl -H "Host: canary-demo.local" localhost:8082; done
   ```
   Verify ~80% of responses show white background (v1) and ~20% show green background (v2).

2. **Run Unit Tests**
   ```bash
   python ./tests/test_app.py
   ```

## Canary Updates

1. **Deploy v3 Canary**
   ```bash
   # Update values.yaml
   canary:
     enabled: true
     replicaCount: 1
     image:
       tag: "v3"
   config:
     canary:
       appColor: "yellow"
   
   # Apply update
   helm upgrade canary-demo ./canary-demo -n canary-demo
   ```

2. **Rollback to v2**
   ```bash
   # Check revision history
   helm history canary-demo -n canary-demo
   
   # Rollback
   helm rollback canary-demo 3 -n canary-demo
   ```

## Monitoring

1. **Access Grafana**
   - URL: http://localhost:3000
   - Username: admin
   - Password: get secret from kubectl get secret command and decode it using base64 -d 

2. **Key Metrics**
   - Request rate by version:
     ```
     sum(rate(http_requests_total[5m])) by (version)
     ```
   - Error rate by version:
     ```
     sum(rate(http_requests_total{code=~"5.*"}[5m])) by (version)
     ```

## Error Budget

Run the error budget calculation:
```bash
python compute_error_budget.py
```

## Troubleshooting Notes

1. **Ingress Configuration**
   - Added `kubernetes.io/ingress.class: "nginx"` annotation to fix ingress controller discovery
   - Verified with `kubectl get ingress -n canary-demo`

2. **Metrics Collection**
   - Application exposes metrics on `/metrics` endpoint
   - ServiceMonitor configured to scrape both main and canary deployments
   - Metrics available in Prometheus under `http_requests_total` and process metrics

## Deployment Observations

1. **Traffic Distribution**
   - Achieved approximately 80:20 split between main and canary
   - Verified through repeated curl tests

2. **Version Management**
   - Successfully deployed v3 (yellow background)
   - Confirmed rollback functionality to v2 (green background)

3. **Monitoring**
   - Prometheus successfully collecting application metrics
   - Process metrics (CPU, memory) tracked for both versions
   - Error budget calculation based on 99.9% SLO

## Improvements Suggested

1. Add ingress.class annotations in Helm chart by default
2. Add post-update hook for ConfigMap changes
3. Include automated smoke tests for canary UI changes
4. Enhance monitoring with custom dashboards
5. Add readiness probes for smoother version transitions