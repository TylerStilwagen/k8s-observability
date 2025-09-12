# Kubernetes Observability: An Implementation Guide

## Summary

This document shows the design and creation of a Kubernetes observability platform focusing on the open-source Grafana ecosystem. The project demonstrates a hands-on approach to robust monitoring, logging, and tracing capabilities, critical for distributed microservice applications.

Having previously worked with commercial solutions like AppDynamics, Datadog, Dynatrace, this project provided an opportunity to gain deep technical proficiency with the open-source Grafana stack (Loki, Tempo, Prometheus). The solution is scalable for small to mid-size deployments and aims to incorporates industry best practices for automated deployments.

---

## 1. K3s Cluster Setup

The foundation of this project is a lightweight Kubernetes cluster - **K3s** was chosen for its minimal resource footprint and ease of deployment, making it an ideal choice for a local development and testing environment. The cluster was provisioned on an Ubuntu 24.04 VM with 2 cores and 8GB of memory.

### Cluster Bootstrap

The cluster was bootstrapped using the K3s installation script:

```bash
curl -sfL https://get.k3s.io | sudo sh -
```

After a successful installation, the `kubeconfig` file was configured for non-root user access to enable interaction with the cluster:

```bash
# Give self access to kubeconfig file & copy into non-root user home folder
sudo chown $(whoami) /etc/rancher/k3s/k3s.yaml
mkdir ~/.kube && cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc

# Test kubectl command
kubectl get pods -A
```

**Helm**, the standard package manager for Kubernetes, was installed to facilitate the deployments.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Persistent Storage

**Longhorn**, a distributed block storage system, was implemented to provide persistent storage for the stateful components of the observability stack. Longhorn was selected for its native Kubernetes integration, lightweight design, and ease-of-use.

```bash
helm repo add longhorn https://charts.longhorn.io &&
helm repo update &&
helm upgrade --install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace

# Optionally, access the Longhorn web console for troubleshooting
# kubectl port-forward -n longhorn-system svc/longhorn-frontend 80
# http://localhost
```

---

## 2. Observability Stack Installation

The observability stack was deployed using **ArgoCD**, a declarative GitOps continuous delivery tool. This approach ensures that the cluster state is consistently synchronized with the desired state, promoting automation and reliability.

### ArgoCD Installation

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Optionally, access web console at http://localhost:8000
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
kubectl port-forward -n argocd svc/argocd-server 8000
```


### Project Structure and GitOps Workflow

[https://github.com/TylerStilwagen/k8s-observability](https://github.com/TylerStilwagen/k8s-observability)

A Git repository was established to manage the manifests for the observability stack, following an **App-of-Apps** pattern. This structure organizes all infrastructure and application components and aims for a phased deployment.


### Monitoring with Prometheus-Stack

The **Prometheus-Stack** Helm chart was used to deploy a comprehensive metrics monitoring solution, including Prometheus, Grafana, and Alertmanager. This chart provides a set of dashboards and alerting rules, creating a strong baseline for monitoring. A key customization was integrating **Alertmanager with PagerDuty**, demonstrating the ability to configure critical alerts for immediate escalation and incident management.

### Logging with Loki

**Loki** was deployed as the log aggregation system. Unlike traditional solutions, Loki is designed to store logs efficiently as compressed streams, with indexing only on metadata. This approach makes it highly scalable and cost-effective. The configuration utilized persistent storage provided by Longhorn.


### Tracing with Tempo

**Tempo** was implemented to provide distributed tracing capabilities. essential for gaining deep visibility into the performance of microservices, helping to visualize request flows and pinpoint latency issues.

### Data Collection with Grafana Alloy

**Grafana Alloy** was used as a single agent for collecting metrics, logs, and traces from the cluster. Alloy's role is to standardize data collection and forward it to their respective backends (Prometheus, Loki, Tempo)


### Accessing Grafana

After deployment, **Grafana** was exposed via `kubectl port-forward` to allow access to the dashboards:

```bash
kubectl port-forward svc/grafana 3000:80 -n monitoring
# Access web console at http://localhost:3000
```

---

## 3. Example Microservices Application

To demonstrate functionality of the observability stack, a sample microservices application, **mythical-beasts**, was deployed. This application is instrumented to generate telemetry data that is collected by the Grafana Alloy and sent to the respective observability backends.

```bash
git clone https://github.com/grafana/intro-to-mltp.git

# Edit lines 32, 90, 155 from <tracingEndpoint> to 'tempo-distributor.monitoring'
sed -i 's/<tracingEndpoint>/tempo-distributor.monitoring/g' intro-to-mltp/k8s/mythical-deployment.yaml

# Apply all files in folder
kubectl apply -f intro-to-mltp/k8s/mythical
```

---

## 4. Conclusion

Conclusion and **Future Improvements**

This project successfully established a comprehensive, end-to-end observability solution using a 100% open-source stack. The implementation demonstrates proficiency in a wide range of cloud-native technologies, including Kubernetes, GitOps with ArgoCD, and the Grafana ecosystem.

Future enhancements to this project would include:

- **Performance Optimization**: Tuning application configurations for better performance and resource utilization
- **Long-term Storage**: Integrating a dedicated **S3 block storage solution** for long-term data retention
- **Security Enhancement**: Implementing a robust **secrets management solution** to securely handle sensitive credentials, such as the PagerDuty integration key
- **High Availability**: Extending the setup to include multi-node clustering
- **Automated Testing**: Adding testing pipelines to validate observability stack functionality
