## Overview

This repository contains the Helm chart and Jenkins pipeline configurations for deploying a simple web application to a Kubernetes cluster. The deployment leverages **KEDA** (Kubernetes Event-driven Autoscaling) to manage autoscaling based on a **cron schedule**, ensuring the application scales up and down at specified times. This setup addresses permission constraints within the Kubernetes cluster by configuring appropriate **RBAC** (Role-Based Access Control) permissions.

## Repository Structure  

- **Chart.yaml**: Defines the Helm chart metadata.  
- **values.yaml**: Contains configurable values for the Helm chart.  
- **templates/**: Directory containing Kubernetes resource templates.  
- **Jenkinsfile**: Defines the Jenkins pipeline for CI/CD.  
- **README.md**: Documentation for the project.  

## Deployment Steps  

Follow these steps to deploy the simple web application with Helm, Jenkins, and KEDA.  

Ensure your `values.yaml` includes the **KEDA** configuration with only the **cron trigger**. Remove any resource-based triggers to align with permission constraints.  
I wasn't able to solve the next issue: Error from server (Forbidden): pods.metrics.k8s.io is forbidden: User "3d2063a2-b545-4c3c-8621-839a3c6442e5" cannot list resource "pods" in API group "metrics.k8s.io" in the namespace "alex"  
Ensure that the scaledobject.yaml template correctly references the keda values.  
Ensure that the ingress.yaml template correctly updated.  
Ensure that the deployment.yaml template correctly updated.  
Ensure that the Jenkinsfile is correctly set up to handle deployment and destruction actions.  
The username for Jenkins is: admin  
The password for Jenkins is: password@1234  

## Testing and Verification  

After deploying the application, perform the following tests to ensure everything is functioning correctly.  
1.Verify Deployment Status  
  kubectl get all -n alex  
2.Check:  
  Pods: All pods should be in the Running state.  
  Services: Services should have the correct type and ports.  
  Deployments: Deployments should be healthy with the desired number of replicas.  
  ReplicaSets: Should reflect the correct number of replicas.  
  HPA: Should list only the cron trigger without resource-based triggers.  

## Test Application Accessibility

From your VM, run:  
curl http://alex-app.local/alex  
The configuration is under /etc/hosts  
