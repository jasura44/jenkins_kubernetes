# Setting minikube timezone to Singapore time
minikube start --driver=docker
minikube ssh
sudo ln -sf /usr/share/zoneinfo/Asia/Singapore /etc/localtime
sudo dpkg-reconfigure -f noninteractive tzdata   # For Debian/Ubuntu-based nodes; ignore if not present
date

minikube addons enable ingress

# Create jenkins namespace
kubectl apply -f namespace.yaml

# Change current namespace to jenkins
kubectl config set-context --current --namespace=jenkins

# Deploy PV
# PV before PVC: PVCs request storage from PVs; if PVs don’t exist, PVCs remain unbound.
kubectl apply -f volume.yaml

# Deploy service accounts & role & role bindings
kubectl apply -f serviceaccount.yaml

# Deploy deployment
# Pods using a ServiceAccount require the ServiceAccount and its permissions to be created beforehand to operate properly without permission errors.

kubectl get serviceaccount jenkins-service-account -o jsonpath='{.secrets[0].name}'

kubectl apply -f deployment.yaml

# Change ownership of jenkins volume directory
# This issue arises because when you use a hostPath volume in Minikube, the directory is often only writable by root, but Jenkins runs as a non-root user inside the container (commonly UID 1000)
minikube ssh
sudo chown -R 1000:1000 /data/jenkins-volume/

# Deploy service
kubectl apply -f service.yaml

# Deploy ingress
kubectl apply -f ingress.yaml

# Get Jenkins admin password
kubectl get pods
kubectl exec -it jenkins-77f78dc47f-xzzq2 -- cat /var/jenkins_home/secrets/initialAdminPassword

# Get password
29db579b19eb4cfb8792e15219716e8a

# Login to URL
minikube tunnel

# Login to console
- Add jenkins.com domain into /etc/hosts
- Login to jenkins.com 
- Install recommended plugins
- Set username, password and email address and save

------------------------------------------------------------------------------------------------

# Create custom kubeconfig file with base64 encoding

- Create a copy of the kubeconfig file at C:\Users\jasur\.kube and save it as config2
- Replace the following in config2
    - Server address to https://kubernetes.default.svc
    - Change certificate-authority to certificate-authority-data and replace with encoded data by running:
        - 
        $bytes = [System.IO.File]::ReadAllBytes("C:\Users\jasur\.minikube\ca.crt")
        $base64String = [Convert]::ToBase64String($bytes)
        Write-Output $base64String

    - Change client-certificate to client-certificate-data and replace with encoded data by running:
        - 
        $bytes = [System.IO.File]::ReadAllBytes("C:\Users\jasur\.minikube\profiles\minikube\client.crt")
        $base64String = [Convert]::ToBase64String($bytes)
        Write-Output $base64String

    - Change client-key to client-key-data and replace with encoded data by running:
        - 
        $bytes = [System.IO.File]::ReadAllBytes("C:\Users\jasur\.minikube\profiles\minikube\client.key")
        $base64String = [Convert]::ToBase64String($bytes)
        Write-Output $base64String

------------------------------------------------------------------------------------------------

# Installing on Kubernetes and Docker Plugin

- Go to Manage Jenkins > Plugins
- Under available plugins, search for kubernetes
- Check and install

- Go to Manage Jenkins > Plugins
- Under available plugins, search for Docker and Docker Pipeline
- Check and install
- Choose restart after installation

------------------------------------------------------------------------------------------------

# Create kubernetes cloud

- Go to Manage Jenkins > Clouds
- Create new cloud
- Give it a name like Kubernetes and choose Kubernetes
- Set Kubernetes URL to https://<minikube_ip>:8443 or https://kubernetes.default.svc
- Set namespace to Jenkins and test connection
- Change pod label: Key: jenkins, Value: agent
- Optional
    - Add credentials > Kind (Secret) > Choose Kubeconfig file > Set ID as kubeconfig 
