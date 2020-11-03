# CD the Gitops way to k8s
In this lab we will need a local git and argocd (and of course k8s installed)
we will create a basic deployment of an application and then will connect it to argocd and see the changes we can do.

## Requirements
Recommendation: This lab is taking a lot of resources from your computer, so it's prefered to run it on a cloud.
This lab assumes that you are running on a linux machine.
Please install theese softwares on your machine/workstation:
* Docker
* Kubernetes

## Environment
This is your working directory for this lab (Labs/gitops-cd-k8s).  
Don't forget to clone this repository !

## Bitbucket
Run on your machine: `kubectl apply -f deploy/bitbucket/`  
With that command, you deployed a bitbucket instance to your environment:
* Bitbucket server (StatefulSet)
* Bitbucket volume (PV)
* Bitbucket service (Service of type NodePort)

Run: `kubectl get svc bitbucket-svc -o json | jq .spec.ports[0].nodePort`  
This will show you your port for outside-cluster communication!  
> kubectl get svc bitbucket-svc -o json | jq .spec.ports[0].nodePort
> 31775

Go on your browser to: http://localhost:<YourNodePort>/setup  
You can also test your api respose:  
> curl -X GET "http://localhost:31775/setup"  
You should see the bitbucket's setup page.  Do the quick setup with the internal DB and no jira integration.  
It shoukd ask you for a license, you can generate it easily (just choose server license when you'll need to).  
Log in to bitbucket and that should be enough for it.

## ArgoCD
Go to: https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml  
You should see a manifest of the argocd installation.  
Run on your local machine: `wget -O deploy/argocd/install-argocd.yaml https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`
* warning: You should NEVER run a yaml from the web right into your system (It can be a virus...), so first download it and see what's inside.  
Create a namespace for argocd to run in: `kubectl create namespace argocd`
Run on your local machine: `kubectl apply -n argocd -f deploy/argocd/install-argocd.yaml`  
By default, the Argo CD API server is not exposed with an external IP. To access the API server, we'll use Port forwarding.  
Run: `kubectl port-forward svc/argocd-server -n argocd 8080:443`


## Docker Hub
Please create a docker hub account at: https://hub.docker.com/

### Docker Hub Repository
Create a docker repository named "nodejs-app-docker".  
NOTE: Don't touch the build settings. (Docker can create automated builds to your image by itself)

## Jenkins
Attached there is a Dockerfile. It will be your jenkins (also docker installed inside)
Run this command:  
    `docker build . -t devops-culture/jenkins-docker:1`  

Ensure that the docker image created:  
    `docker image ls`  

In your machine - please create the following directory:  
    `mkdir /tmp/jenkins_home`  
* IMPORTANT: This is your jenkins data so if you preffer to save it somewhere else please edit the docker-compose too.

Attached there is a docker-compose.yml file. It will set up your Jenkins server.  
On your own:
* Look at the compose file and see whats inside.
* Find the address that jenkins will listen to.  

run in the working directory:  
    `docker-compose up -d`  
View the generated administrator password to log in the first time:  
    `docker exec jenkins-docker cat /var/jenkins_home/secrets/initialAdminPassword`  
Enter jenkins and log in.

![Jenkins First Login](pictures/jenkins-first-login.png)

Enter the password and continue.  
In next screen, click “Install suggested plugins” and wait for them to install.  
In the next section you need to create a user for yourself.

![Jenkins first user admin](pictures/create-first-user-admin.png)

Click "get start using Jenkins"  
Now we need to install two plugins:
* blueocean plugin to beautify your jenkins. This plugin better the usability of jenkins and focus in pipelines scripts.
* locale plugin to jenkins always show english text. You don’t need install this, but i suggest install to search errors directly in english.

Go to Plugin Manager again and search for locale.

To install plugins:
- In home of jenkins, click in Manage Jenkins:

![Jenkins Menu](pictures/jenkins-menu.png)

- Click manage plugins:

![Jenkins Menu Manage Plugins](pictures/jenkins-manage-plugins.png)

In Plugin manager, you need click in Available tab and after load the tab, search the plugins:

![Jenkins Plugin Manager](pictures/plugin-manager.png)

Search for locale, mark the checkbox and click in install button:

![Locale Installation](pictures/locale-install.png)

After install locale, go to Home > Manage Jenkins > Configure System.

![Configure System](pictures/jenkins-configure-system.png)

Search for locale. Input “en” in “Default language” and check “ignore browser preference and force this language to all users”:

![Choose Locale Language](pictures/locale-lang.png)

Locale Plugin installed and configured, lets install Blue ocean.  
Back to jenkins home > Manage jenkins >Manage Plugins. In Available tab search for blueocean and install it.

![Blue Ocean Installation](pictures/blueocean-install.png)

All done. But you see the visual without blue ocean plugin, let’s use it.  
In Jenkins home, Click in “Open Blue Ocean” (left menu):

![Choose Blue Ocean](pictures/jenkins-menu-blueocean.png)

Now, you will see a screen like this:

![Blue Ocean View](pictures/blue-ocean-jenkins-view.png)

All Right, now we have jenkins installed and we are using blue ocean plugin to have a better experience in your pipelines.

### Docker Hub Credential
Go to jenkins home, click on "Manage Jenkins" and “Manage Credentials”, finaly, click "global".

![Jenkins Credentials Settings](pictures/jenkins-cred.png)

Click on “Add Credentials” in left menu, put your credential and save it.

![Add Credentials](pictures/jenkins-create-credentials.png)

* IMPORTANT: If you didn't called your credential 'dockerhub' you should change it also in the Jenkinsfile.

Great!  
The next step is to create a pipeline to build some things for us.

## Jenkinsfile
Attached is a Jenkinsfile for this lab, please read the explanation inside this file.  
* IMPORTANT: For this to work, in the Jenkinsfile, please replace the docker_hub_account with your account.

## Creating a job to test docker command
Go back to the blue ocean view.
Click on "Create new pipeline"
Fill the form:
* Choose GitHub
* Give an access token which you can create via the "create access token" link
* Choose user user
* Choose the nodejs-app-docker repository
Click "Create pipeline"

![Choose GitHub](pictures/create-pipeline.png)

## Building the first docker image 
After saving this pipeline, Jenkins will start running it's first run.
It suppose to succeed:

![Run Succeeded](pictures/jenkins-succeed-run.png)

You can go to your docker hub repository and look at the image pushed:

![Image Pushed](pictures/image-pushed.png)

Now pull the image from your docker repository:  
    `docker pull docker_account/nodejs-app-docker:1`  
Run the docker app:  
    `docker run -p 3000:3000 docker_account/nodejs-app-docker:1 nodejs-app`

### Clean your lab
1. From the working directory, stop the compose:  
    `docker-compose down -v`
1. In case you don't want to save your jenkins data, remove /tmp/jenkins_home from your machine:  
    `rm -rf /tmp/jenkins_home`  
1. Clear ddclient config:
    `rm -rf /tmp/ddclient-config`

## Biblography
This Lab is based on these tutorials:  
* https://medium.com/@gustavo.guss/quick-tutorial-of-jenkins-b99d5f5889f2
* https://medium.com/@gustavo.guss/jenkins-building-docker-image-and-sending-to-registry-64b84ea45ee9
