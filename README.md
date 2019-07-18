# devops-exercise

For this exercise I used VMware Workstation as the virtualization solution.
I deployed a virtual machine based on Ubuntu Server 18.04 with the following hardware settings (CPU: 2 | Memory: 4GB).
Within the VM I used Ansible to deploy Docker, Minikube, and the Jenkins and Registry container.
All you need to do is to clone this repository and execute "ansible-playbook site.yml".
You will get a prompt where you will have to enter the user who will run the tasks.
It is recommended that this user can execute commands using sudo without supplying a password. You do not need to distribute SSH to the target hosts as we are going to run the playbooks only on localhost.

The site.yml includes seven roles which are used to:
1) install the needed packages for docker and similar
2) install Docker-ce
3) install Minikube
4) install the Kubectl command line utility
5) deploy a Jenkins container on Minikube
6) generate Kubernetes credentials to access the Kubernetes cluster from Jenkins
7) deploy a Docker Registry on Minikube

After a successful ansible run you will have to configure Jenkins.
