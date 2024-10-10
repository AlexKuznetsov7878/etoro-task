pipeline {
    agent any
    environment {
        PATH = "/usr/local/bin:/usr/bin:/bin:${env.PATH}"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"
    }
    parameters {
        choice(name: 'ACTION', choices: ['deploy', 'destroy'], description: 'Choose action')
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Set Up Kubernetes Context') {
            steps {
                sh '''
                az login --identity
                az aks get-credentials --resource-group devops-interview-rg --name devops-interview-aks --overwrite-existing --file $KUBECONFIG
                kubelogin convert-kubeconfig -l msi --kubeconfig $KUBECONFIG
                '''
            }
        }
        stage('Helm Deploy') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                sh 'helm upgrade --install simple-web ./ -n alex'
            }
        }
        stage('Helm Destroy') {
            when {
                expression { params.ACTION == 'destroy' }
            }
            steps {
                sh 'helm uninstall simple-web -n alex'
            }
        }
    }
}

