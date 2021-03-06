
<img src="https://docs.microsoft.com/en-us/azure/architecture/example-scenario/apps/media/devops-with-aks/architecture-devops-with-aks.png">

## To Add into the pipeline:
* Build Automation
* Test Automation
* Security Checks (for e.g. WhiteHat scan)
* Deployment Automation (for e.g. Bamboo, Urbancode, scripting)
* Quality Checks (for e.g. SonarQube)
* Approval (for e.g. manual approval to deploy in UAT, Preprod or Prod environment)
* Notification, or Group Chat notification (for e.g. Slack, Hipchat).

## Deploy Java on Ubuntu 18.04
```
sudo apt-get install openjdk-8-jdk
sudo dpkg --purge --force-depends ca-certificates-java
sudo apt-get install ca-certificates-java
```
## Deploy Jenkins
```
sudo apt-get install jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
8888888844ce44e3bd49737688888888
# sudo ufw allow 8080
# sudo ufw enable
curl ifconfig.co
8.8.8.8
az vm open-port --port 8080 --resource-group u1804-rg --name u1804
open 8.8.8.8:8080
```
## Creating Jobs
```
New Project / Project Name : SampleBuildJob1 / Freestyle
Build Triggers / Build / Execute Shell
date
echo "Job built successfully."

New Project / Project Name : SampleDeployJob / Freestyle
Build Triggers / Build / Execute Shell
date
echo "Job built successfully."
Build after other projects are built / SampleBuildJob

New Project / Project Name : SampleTestJob / Freestyle
Build Triggers / Build / Execute Shell
date
echo "Job built successfully."
Build after other projects are built / SampleDeployJob
```

## Plugin : Delivery Pipeline
```
https://plugins.jenkins.io/delivery-pipeline-plugin
Manage Jenkins / Manage Plugins / Available / delivery

Download delivery-pipeline-plugin manually and upload the hpi file.  Workaround to plugin install hanging
https://updates.jenkins-ci.org/download/plugins/delivery-pipeline-plugin/

systemctl restart jenkins

Goto + on Main Page
Goto Delivery Pipeline View
Pipelines / Component / Add
- Name / BuildJob
- Initial Job / SampleBuildJob

Edit View /
Enable start of new pipeline build
Enable Trigger Rebuild - run a new job within the pipeline, but don't re-run the entire pipeline
Enable Show Total Build Time
Theme / Contrast
```
## Plugin : Build Pipeline
```
Manage Plugins / Build Pipeline
https://updates.jenkins-ci.org/download/plugins/build-pipeline-plugin/
View Name : BuildPipelineTest
Enable Build Pipeline View
Select Initial Job / SampleBuildJob
No of displayed Builds / 5
```
## CICD Demo using Jenkins with Kubernetes to roll out a container
```
$ git clone https://github.com/dazdaz/hellowhale
$ docker build . -t hellowhale
$ docker run -d -p80:80 -name hellowhale hellowhale
$ docker tag hellowhale dazdaz/hellowhale
$ docker login -u dazdaz -p mypass
$ docker push dazdaz/hellowhale

# Store our ACR login credentials into Kubernetes Secrets
$ kubectl create secret docker-registry acr-cred --docker-server=myacr.azurecr.io --docker-username=myacr --docker-password=<my_password> --docker-email=me@microsoft.com

# Apply this manifest to setup our app and configure the Azure LB
$ kubectl apply -f deployment-hellowhale-acr.yaml
```
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: hellowhale
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: hellowhale
    spec:
      containers:
      - name: hellowhale
        image: myacr.azurecr.io/dazdaz/hellowhale
        ports:
        - containerPort: 80
      imagePullSecrets:
       - name: acr-cred
---
apiVersion: v1
kind: Service
metadata:
  name: hellowhale-svc
spec:
  selector:
    app: hellowhale
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
```
```
# Add jenkins user to /etc/group as a secondary group so that he can build docker images
$ sudo usermod -G docker jenkins

# Copy Kubernetes keys to jenkins user, so that he can run kubectl commands against our cluster
$ sudo su - jenkins
$ sudo cp ~myuser/.kube/config ~jenkins/.kube/
$ chown jenkins:jenkins ~jenkins/.kube/config
$ cp ~myuser/.kube/config /var/lib/jenkins/.kube/
$ chown jenkins:jenkins /var/lib/jenkins/.kube/

################################################### J.E.N.K.I.N.S ###############################################
Configure System / Global Properties / Environment Variables
DOCKERHUB_PASS
mypassword

Jenkins Location / Sysadmin email
myuser@myhost.com

Check that this IP is really where your Jenkins server listens

GitHub / Settings / Integrations and Services
Select "Jenkins (GitHub plugib)" and add :
http://<Jenkins-IP>:8080/github-webhook/

Uncheck "Manage hook" if you don't have admin access or don't want to manage hooks from Jenkins.

Freestyle Project / hellowhale
GitHub Project / Project URL :
https://github.com/dazdaz/hellowhale.git

Source Code Management / Git / Repositories / Repository URL :
https://github.com/dazdaz/hellowhale.git

Enable GitHub hook trigger for GITScm polling

# Execute Shell - Choose ACR (here) or DockerHub (below) to store your images
# Jenkins Build Config using *ACR*
IMAGE_NAME="dazdaz/hellowhale:${BUILD_NUMBER}"
docker build . -t $IMAGE_NAME
docker tag dazdaz/hellowhale:${BUILD_NUMBER} myacr.azurecr.io/dazdaz/hellowhale:${BUILD_NUMBER}
docker login myacr.azurecr.io -u dazacr -p ${ACR_PASS}
docker push myacr.azurecr.io/dazdaz/hellowhale:${BUILD_NUMBER}

# Execute Shell - Choose ACR (above) or DockerHub (below) to store your images
# Jenkins Build Config using *Docker Hub*
IMAGE_NAME="dazdaz/hellowhale:${BUILD_NUMBER}"
docker build . -t $IMAGE_NAME
docker login -u dazdaz -p ${DOCKER_HUB}
docker push $IMAGE_NAME

# Execute Shell - Select ACR or DockerHub
# Set the new image tag on the deployment object, with container image stored on ACR
IMAGE_NAME="myacr.azurecr.io/dazdaz/hellowhale:${BUILD_NUMBER}"
kubectl set image deployments/hellowhale hellowhale=$IMAGE_NAME

# Set the new image tag on the deployment object, with container image stored on DockerHun
IMAGE_NAME="dazdaz/hellowhale:${BUILD_NUMBER}"
kubectl set image deployments/hellowhale hellowhale=$IMAGE_NAME

$ sudo service jenkins restart

Go back to GitHub repo / Settings Integrations / Your integration

Create a new file within the repo or update a file and then select Test Service
https://github.com/dazdaz/hellowhale/settings

# Then to demonstrate the CICD pipeline, modify the index.html and push the changes
$ git init
$ git config user.name "Derek"
$ git config user.email "derek@indeed.com"
$ git add html/index.html
$ git commit -am "Change message"
$ git push origin master

$ kubectl set image deployment <deployment> <container>=<image>
$ kubectl rollout history deployment/hellowhale
$ kubectl rollout undo deployment/hellowhale
```

## Resetting your Jenkins password
https://www.learnitguide.net/2018/08/reset-jenkins-admin-users-password.html

## Blue Ocean
* Blue Ocean is a new interface for Jenkins
* https://jenkins.io/projects/blueocean/
* https://jenkins.io/projects/blueocean/roadmap/ Blue Ocean is still a WIP, features are missing and the UI for me was very slow!
```
Manage Plugins / Blue Ocean
Open Blue Ocean (menu on the left)
```
# List of all Jenkins plugins
* https://updates.jenkins-ci.org/download/plugins/
* http://www.inanzzz.com/index.php/post/jnrg/running-jenkins-build-via-command-line

## Links
* https://stackoverflow.com/questions/tagged/jenkins
* https://www.digitalocean.com/community/tutorials/how-to-set-up-continuous-integration-pipelines-in-jenkins-on-ubuntu-16-04
* http://www.scmgalaxy.com/tutorials/complete-guide-to-use-jenkins-cli-command-line

## Random Jenkins tips
```
http://<Jenkins-IP>:8080/restart
```
