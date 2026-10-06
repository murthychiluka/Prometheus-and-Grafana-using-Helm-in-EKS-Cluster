# Deployment of Prometheus and Grafana using Helm in EKS Cluster

## Overview

This document provides a complete step-by-step guide to deploy:

* Prometheus
* Grafana
* Alertmanager
* kube-state-metrics
* node-exporter

![Prometheus Grafana EKS Architecture](https://raw.githubusercontent.com/aniljadhavmca/Prometheus-and-Grafana-using-Helm-in-EKS-Cluster/main/images/grphana-pgrduty.png)

using Helm on an Amazon EKS cluster.

This setup helps monitor:

* Kubernetes cluster health
* Node metrics
* Pod metrics
* CPU & Memory utilization
* Alerts & notifications
* Dashboard visualization

---

# Architecture

## Components

| Component           | Purpose                        |
| ------------------- | ------------------------------ |
| Prometheus          | Collects and stores metrics    |
| Grafana             | Visualization dashboards       |
| Alertmanager        | Sends alerts and notifications |
| kube-state-metrics  | Kubernetes object metrics      |
| node-exporter       | Node/server metrics            |
| Prometheus Operator | Manages Prometheus CRDs        |

---

# kubectl Installation

## Download kubectl

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
```

## Make kubectl Executable

```bash
chmod +x ./kubectl
```

## Move Binary to PATH

```bash
sudo mv ./kubectl /usr/local/bin/kubectl
```

## Verify Installation

```bash
kubectl version --client
```

---

# eksctl Installation

## Download and Extract Latest Release

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```

## Move Binary to PATH

```bash
sudo mv /tmp/eksctl /usr/local/bin
```

## Verify Installation

```bash
eksctl version
```

---

# Helm Installation

## Download Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Verify Helm

```bash
helm version
```

---

# AWS CLI Configuration

## Configure AWS Credentials

```bash
aws configure
```

Provide:

* AWS Access Key
* AWS Secret Key
* Region
* Output format

Verify:

```bash
aws sts get-caller-identity
```

---

# Prerequisites

## Install Required Tools

### AWS CLI

```bash
aws --version
```

### kubectl

```bash
kubectl version --client
```

### eksctl

```bash
eksctl version
```

### Helm

```bash
helm version
```

---

# Create EKS Cluster

```bash
eksctl create cluster \
--name monitoring-cluster \
--region us-east-1 \
--nodegroup-name workers \
--node-type t3.medium \
--nodes 2
```

Verify cluster:

```bash
kubectl get nodes
```

---

# Create Namespace

```bash
kubectl create namespace prometheus
```

Verify:

```bash
kubectl get ns
```

---

# Add Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

Update repo:

```bash
helm repo update
```

Verify:

```bash
helm search repo prometheus-community
```

---

# Install kube-prometheus-stack

```bash
helm install stable prometheus-community/kube-prometheus-stack -n prometheus
```

Verify installation:

```bash
helm list -A
```

---

# Check Pods

```bash
kubectl get pods -n prometheus
```

Expected output:

```text
alertmanager-stable-kube-prometheus-sta-alertmanager-0
prometheus-stable-kube-prometheus-sta-prometheus-0
stable-grafana
stable-kube-prometheus-sta-operator
stable-kube-state-metrics
stable-prometheus-node-exporter
```

Watch pods live:

```bash
kubectl get pods -n prometheus -w
```

---

# Check Services

```bash
kubectl get svc -n prometheus
```

Expected services:

| Service       | Port |
| ------------- | ---- |
| Grafana       | 80   |
| Prometheus    | 9090 |
| Alertmanager  | 9093 |
| node-exporter | 9100 |

---

# Expose Grafana using LoadBalancer

## Option 1 — Patch Command

```bash
kubectl patch svc stable-grafana -n prometheus -p '{"spec":{"type":"LoadBalancer"}}'
```

Verify:

```bash
kubectl get svc -n prometheus
```

Wait until EXTERNAL-IP is assigned.

Access:

```text
http://<EXTERNAL-IP>
```

---

# Expose Prometheus using LoadBalancer

```bash
kubectl patch svc stable-kube-prometheus-sta-prometheus -n prometheus -p '{"spec":{"type":"LoadBalancer"}}'
```

Verify:

```bash
kubectl get svc -n prometheus
```

Access:

```text
http://<EXTERNAL-IP>:9090
```

---

# Grafana Login Credentials

## Username

```text
admin
```

## Get Password

```bash
kubectl get secret stable-grafana -n prometheus -o jsonpath="{.data.admin-password}" | base64 --decode
```

---

# Configure Prometheus Datasource in Grafana

Usually auto-configured.

If manual configuration needed:

## Prometheus URL

```text
http://stable-kube-prometheus-sta-prometheus.prometheus.svc.cluster.local:9090
```

---

# Useful Prometheus Queries

## CPU Usage %

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

---

## Memory Usage %

```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

---

## Disk Usage %

```promql
100 - ((node_filesystem_avail_bytes * 100) / node_filesystem_size_bytes)
```

---

## Running Pods Count

```promql
count(kube_pod_status_phase{phase="Running"})
```

---

## Node Status

```promql
kube_node_status_condition{condition="Ready",status="true"}
```

---

## Pod CPU Usage

```promql
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)
```

---

## Pod Memory Usage

```promql
sum(container_memory_working_set_bytes{container!=""}) by (pod)
```

---

# Recommended Grafana Panels

| Metric          | Panel Type          |
| --------------- | ------------------- |
| CPU Usage       | Gauge / Time Series |
| Memory Usage    | Gauge               |
| Disk Usage      | Gauge               |
| Pod Count       | Stat                |
| Node Health     | Stat                |
| Network Traffic | Time Series         |

---

# Configure Thresholds

## Running Pods Example

Query:

```promql
count(kube_pod_status_phase{phase="Running"})
```

Thresholds:

| Value | Color |
| ----- | ----- |
| 0     | Red   |
| 1+    | Green |

---

# CPU Stress Testing for Alert Validation

## Create CPU Stress Pod

```bash
kubectl run cpu-stress --image=progrium/stress --restart=Never -- stress --cpu 4 --timeout 300s
```

Monitor:

```bash
kubectl top nodes
```

Delete pod:

```bash
kubectl delete pod cpu-stress
```

---

# PagerDuty Integration

## Overview

PagerDuty integration helps send:

* Critical alerts
* CPU alerts
* Memory alerts
* Pod failures
* Node down notifications

from Alertmanager to PagerDuty.

---

# Create PagerDuty Service

1. Login to PagerDuty
2. Create Service
3. Select Prometheus Integration
4. Copy Integration Key

Example:

```text
1234567890abcdef1234567890abcdef
```

---

# Create Alertmanager Configuration

Create file:

```bash
vi alertmanager-values.yaml
```

Example configuration:

```yaml
grafana:
  enabled: true

