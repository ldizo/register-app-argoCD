# register-app-argoCD
In this project, we shall build acomplete CI/CD Pipeline using Jenkins, then we integrate the Application SCM GitHub Repository, Maven, Docker, EKS Cluster, Trivy, ArgoCD Slack and the 2nd GitHub repository (which host those Manifest Files) to the Jenkins Pipeline.
Such that, when a Developer makes any changes to the Application source code and pushes it to the GitHub Application Source code Repository, it automatically triggers the Continuous Integration (CI) Jenkins Job. This will immediately pull the Application code from that GitHub Repository and the build process (by Maven) will automatically get initiated. Then Static Code Analysis (by SonarQube) will proceed to perform quality test on the Application. In the Jenkinsfile, it instructs Jenkins to futher package and build the image using the Dockerfile and further push it to our defined DockerHub Repository. After that this created Docker Image will go through a scanning process by Trivy-scan to eliminate any vulnerabilities. This action ends the CI Job.
When everything is automated, the Continuous Deployment (CD) Job still defined in Jenkins as the 2nd Job get trigerred immediately. It will first of all update the the Build Number (the Release Number) in the Deployment Yaml File within the 2nd GitHub Repository that is hosting all the Kubernetes Manifest Files. From that 2nd GitHub Repository, ArgoCD which is installed in the K8s cluster will automaticall pull the changes noticed in that 2nd Repo and it will automatically deploy the resources inside the EKS Cluster. 
And once that CD Job is completed, it will send a notification on Slack to the entire team or it can send an email to team members concerned.

![Screenshot 2024-11-12 at 9 35 34 PM](https://github.com/user-attachments/assets/5b413388-42ba-419a-9f86-603777520b95)

                                           -- Implementation --
We shall use the Jenkins Master-Client Architecture, so that the main Jenkins server will not be overloaded.
  (1) Install and configure the Jenkins-Master server and the Jenkins-Agent server
Create a Jenkins Master Server in the Console. so
- Locate and click on "Launch Instance"
- Name: **Jenkins-master**
- OS image: **Ubuntu**
- Amazon Machine Image (AMI): **Ubuntu server 22.04 LTS**
- Architecture: **64-bit(x86)**
- Keypair
  . click on "create new keypair"
  . Keypair name: **Jenkins-vm-keypair**
  . click on "create" to create the Keypair
- Configure storage (Hard Disk): **15 GB**
- Now, click on "Launch Instance" to create the instance
- Now, copy the Public Ip of this Jenkins-Master Vm and ssh into it from your local system
- You should have it as:
- 
ubuntu@ip-172-31-0-62:~$
- 
- Now, first of all update the system.
***sudo apt update***
- Then, proceed to upgrade the system.
***sudoapt upgrade***
- Open and rename the Hostname here.
***vi /etc/hostname***
- Now, Erase everything in there and type this
***Jenkins-Master***
- Now, save and quite.
***:wq!***
- Now, reboot the system.
***sudo init 6***
  
- Now, go to the console and open the Firewall Rule.
- So, go to "Security Group"
- click on "create a Security Group"
- Click on "Edit inbound rules"
- Click on "Add rule"
- Type: ***Custom***, TCP Protocol: ***TCP***, Port range: ***8080***, Source: ***Anywhere ipv4***
- Then click on "create" to create the Firewall Rule

- Proceed to install Java on this Jenkins VM. Use this command to install java.
***sudo apt install openjdk-17-jre***
- Now, check to confirm that Java is succesfully installed.
***java -version***
  
- It should show you the version of openjdk thats running in the system
  
- Now, proceed to install Jenkins. You can go to the Jenkins Documentation page for Ubuntu/Debian and copy the code under "Weekly release"
  ***sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins***

- now, Enable the Jenkins service to start at boot.
***sudo systemctl enable jenkins***
- Then, Start Jenkins as a service.
***sudo systemctl start jenkins***
- Now, Check the status of jenkins to ensure that its running and Active.
***systemctl status jenkins***
  It should show "Active (running)"
- At this point, JENKINS IS RUNNING IN THIS SERVER.


