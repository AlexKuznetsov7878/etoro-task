## Overview

This repository contains the Helm chart and Jenkins pipeline configurations for deploying a simple web application to a Kubernetes cluster. The deployment leverages **KEDA** (Kubernetes Event-driven Autoscaling) to manage autoscaling based on a **cron schedule**, ensuring the application scales up and down at specified times. This setup addresses permission constraints within the Kubernetes cluster by configuring appropriate **RBAC** (Role-Based Access Control) permissions.

---


## Repository Structure

etoro-task/ 
├── charts/ 
│   └── simple-web/ 
│       ├── Chart.yaml 
│       ├── values.yaml 
│       └── templates/ 
│           ├── deployment.yaml 
│           ├── service.yaml 
│           ├── ingress.yaml 
│           └── scaledobject.yaml 
├── Jenkinsfile 
└── README.md

- **Chart.yaml**: Defines the Helm chart metadata.
- **values.yaml**: Contains configurable values for the Helm chart.
- **templates/**: Directory containing Kubernetes resource templates.
- **Jenkinsfile**: Defines the Jenkins pipeline for CI/CD.
- **README.md**: Documentation for the project.

---

## Deployment Steps

Follow these steps to deploy the simple web application with Helm, Jenkins, and KEDA.

Ensure your `values.yaml` includes the **KEDA** configuration with only the **cron trigger**. Remove any resource-based triggers to align with permission constraints.
I wasn't able to solve the next issue: Error from server (Forbidden): pods.metrics.k8s.io is forbidden: User "3d2063a2-b545-4c3c-8621-839a3c6442e5" cannot list resource "pods" in API group "metrics.k8s.io" in the namespace "alex"


replicaCount: 1

image:
  repository: acrinterview.azurecr.io/simple-web
  pullPolicy: IfNotPresent
  tag: "latest"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  automount: true
  annotations: {}
  name: ""

podAnnotations: {}
podLabels: {}

podSecurityContext: {}
securityContext: {}

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  hosts:
    - host: "alex-app.local"  # Dummy domain name
      paths:
        - path: /alex
          pathType: Prefix
  tls: []

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"

livenessProbe:
  httpGet:
    path: /
    port: http
readinessProbe:
  httpGet:
    path: /
    port: http

autoscaling:
  enabled: false

keda:
  enabled: true
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: cron
      metadata:
        timezone: "UTC"
        start: "0 8 * * *"       # Scale up at 08:00 UTC
        end: "0 18 * * *"        # Scale down at 18:00 UTC
        desiredReplicas: "3"

volumes: []
volumeMounts: []

nodeSelector: {}
tolerations: []
affinity: {}

Ensure that the scaledobject.yaml template correctly references the keda values.

# charts/simple-web/templates/scaledobject.yaml

{{- if .Values.keda.enabled }}
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: {{ include "simple-web.fullname" . }}
  namespace: alex
  labels:
    {{- include "simple-web.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    name: {{ include "simple-web.fullname" . }}
  minReplicaCount: {{ .Values.keda.minReplicaCount }}
  maxReplicaCount: {{ .Values.keda.maxReplicaCount }}
  triggers:
    {{- toYaml .Values.keda.triggers | nindent 6 }}
{{- end }}


Ensure that the ingress.yaml template correctly updated.
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "simple-web.fullname" . }}
  namespace: alex
  labels:
    {{- include "simple-web.labels" . | nindent 4 }}
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: {{ (index .Values.ingress.hosts 0).host | quote }}
      http:
        paths:
          - path: {{ (index (index .Values.ingress.hosts 0).paths 0).path }}
            pathType: {{ (index (index .Values.ingress.hosts 0).paths 0).pathType }}
            backend:
              service:
                name: {{ include "simple-web.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}


Ensure that the Jenkinsfile is correctly set up to handle deployment and destruction actions. Here's is the used Jenkinsfile. 
The username for Jenkins is: admin
The password for Jenkins is: password@1234

// Jenkinsfile

pipeline {
    agent any
    parameters {
        choice(name: 'ACTION', choices: ['deploy', 'destroy'], description: 'Choose deploy to deploy or destroy to delete the release')
    }
    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }
        stage('Set Up Kubernetes Context') {
            steps {
                sh '''
                az login --identity
                az aks get-credentials --resource-group devops-interview-rg --name devops-interview-aks --overwrite-existing --file /var/lib/jenkins/.kube/config
                kubelogin convert-kubeconfig -l msi --kubeconfig /var/lib/jenkins/.kube/config
                '''
            }
        }
        stage('Helm Action') {
            steps {
                script {
                    if (params.ACTION == 'deploy') {
                        sh 'helm upgrade --install simple-web ./charts/simple-web -n alex'
                    } else if (params.ACTION == 'destroy') {
                        sh 'helm uninstall simple-web -n alex'
                    }
                }
            }
        }
        stage('Post-Deployment Verification') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                script {
                    // Simple health check using curl
                    def status = sh(script: 'curl -o /dev/null -s -w "%{http_code}" http://alex-app.local/alex', returnStdout: true).trim()
                    if (status != '200') {
                        error "Health check failed with status code ${status}"
                    } else {
                        echo "Health check passed with status code ${status}"
                    }
                }
            }
        }
    }
    post {
        failure {## Overview

This repository contains the Helm chart and Jenkins pipeline configurations for deploying a simple web application to a Kubernetes cluster. The deployment leverages **KEDA** (Kubernetes Event-driven Autoscaling) to manage autoscaling based on a **cron schedule**, ensuring the application scales up and down at specified times. This setup addresses permission constraints within the Kubernetes cluster by configuring appropriate **RBAC** (Role-Based Access Control) permissions.

---


## Repository Structure

etoro-task/ 
├── charts/ 
│   └── simple-web/ 
│       ├── Chart.yaml 
│       ├── values.yaml 
│       └── templates/ 
│           ├── deployment.yaml 
│           ├── service.yaml 
│           ├── ingress.yaml 
│           └── scaledobject.yaml 
├── Jenkinsfile 
└── README.md

- **Chart.yaml**: Defines the Helm chart metadata.
- **values.yaml**: Contains configurable values for the Helm chart.
- **templates/**: Directory containing Kubernetes resource templates.
- **Jenkinsfile**: Defines the Jenkins pipeline for CI/CD.
- **README.md**: Documentation for the project.

---

## Deployment Steps

Follow these steps to deploy the simple web application with Helm, Jenkins, and KEDA.

Ensure your `values.yaml` includes the **KEDA** configuration with only the **cron trigger**. Remove any resource-based triggers to align with permission constraints.
I wasn't able to solve the next issue: Error from server (Forbidden): pods.metrics.k8s.io is forbidden: User "3d2063a2-b545-4c3c-8621-839a3c6442e5" cannot list resource "pods" in API group "metrics.k8s.io" in the namespace "alex"


replicaCount: 1

image:
  repository: acrinterview.azurecr.io/simple-web
  pullPolicy: IfNotPresent
  tag: "latest"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  automount: true
  annotations: {}
  name: ""

podAnnotations: {}
podLabels: {}

podSecurityContext: {}
securityContext: {}

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  hosts:
    - host: "alex-app.local"  # Dummy domain name
      paths:
        - path: /alex
          pathType: Prefix
  tls: []

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"

livenessProbe:
  httpGet:
    path: /
    port: http
readinessProbe:
  httpGet:
    path: /
    port: http

autoscaling:
  enabled: false

keda:
  enabled: true
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: cron
      metadata:
        timezone: "UTC"
        start: "0 8 * * *"       # Scale up at 08:00 UTC
        end: "0 18 * * *"        # Scale down at 18:00 UTC
        desiredReplicas: "3"

volumes: []
volumeMounts: []

nodeSelector: {}
tolerations: []
affinity: {}

Ensure that the scaledobject.yaml template correctly references the keda values.

# charts/simple-web/templates/scaledobject.yaml

{{- if .Values.keda.enabled }}
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: {{ include "simple-web.fullname" . }}
  namespace: alex
  labels:
    {{- include "simple-web.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    name: {{ include "simple-web.fullname" . }}
  minReplicaCount: {{ .Values.keda.minReplicaCount }}
  maxReplicaCount: {{ .Values.keda.maxReplicaCount }}
  triggers:
    {{- toYaml .Values.keda.triggers | nindent 6 }}
{{- end }}


Ensure that the ingress.yaml template correctly updated.
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "simple-web.fullname" . }}
  namespace: alex
  labels:
    {{- include "simple-web.labels" . | nindent 4 }}
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: {{ (index .Values.ingress.hosts 0).host | quote }}
      http:
        paths:
          - path: {{ (index (index .Values.ingress.hosts 0).paths 0).path }}
            pathType: {{ (index (index .Values.ingress.hosts 0).paths 0).pathType }}
            backend:
              service:
                name: {{ include "simple-web.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}


Ensure that the Jenkinsfile is correctly set up to handle deployment and destruction actions. Here's is the used Jenkinsfile. 
The username for Jenkins is: admin
The password for Jenkins is: password@1234

// Jenkinsfile

pipeline {
    agent any
    parameters {
        choice(name: 'ACTION', choices: ['deploy', 'destroy'], description: 'Choose deploy to deploy or destroy to delete the release')
    }
    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }
        stage('Set Up Kubernetes Context') {
            steps {
                sh '''
                az login --identity
                az aks get-credentials --resource-group devops-interview-rg --name devops-interview-aks --overwrite-existing --file /var/lib/jenkins/.kube/config
                kubelogin convert-kubeconfig -l msi --kubeconfig /var/lib/jenkins/.kube/config
                '''
            }
        }
        stage('Helm Action') {
            steps {
                script {
                    if (params.ACTION == 'deploy') {
                        sh 'helm upgrade --install simple-web ./charts/simple-web -n alex'
                    } else if (params.ACTION == 'destroy') {
                        sh 'helm uninstall simple-web -n alex'
                    }
                }
            }
        }
        stage('Post-Deployment Verification') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                script {
                    // Simple health check using curl
                    def status = sh(script: 'curl -o /dev/null -s -w "%{http_code}" http://alex-app.local/alex', returnStdout: true).trim()
                    if (status != '200') {
                        error "Health check failed with status code ${status}"
                    } else {
                        echo "Health check passed with status code ${status}"
                    }
                }
            }
        }
    }
    post {
        failure {
            echo 'Pipeline failed.'
        }
        success {
            echo 'Pipeline succeeded.'
        }
    }
}

Testing and Verification
After deploying the application, perform the following tests to ensure everything is functioning correctly.
1. Verify Deployment Status
kubectl get all -n alex
Check:
Pods: All pods should be in the Running state.
Services: Services should have the correct type and ports.
Deployments: Deployments should be healthy with the desired number of replicas.
ReplicaSets: Should reflect the correct number of replicas.
HPA: Should list only the cron trigger without resource-based triggers.

Test Application Accessibility

From your VM, run:
curl http://alex-app.local/alex
The configuration is under /etc/hosts

            echo 'Pipeline failed.'
        }
        success {
            echo 'Pipeline succeeded.'
        }
    }
}

Testing and Verification
After deploying the application, perform the following tests to ensure everything is functioning correctly.
1. Verify Deployment Status
kubectl get all -n alex
Check:
Pods: All pods should be in the Running state.
Services: Services should have the correct type and ports.
Deployments: Deployments should be healthy with the desired number of replicas.
ReplicaSets: Should reflect the correct number of replicas.
HPA: Should list only the cron trigger without resource-based triggers.

Test Application Accessibility

From your VM, run:
curl http://alex-app.local/alex
The configuration is under /etc/hosts