alertmanager:
  config:
    global:
      resolve_timeout: 5m

    route:
      receiver: pagerduty
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 1h

    receivers:
      - name: pagerduty
        pagerduty_configs:
          - routing_key: YOUR_PAGERDUTY_INTEGRATION_KEY
            severity: critical
```

---

# Upgrade Helm Release with PagerDuty Config

```bash
helm upgrade stable prometheus-community/kube-prometheus-stack \
-n prometheus \
-f alertmanager-values.yaml
```

---

# Create CPU Alert Rule

Create file:

```bash
vi cpu-alert-rule.yaml
```

Add:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cpu-alert-rules
  namespace: prometheus
spec:
  groups:
  - name: cpu-alerts
    rules:
    - alert: HighCPUUsage
      expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 70
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: High CPU Usage Detected
        description: CPU usage exceeded 70%
```

Apply:

```bash
kubectl apply -f cpu-alert-rule.yaml
```

---

# Test PagerDuty Alert

## Generate CPU Stress

```bash
kubectl run cpu-stress --image=progrium/stress --restart=Never -- stress --cpu 4 --timeout 300s
```

Monitor:

```bash
kubectl top nodes
```

Expected Flow:

1. CPU usage increases
2. Prometheus detects threshold breach
3. Alert fires
4. Alertmanager receives alert
5. PagerDuty incident created
6. Notification sent

---

# Verify Alerts in Prometheus

Open Prometheus UI:

```text
http://<PROMETHEUS-LOADBALANCER>:9090
```

Navigate:

```text
Alerts
```

---

# Verify Alertmanager

Expose Alertmanager:

```bash
kubectl patch svc stable-kube-prometheus-sta-alertmanager -n prometheus -p '{"spec":{"type":"LoadBalancer"}}'
```

Access:

```text
http://<ALERTMANAGER-LOADBALANCER>:9093
```

---

# PagerDuty Integration Flow

1. Prometheus detects threshold breach
2. Alertmanager receives alert
3. Alertmanager forwards alert to PagerDuty
4. PagerDuty creates incident
5. Notification sent to teams

---

# Install Metrics Server

If kubectl top commands fail:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verify:

```bash
kubectl get deployment metrics-server -n kube-system
```

---

# Useful kubectl Commands

## Check Nodes

```bash
kubectl get nodes
```

## Check Pods

```bash
kubectl get pods -A
```

## Check Services

```bash
kubectl get svc -A
```

## Describe Pod

```bash
kubectl describe pod <pod-name> -n prometheus
```

## Pod Logs

```bash
kubectl logs <pod-name> -n prometheus
```

---

# Helm Commands

## Install Chart

```bash
helm install stable prometheus-community/kube-prometheus-stack -n prometheus
```

## Check Releases

```bash
helm list -A
```

## Uninstall Release

```bash
helm uninstall stable -n prometheus
```

---

# Delete Complete Setup

## Remove Helm Release

```bash
helm uninstall stable -n prometheus
```

## Delete Namespace

```bash
kubectl delete namespace prometheus
```

---

# Troubleshooting

## Pods Stuck in Pending

Check:

```bash
kubectl describe pod <pod-name> -n prometheus
```

---

## External-IP Pending

Possible reasons:

* AWS LoadBalancer not provisioned yet
* Missing IAM permissions
* Subnet tagging issue

Check:

```bash
kubectl describe svc stable-grafana -n prometheus
```

---

## CrashLoopBackOff

Check logs:

```bash
kubectl logs <pod-name> -n prometheus
```

---

# Best Practices

* Use separate namespace for monitoring
* Enable persistent volumes for Prometheus
* Configure retention policies
* Secure Grafana with strong passwords
* Use Ingress + SSL in production
* Configure RBAC properly
* Backup Grafana dashboards

---

# Production Recommendations

| Component     | Recommendation                  |
| ------------- | ------------------------------- |
| Grafana       | Use Ingress + TLS               |
| Prometheus    | Use persistent storage          |
| Alertmanager  | Configure email/PagerDuty/Slack |
| Node Exporter | Run as DaemonSet                |
| Security      | Enable IAM roles & RBAC         |

---

# References

* Helm Charts
* Prometheus Documentation
* Grafana Documentation
* Kubernetes Documentation
* AWS EKS Documentation

---

# Conclusion

This setup provides:

* Complete Kubernetes monitoring
* Metrics collection
* Dashboard visualization
* Alerting capability
* CPU & memory monitoring
* Node and pod visibility
* PagerDuty integration testing

This monitoring stack is widely used in production Kubernetes environments.

---
