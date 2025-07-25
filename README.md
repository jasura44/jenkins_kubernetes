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
kubectl apply -f deployment.yaml

# Change ownership of jenkins volume directory
# This issue arises because when you use a hostPath volume in Minikube, the directory is often only writable by root, but Jenkins runs as a non-root user inside the container (commonly UID 1000)
minikube ssh
sudo chown -R 1000:1000 /mnt/jenkins-data

# Deploy service
kubectl apply -f service.yaml

# Deploy ingress
kubectl apply -f ingress.yaml

# Get Jenkins admin password
kubectl exec -it jenkins-65748b8b68-829tw -- cat /var/jenkins_home/secrets/initialAdminPassword

# Get password
d5f6c6d2fa0143b499bcc472edf620ce

------------------------------------------------------------------------------------------------
# Configuring on Agent on Jenkins UI

# Create and Configure the Agent Node
- Go to Manage Jenkins > Manage Nodes and Clouds.

- Click New Node, enter the agent name as jenkins-agent

- Choose Permanent Agent (fixed agent)

- Set no of executors to 3

- Set label to jenkins-agent

- In the agent configuration, set the Launch method to Launch agent by connecting it to the controller (JNLP).

- Set remote root directory to /home/jenkins/agent

- Save the configuration. Jenkins will generate a secret for this agent (the equivalent of ${computer.jnlpmac}). Use that in the yaml file before deploying deployment.yaml

------------------------------------------------------------------------------------------------

# Installing on Kubernetes and Docker Plugin

- Go to Manage Jenkins > Plugins
- Under available plugins, search for kubernetes
- Check and install

- Go to Manage Jenkins > Plugins
- Under available plugins, search for Docker
- Check and install
- Choose restart after installation

------------------------------------------------------------------------------------------------

# Create kubernetes cloud

- Go to Manage Jenkins > Clouds
- Create new cloud
- Give it a name like Minikube and choose Kubernetes
- Set Kubernetes URL to https://<minikube_ip>:8443
- Create a copy of the kubeconfig file at C:\Users\jasur\.kube >> config2
- Replace the following in config 2
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
    - Server address to https://kubernetes.default.svc
    - Set namespace to Jenkins and test connection
    - Save and exit

------------------------------------------------------------------------------------------------

# Create credentials

- Go to Manage Jenkins > Credentials > System Global Credentials
- Add credentials

    # For Docker
    - Kind: Username with password
    - Fill in username, password
    - Set an ID like dockerid
    - Save

    # For Kubeconfig
    - Kind: Secret file
    - Choose config2
    - Set an ID like kubeconfig
    - Save

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
- Select do not allow concurrent builds
- Select GitHub Project and state the project name like https://github.com/jasura44/nodejs_minikube.git
- Under Pipeline, choose Pipeline script from SCM
    - SCM = GIT
    - Repository URL = https://github.com/jasura44/nodejs_minikube.git
    - Branch to build = master
    - Scrip Path = Jenkinsfile
- Save and exit


------------------------------------------------------------------------------------------------

## Utilities

# To get pod status
kubectl get pod <pod_name> -o wide

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