- Click on Pod Templates and Add Pod Template
    - Name: jenkins-agent
    - Namespace: jenkins
    - Labels: jenkins-agent
    - Usage: Only build jobs with labels expressions matching this node
    - Name of container that will run the Jenkins agent: jnlp ??
    - Add container 1
        - Name: jnlp
        - Docker image: jenkins/inbound-agent:latest
        - Always pull image: Check
        - Working directory: /home/jenkins/agent
     - Add container 1
        - Name: docker
        - Docker image: docker:20-dind
        - Always pull image: Check
        - Working directory: /home/jenkins/agent   
    - Add container 1
        - Name: kubectl
        - Docker image: bitnami/kubectl:latest
        - Always pull image: Check
        - Working directory: /home/jenkins/agent
    - Add Volume Mount for Docker
        - Add Host Path Volume
        - Host Path: /var/run/docker.sock
        - Mount Path: /var/run/docker.sock   
    - Serviceaccount: jenkins-service-account
    - Click create

------------------------------------------------------------------------------------------------

# Create credentials

- Go to Manage Jenkins > Credentials > System Global Credentials
- Add credentials

    # For Docker
    - Kind: Username with password
    - Fill in username, password
    - Set an ID like dockerhubid
    - Save

    # For Kubeconfig
    - Kind: Secret file
    - Choose config2
    - Set an ID like kubeconfig
    - Save

------------------------------------------------------------------------------------------------

# Configuring on Agent (Manual Approach)

# Create and Configure the Agent Node
- Go to Manage Jenkins > Manage Nodes and Clouds.

- Click New Node, enter the agent name as jenkins-agent

- Choose Permanent Agent (fixed agent)

- Set no of executors to 3

- Set label to jenkins-agent

- In the agent configuration, set the Launch method to Launch agent by connecting it to the controller (JNLP).

- Set remote root directory to /home/jenkins/agent

- Save the configuration. Jenkins will generate a secret for this agent (the equivalent of ${computer.jnlpmac}). Use that in the yaml file before deploying deployment.yaml

- Save the secret into Kubernetes Secrets
    - Go to Manage Jenkins > Nodes
    - Go to the Nodes > Status 
    - Get the secret key
    - Encode secret key with Base 64
        - [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes("c9e2da59704b9903423b8bfea6c7c2399abf5e38e1711f79e90cf0f23ccb92e5"))
        - Inject secret into agent.yaml
        - kubectl apply -f agent-secret.yaml

------------------------------------------------------------------------------------------------

# Create pipeline

- Go to Jenkins Home
- Select New Item
- Set a name like Deploy-NodeJS-To-Minikube
- Select pipeline
- Create a description like This pipeline build a Node JS docker image, pushes to DockerHub and deploys the image into Minikube namespace called backend.
- Select discard old builds
    - Days to keep builds = 1
    - Max # of builds to keep = 1
- Select do not allow concurrent builds and check on abort previous builds
- Under Pipeline, choose Pipeline script from SCM
    - SCM = GIT
    - Repository URL = https://github.com/jasura44/nodejs_minikube.git
    - Branch to build = master
    - Scrip Path = Jenkinsfile
- Save and exit

------------------------------------------------------------------------------------------------

## Utilities

# To view container logs inside a pod
kubectl logs jenkins-agent-1xpj8 -c kubectl -n jenkins

# To go into kubectl container inside pod and run shell commands
kubectl exec -it -c kubectl jenkins-agent-lk1s4 -- sh
kubectl get pods

# To view jenkins agent logs
kubectl exec -it -c docker jenkins-agent-pdhpw -- sh

# To get pod status
kubectl get pod <pod_name> -o wide

# To get pods with labels
kubectl get pods --show-labels

# To watch all pods
kubectl get -A pods --watch

# To show kubernetes events
kubectl get events --watch

# To describe pods
kubectl describe pod <pod_name>

# To increase minikube resources
minikube stop
minikube start --cpus=4 --memory=8192

# To determine the OS of a running container within a pod
kubectl exec --stdin --tty <pod_name> -- /bin/bash -c "cat /etc/os-release"

# To inject secret and name
kubectl exec -it jenkins-agent-6cb9d895d-5kltj -- curl -sO http://jenkins.com/jnlpJars/agent.jar
kubectl exec -it jenkins-agent-6cb9d895d-5kltj -- curl -sO java -jar agent.jar -url http://jenkins.com/ -secret bf5fa486407ee0f0341bd56d633c415b4f9310ae8cfdf7d08cc96a9df03f677f -name "jenkins-agent" -webSocket -workDir "/var/jenkins"

# To check if kube system is running
kubectl get pods -n kube-system

# To get verbose logs
kubectl version --v=9






