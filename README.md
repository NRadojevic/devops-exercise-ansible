# DevOps Exercise

For this exercise I used VMware Workstation as the virtualization solution.
I deployed a virtual machine based on Ubuntu 18.04.02 LTS with the following hardware settings (CPU: 2 | Memory: 4GB).
Within the VM I used Ansible to deploy Docker, Minikube, and the Jenkins and Registry container.
All you need to do is to clone this repository and execute "ansible-playbook site.yml".
You will get a prompt where you will have to enter the user who will run the tasks.
It is recommended that this user can execute commands using sudo without supplying a password. You do not need to distribute SSH keys to the target hosts as we are going to run the playbooks only on the local virtual machine.

The site.yml includes seven roles which are used to:
1) install the needed packages for docker and similar
2) install Docker-ce
3) install Minikube
4) install the Kubectl command line utility
5) deploy a Jenkins container on Minikube
6) generate Kubernetes credentials to access the Kubernetes cluster from Jenkins
7) deploy a Docker Registry on Minikube

After a successful ansible run you will have to configure Jenkins. In order to access the Jenkins UI from my laptop I set NAT forwarding in my Virtual Network settings.

![img](https://github.com/NRadojevic/devops-exercise-ansible/blob/master/pictures/workstation-nating.PNG)

The initial Jenkins AdminPassword can be retrieved from the Ansible outputs:

![img](https://github.com/NRadojevic/devops-exercise-ansible/blob/master/pictures/ansible-outcome.PNG)

In Jenkins I installed the following plugins:  
Docker Pipeline, Kubernetes Continuous Deploy Plugin, NodeJS Plugin, Pipeline.
I used the Docker Pipeline for building and pushing Docker images to my private registry. The Kubernetes Continous Deploy Plugin provides secrets configuration which I mainly used to deploy pods or services based on templates on Minikube. NodeJS Plugin made it possible to run npm commands which I needed to build my Angular application.

As it was not possible to create Git webhooks for https://github.com/NRadojevic/angular-hello-world due to the fact that Github only allows URLs or rather IP addresses reachable from the Internet, I found an alternative for the pipeline trigger.
I created a Multibranch Pipeline pointing to the repo mentioned above. Within this repo you can find a Jenkinsfile that only starts the build pipeline for the test/build/deployment of the Angular application. Additionally, I configured "Periodically if not otherwise run" interval to 2 minutes, so that the pipeline is triggered frequently if someone changes the source code.

![img](https://github.com/NRadojevic/devops-exercise-ansible/blob/master/pictures/multibranch-1.PNG)

![img](https://github.com/NRadojevic/devops-exercise-ansible/blob/master/pictures/multibranch-2.PNG)

That's how the pipeline code looks like:

```groovy
node {
    
    env.NODEJS_HOME = "${tool 'nodejs'}"
    env.PATH="${env.NODEJS_HOME}/bin:${env.PATH}"
    
    stage('Git clone') {
       git 'https://github.com/NRadojevic/angular-hello-world.git'
    }
    
    stage('Npm install') {
        sh 'npm install'
    }
    
    stage('Test and build app') {
        sh 'npm run test'
        sh 'npm run ng build --prod'
    }
    
    stage('Build and push Docker image') {
        docker.withRegistry('http://10.106.190.136:5000') {
        def customImage = docker.build("my-image:latest")
        customImage.push()
        }
    }
    
    stage('Deploy to k8s') {
        kubernetesDeploy(kubeconfigId: 'kubeConfig',
        configs: 'angular-app.yaml')
    }
}
```
The groovy file is located [here](https://github.com/NRadojevic/angular-hello-world/blob/master/pipelines/test-build-deploy.groovy/test-build-deploy.groovy) and configured in the following way in the Jenkins build pipeline:

![img](https://github.com/NRadojevic/devops-exercise-ansible/blob/master/pictures/test-build-pipeline-configuation.PNG)

As you can see I have deployed and used a non-secured private Docker registry. I tried to create a self-signed certificate passing the key and cert as enviroment variables in the registry container, but I got different errors, spent too much time on trying to fix that and therefore decided to skip that part. That's why I added the daemon.json file under /etc/docker/ on the host (https://github.com/NRadojevic/devops-exercise-ansible/blob/38636717173b5a23bdb7489c215420b2efc97f9a/roles/minikube-registry/tasks/main.yml#L51) and restarted the docker service at the end of the ansible run.

As already mentioned, I created a Kubernetes configuation with the "Kubernetes Continuous Deploy Plugin" for the access and interaction with the k8s cluster. The ansible role "minikube-angular-app" is used to prepare a kubernetes namespace for the test app and also returns a kubeconfig file I added directly in the kubeconfig stored in Jenkins credentials store:

![img](https://github.com/NRadojevic/devops-exercise-ansible/blob/master/pictures/jenkins-kubeconfig-configuration.PNG)

After the pipeline has successfully finished you should see the Angular app in your browser (assuming you have also exposed the nodePort [4200 -> 30002 in my case] in your service and configured port forwarding):

![img](https://github.com/NRadojevic/devops-exercise-ansible/blob/master/pictures/angular-app.PNG)

Due to lack of time I did not manage to deploy the containers as a Deployment object. That should be the correct choice if you want a blue-/green or at least a rolling deployment.

