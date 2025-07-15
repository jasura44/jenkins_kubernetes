# Set alias for kubectl commands
Set-Alias k "minikube kubectl --"

# jenkins_kubernetes
Set up a Jenkins cluster on minikube

# View cluster info
kubectl config view --minify | grep server

# To encode certs to base64 to replace onto kubeconfig file (Since the cert content cannot be read from the jenkins namespace)
powershell "[Convert]::ToBase64String([System.IO.File]::ReadAllBytes('ca.crt'))"
powershell "[Convert]::ToBase64String([System.IO.File]::ReadAllBytes('client.crt'))"
powershell "[Convert]::ToBase64String([System.IO.File]::ReadAllBytes('client.key'))"

# Pre-requisites
1. Install minikube
2. Install kubectl
3. Install Helm (Package manager / Template engine) >> choco install kubernetes-helm

# Create a Jenkins service account
Refer to jenkins_serviceaccount.yaml

# Create a cluster role with permissions across namespaces
Refer to jenkins_namespace.yaml

# Create the role binding
Refer to jenkins_clusterrole.yaml

# To check if kubernetes port is running
netstat -an | findstr 50463

# Create service account in Jenkins Namespace
Refer to jenkins_rolebinding.yaml

# (Optional) Create a Persistent Volume: You can create a persistent volume to store Jenkins data:
kubectl apply -f jenkins_volume.yaml >> Set storage class name like jenkins-sc for both pv and pvc to bind

# Option A Deploy jenkins with HELM
helm repo add jenkins-stable https://charts.jenkins.io
helm repo update
helm install jenkins jenkins-stable/jenkins -n jenkins

# Option B Deploy jenkins with Deployment.YAML
kubectl apply -f jenkins-deployment.yaml -n jenkins

# Get your 'admin' user password
kubectl exec -it jenkins-5688964df5-klg9z -n jenkins -- cat /var/jenkins_home/secrets/initialAdminPassword
Password: 8810da916bc347efae4e3af7d309347b

# To run curl command inside pod
kubectl exec -it jenkins-5688964df5-klg9z -n jenkins -- curl https://127.0.0.1:50463

# Forward the Jenkins port
kubectl port-forward svc/jenkins 8080:8080 --namespace jenkins

# Get minikube IP
minikube ip

# Add line to your local /etc/hosts (Linux/Mac) or C:\Windows\System32\drivers\etc\hosts (Windows):
127.0.0.1 jenkins.local

# Access Jenkins
kubectl get svc -n jenkins

# Install kubernetes plugin
Install kubernetes plugin under plugins in Jenkins

# Install docker plugin
Install docker and docker pipeline plugin

# Configure cloud
Go to "Manage Jenkins" -> "Manage Nodes and Clouds" -> "Configure Clouds" and add a new Kubernetes cloud
Add credentials > Secret file > Choose kubeconfig file (C:\Users\username\.minikube\config)
Give a name > mykubeconfig

# Docker as a plugin
Go to Dashboard > Manage Jenkins > Credentials > System > Global credentials (unrestricted)
Add credentials (Username, password and set an unique ID for reference in Jenkinsfile) -.e.g dockercredentailsid

# Define agent templates
Within the Jenkins configuration, define templates for the Kubernetes Pods that will be created as agents. This includes the Docker image, required environment variables, and other settings. 

# Check services and pods
kubectl get service -n jenkins
kubectl get pods -n jenkins

# Create a test pipeline
Refer to Jenkinsfile

# Userful resources
Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http://127.0.0.1:8080/configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos


