# register-app-argoCD
In this project, we shall build acomplete CI/CD Pipeline using Jenkins, then we integrate the Application SCM GitHub Repository, Maven, Docker, EKS Cluster, Trivy, ArgoCD Slack and the 2nd GitHub repository (which host those Manifest Files) to the Jenkins Pipeline.
Such that, when a Developer makes any changes to the Application source code and pushes it to the GitHub Application Source code Repository, it automatically triggers the Continuous Integration (CI) Jenkins Job. This will immediately pull the Application code from that GitHub Repository and the build process (by Maven) will automatically get initiated. Then Static Code Analysis (by SonarQube) will proceed to perform quality test on the Application. In the Jenkinsfile, it instructs Jenkins to futher package and build the image using the Dockerfile and further push it to our defined DockerHub Repository. After that this created Docker Image will go through a scanning process by Trivy-scan to eliminate any vulnerabilities. This action ends the CI Job.
When everything is automated, the Continuous Deployment (CD) Job still defined in Jenkins as the 2nd Job get trigerred immediately. It will first of all update the the Build Number (the Release Number) in the Deployment Yaml File within the 2nd GitHub Repository that is hosting all the Kubernetes Manifest Files. From that 2nd GitHub Repository, ArgoCD which is installed in the K8s cluster will automaticall pull the changes noticed in that 2nd Repo and it will automatically deploy the resources inside the EKS Cluster. 
And once that CD Job is completed, it will send a notification on Slack to the entire team or it can send an email to team members concerned.

**![Screenshot 2024-11-12 at 9 35 34 PM](https://github.com/user-attachments/assets/5b413388-42ba-419a-9f86-603777520b95)**

                                           -- [Implementation] --
We shall use the Jenkins Master-Client Architecture, so that the main Jenkins server will not be overloaded.

  **(1) Install and configure the Jenkins-Master server and the Jenkins-Agent server**
 
Create a Jenkins Master Server in the Console. so
- Locate and click on "Launch Instance"
  - Name: **Jenkins-master**
  - OS image: **Ubuntu**
  - Amazon Machine Image (AMI): **Ubuntu server 22.04 LTS**
  - Architecture: **64-bit(x86)**
  - Keypair
    - click on "create new keypair"
    - Keypair name: **Jenkins-vm-keypair**
    - click on "create" to create the Keypair
    - Configure storage (Hard Disk): **15 GB**
- Now, click on "Launch Instance" to create the instance
- Now, copy the Public Ip of this Jenkins-Master Vm and ssh into it from your local system
- You should have it as:
  
  **ubuntu@ip-172-31-0-62:~$**
  
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

Our objective is to use the Jenkins Master-Client Architecture, so that we dont overload the Master Server. So whatever command that we pass in the Master will get executed in the Client Server. So, we now proceed to create a Client Sevrer. so go back to the console and create a client server.so
- Click again on "Launch Instance"
   - Name: ***Jenkins-Agent***
   - OS: ***Ubuntu***
   - Amazon Machine Image (AMI): ***Ubuntu server 22.04 LTS***
   - Architecture: ***64-bit(x86)***
   - Keypair: select our keypair: **Jenkins-vm-keypair**
   - Configure Storage: Hard Disk): ***15 GB***
- Then click on "Launch Instance" to create the Jenkins Agent VM
- Now, copy the public IP of the Agent VM and use it to ssh into it from your local. it should appear as follows
  
  **ubuntu@ip-172-31-6-16:~$**

- Now, first of all update the system.
***sudo apt update***
- Then, proceed to upgrade the system.
***sudoapt upgrade***
- Open and rename the Hostname here.
***vi /etc/hostname***
- Now, Erase everything in there and type this
***Jenkins-Agent***
- Now, save and quite.
***:wq!***
- Now, reboot the system.
***sudo init 6***
- it should now appear as follows
  
  **ubuntu@Jenkins-Agent:~$**

- Also install Java in the Agent. so run this command
***sudo apt install openjdk-17-jre***
- Now, check to confirm that Java is succesfully installed.
***java -version***
  
- It should show you the version of openjdk thats running in the system
- Now, install DOCKER inside this Jenkins Agent Server.
- so do
***sudo apt-get install docker.io***
-Do you want to continue? **y**

- Now, give full rights or permissions to the current User inside the Jenkins-Agent on Docker. Use this command
***sudo usermod -aG docker $USER***
- Now, reboot the system. So do
***sudo init 6***
- Still in the Jenkins-Agent, we shall connect the Jenkins-Agent to the Jenkins-Master fot them to be connected together.
- So, still in the Jenkins-Agent, open the ssh configuration File.
***sudo vi /etc/ssh/sshd_config***
- inside the configuration File, scroll down to locate and uncomment "Public Key Authentication" to turn it to "yes". So uncomment it to appear as follows;
***pubKeyAuthentication yes***
- Then continue to scroll down again to locate and uncomment "AuthorizedKeysFile" to appear as follows
***AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2***
- Now, save and quite.
***:wq!***

- Now, navigate to the Jenkins-Master server, which is **Ubuntu@Jenkins-Master:~$**
- Proceed to open the configuration file of this Master server. So do
***sudo vi /etc/ssh/sshd_config***
- In here, also scroll down to locate and uncomment "Public Key Authentication" to turn it to "yes". So uncomment it to appear as follows;
***pubKeyAuthentication yes***
- Then continue to scroll down again to locate and uncomment "AuthorizedKeysFile" to appear as follows
***AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2***
- Now, save and quite.
***:wq!***
- At this point, while still inside the Jenkins-Master, reload the ssh service. So do
***sudo service sshd reload***
- At this point, also go to the Jenkins-Agent and reload the ssh service as well. So in the Agent, also do
***sudo service sshd reload***
- Now, go to the Jenkins-Master server and generate the ssh key. So in the Master Server, do
***ssh-keygen***
- It says "Enter File in which to save the key (/home/ubuntu/.ssh/id_rsa):
***press ENTER on your keyboard***
- It further says "Enter passphrase (empty for no passphrase):
***press ENTER on your keyboard again***
- It also says "Enter same passphrase again"
***press ENTER again on your keyboard***
- The key has been generated
- Now do ***pwd*** to ensure that you are inside /home/ubuntu
- Now, go into .ssh directory. so do
***cd .ssh/***
- Inside this .ssh directory, do **ls**
- You will see "authorized_keys"
- You will also see both your private key **id_rsa** and the public key **id_rsa.pub**
- Now, open the public key. So do
***cat id_rsa.pub***
- Now, copy the complete key from "**ssh** all the way to the end **-master**"

- Now, go into the Jenkins-Agent server where you have **ubuntu@Jenkins-Agent:~$**
- Here, do ***pwd*** to see that you are in **/home/ubuntu**
- Then, go to the **.ssh** Directory. so do
***cd .ssh/***
- Inside this Directory, do **ls**
- You will see a File called **authorized_keys**
- Now, get into this "**authorized_keys**" and paste that public key of the Jenkins-Master server here
  "If you cat authorized_key File, you will see both the private and the public key"
- Now, get inside the Public Key and you go below that public key and paste the public key of the Jenkins-Master server that you copied
- Then save and quit
***:wq!***
- Now, cat the authorized_key file to see those 2 public keys aligned there. So do
***cat authorized_keys***
- you will see a "READONLY" public key content of the Agent first above it, then you see the public key content of the Master second directly below it

**(2) Access the Jenkins-Master Server and configure Jenkins to integrate the Agent to the Master Node**

- **So, copy the Public IP of the Jenkins-Master and take it to a Browser to open it with port 8080**
  - **172.31.0.62:8080**
- Now, unlock Jenkins. so
  - Copy the path in red and take it to the Master Node to generate the needed password. so
  - While in the Master Node, do ***sudo cat <paste that path here>***
  - Now, copy the password and take it to the Jenkins UI
  - Paste the Password in the "Administrator Password" box
  - Then click on ***Continue***
  - Click on ***install suggested plugins***
    
  **CREATE FIRST ADMIN USER**
  - Usernamae: ***Clouduser***
  - password: ***admin***
  - confirm password: ***admin***
  - Full name: ***Kenneth***
  - Email address:
- Click now on "Save and continue"
- Then click on "Save and Finish"
- Then click on "Start using Jenkins"
- Now, go to "Manage Jenkins" and click on it
- Now, click on "Nodes"
- Click on "Build-in Node" in the middle. Click on the name itself
  - Then locate on the left and click on "Configure"
  - Number of Executors? **0**
  - Then click on "Save"
- Now, click on **Dashboard** and click again on "Manage Jenkins"
  - Click again on "Nodes"
  - Now, click on "New Nodes" at the top right
  - Node name: Jenkins-Agent
  - Check the box on "Permanent Agent"
  - Click now on **Create**
  - Number of Executors: 2
  - Remote root directory: ***/home/ubuntu*** {this is the home Directory of the Agent Server}
  - Labels: ***Jenkins-Agent***
  - Usage: **Use this node as much as possible**
  - Host: Post the private IP of the Jenkins-Agent here {Because both are in thesame VPC. If they were in different VPC, we can use External Ip}
  - Credentials: **None**
  - Click on "+Add"
  - Then click on "Jenkins"
  - Kind: **SSH Username with private key**
  - ID: **Jenkins-Agent**
  - Username: **Ubuntu**
  - Private Key
     - check the box on "Enter directly"
  - Click now on "Add"
  - Enter the New Secret Below
  - Key
    - Now, go to the Jenkins-Master (i.e ubuntu@Jenkins-Master:~) and do ***/.ssh*** to populate the Private Key to copy it. Copy the Private Key of this Jenkins-Master from ***---BEGIN OPENSSH PRIVATE KEY*** right up to ***---END OPENSSH PRIVATE KEY---***
    - Paste the Private Key here
  - Credentials: Select ***Ubuntu(Jenkins-Agent)*** {which we have just created}
  - Host Key Verification Strategy: ***Non verifying verification Strategy***
  - Now, click on ***Save***
Now the Jenkins-Agent has just been added to our Jenkins-Master

- Now go to the Dashboard and click on "***New item**"
- Enter an item name: **Test**
- Then click on **"Pipeline"**
- Then click on **"OK"**
- Scroll down to locate **"Pipeline"**
- Script: on the box at the right, select ***"Hello world"** {so as to start building the script}
- Click now on ***Apply*** and click on ***Save***
- Click now on "Build Now"
- Now, the Build has been completed succesfully as a test in the Agent. Which means that our connection between the Jenkins-Master and Jenkins-Agent is succesfull. As you can see here saying
- "**Running on Jenkins-Agent in /home/ubuntu/workspace/Test**"
- You can now delete the **Test** job (so as not to confuse us)
  
**(3.0) Integrate Maven to Jenkins and Add GitHub Credentials to Jenkins**

We shall start by configuring few plugins in Jenkins on this Jenkins DASHBOARD. So
- Go to "**Manage Jenkins**"
- Then locate and click on **Plugins**
- Then click on **Available plugins** in the left
- On the search bar, type and search for "**Maven**"
  - Check the box on "**Maven Integration**"
  - Check also the box on "**Pipeline Maven Integration**"
- Now, type and search again for "**Eclipse Temurin Installer**"
  - Check the box on "**Eclipse Temurin Installer**"
- Then you proceed to click on ***Install*** at the top right.

- Now, we need to configure these Plugins. So go to the DASHBOARD again and click on "**Manage Jenkins**"
- Then click on ***Tools***
  - Scroll down to locate "**Maven Installations**"
  - Then click on "**Add Maven**"
  - Name: ***Maven3***  {Note that, this name will be recalled in the Pipeline}
  - Check this box on "**Installed automatically**"
  - Then click on "Apply and click on "Save"

- Click again on ***Tools***
   - Scroll down to locate "**JDK Installations**"
   - Click on "**Add JDK**"
   - Name: "**Java17**" {Note; This name will be recalled in the Pipeline script}
   - Click and check the box on "**Install automatically**"
   - Then click on "**Add installer**"
   - Then select: "**Install from adoptium.net**"
   - Version: "**jdk-17.0.5+8**" {This version, we shall use for jdk}
   - Then click on "Apply" and then click on "Save"

(3.1) Proceed to add our Github Credentials to Jenkins {Use this approach if your GitHub Account is private}

- So, go to the Jenkins DASHBOARD and click again on "**Manage Jenkins**"
- Now, locate "SECURITY" and click on "**Credentials**"
- Under "Stores Scoped to Jenkins", locate "Domains" and click below it on "**global**"
- Then click on "**Add credentials**"
  - Username: **Klekeanyi** {Pass the username of your GitHub Account here}
  - Password: ....... {Paste here your Personal Access Token of your GitHub Account here}
    - To generate or create this Personal Access Token;
    - Go to your GitHub Account and click on your picture icon or name at the far top right
    - click now on "**Settings**" at the bottom of the populated page
    - At the left, scroll down and click on "**Developer settings**"
    - Then click on "**Personal Access Tokens**"
    - Then click on "**Tokens (classic)**"
    - Now click at the top right on "**Generate new token**" to generate the token
  - ID: **github** {We shall recall this github ID in the Pipeline Script
  - Description: **github**
  - Now, click on "**Create**"

  **Link your Github Application Source Code Repository to Jenkins.**

- So, In the Jenkins DASHBOARD, CLICK ON "**New Item**"
  - Name: **register-app-ci**
  - Then click on **Pipeline**
  - Now, click on **OK**
  - Under **General**;
  - Check the box on "**Discard old builds**"
  - Max number of build to keep: **3**
  - Pipeline Script: Select: **Pipeline Script from SCM**
  - SCM: **Git**
  - Repositories
    - Repository URL: **https://github.com/Ashfaque-9x/register-app** {replace this with your Application Source code Repo git repo URL}
    - Credentials: "Select your github credentials" **Ashfaque-9x/***(github)**
    - Branch Specifier: "****/main***"
    - Script path: **Jenkinsfile**
    - Check the box on "Lightweight checkout"
    - Click now on "Apply" and then you click on "Save"
    - Now go up and click on **Build now**  (No build trigger yet for now)

**(4) Install and Configure Sonarqube**

- So, go to the console and create a Sonarqube VM Instance. So click on "Launch Instance"
  - Name: **Sonarqube**
  - OS: **Ubuntu**
  - Architecture: **64-bit(x86)**
  - Instance type: **t3. medium** {t2.micro is insuffient for Sonarqube, because we shall be installing postgresql and Sonarqube in this Vm}
  - Keypair: Select our **Jenkins-vm-keypair**
  - Configure Storage (Disk): **15 GB**
  - Now, click on "Launch Instance"
- Open the Firewall or Security Group of this SonarQube. Note that SonarQube runs on port **9000**. so
  - On this newly created Sonarqube instance, click beside it to check it box.
  - Then click on "Security" to go to "Security Group"
  - Click now on "Edit inbound rules"
  - Click on "Add rule"
  - Type: **Custom TCP**, Protocol: **TCP**, Port range: **9000**, Source: **Anywhere ipv4**.
  - Then click on "Save rule".
- Now ssh into this VM using it External IP. so you will have something like this
  
  **Ubuntu@ip-172-31-15-204:~$**

- Now, update the package repository. So do
  ***sudo apt update***
- Also upgrade the system. So do
  ***sudo apt upgrade***
- Now, add the Postgresql Repository by running this command
  ***sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'***
- Then run this command as well still adding postgresql.
  ***wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null***
- Now, install Postgresql. Use this command to do so.
  ***sudo apt update*** to first update the system
- Then proceed to install postgresql by running this command
  ***sudo apt-get -y install postgresql postgresql-contrib***
- As postgresql hass been installed, enable it. so do
  ***sudo systemctl enable postgresql***
- Now, create a password for SonarQube using this command.
  ***sudo passwd postgres***
  - New password: **sonar** {you have to remember this password}
  - Retype new password: **sonar**
- Now, run this command to change the user to "postgres"
  ***su - postgres***
  - Password: ***sonar***
- Now, create a user for sonarQube. So do
  ***createuser sonar***
- Run this command as well. So do
  ***psql***
- As the User is now postgres, run this command to alter the user with an encrypted password
  ***ALTER USER sonar WITH ENCRYPTED password 'sonar';***
- Now, create a Database for SonarQube with this command
  ***CREATE DATABASE sonarqube OWNER sonar;***
- Then proceed to grant privileges to the Database. So do
  ***grant all privileges on DATABASE sonarqube to sonar;***
- Now, run this command
  ***\q***
- Then exit from the postgres user.
- So do ***exit***
  Now, the Database for SonarQube has been created succesfully.
- So, proceed to add the Adoptium Repository. Start by doing
  ***sudo bash***
- Then run this command
  ***wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc***
- Now, run this long command too
  ***echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list***
- Now proceed to install Java17 on this SonarQube VM.
- But first of all update the system.
  ***apt update***
- Install tumerin 17 with jdk. So do
  ***apt install temurin-17-jdk***
- Do you want to proceed? **y**
- Now update it alternatives. So do
  ***update-alternatives --config java***
- Then run this command
  ***/usr/bin/java --version***
  You will see the version of Java "openjdk 17.0.8.1" which has been installed.
- Now, exit.
  ***exit***

- Now, we need to do some Linux Kernel turning. But first of all, we will increase the limits. So run this command to increase the limit in the Conf file
  ***sudo vi /etc/security/limits.conf***
  - At the last end of the configuration file where you see **# End of File**, go to the next line and add this command
    - ***sonarqube   -   nofile   65536***
    - ***sonarqube   -   nproc    4096***
  - Now, save and quit. So do ***:wq!***
  - Proceed to increase the mapped memory regions. So open this file by doing
    - ***sudo vim /etc/sysctl.conf***
  - Now, go to the tail end of this config file and insert this without commenting it
    - ***vm.max_map_count = 262144***
  - Now, save and quit. So do ***:wq!***
  - Now, reboot the system. So do
    ***sudo init 6***

Now, actually start the SonarQube installation proper. So first of all download the packages with this command
   ***sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.0.65466.zip***
- Now, install the unzip in your system by doing
  ***sudo apt install unzip***
- Now, unzip SonarQube. So do
  ***sudo unzip sonarqube-9.9.0.65466.zip -d /opt***
- Now run this command
  ***sudo mv /opt/sonarqube-9.9.0.65466 /opt/sonarqube***
- Then you create a User group and set permissions for that User. But first of all run this command to add the group user. So do
  ***sudo groupadd sonar***
-Then also run  this command
***sudo useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar***
- Now, change ownership by running this command. so do
  ***sudo chown sonar:sonar /opt/sonarqube -R***
- Then proceed to update SonarQube properties with the database credentials. So open this File by doing
  ***sudo vi /opt/sonarqube/conf/sonar.properties***
  - Inside this configuration File, locate and add these values to appear as follows
    - ***sonar.jdbc.username=sonar***    {Note uncomment it}
    - ***sonar.jdbc.password=sonar***    {Note uncomment it}
  - Now, still inside this configuration file, locate the sonar jdbc postgresql url and change it to appear as follows
    - ***sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube***
  - Now, save and quite. So do ***:wq!***
- Now, proceed to create Service for SonarQube. Start by creating and vi into the sonar.service file. So do
  ***sudo vim /etc/systemd/system/sonar.service***
  - Then, paste this content inside that sonar.service file
    ***[Unit]
     Description=SonarQube service
     After=syslog.target network.target

     [Service]
     Type=forking

     ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
     ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

     User=sonar
     Group=sonar
     Restart=always

     LimitNOFILE=65536
     LimitNPROC=4096

     [Install]
     WantedBy=multi-user.target***
  -  Now, save and quite. So do ***:wq!***
- At this point, start sonarQube and enable Service. So do
  ***sudo systemctl start sonar***
- Then you enable sonarqube by doing
  ***sudo systemctl enable sonar***
- Now, check the status to Sonar to enure that sonarqube.service is running. So do
  ***sudo systemctl status sonar***
  It should say "Active (running)
  - So Sonarqube.Service is running in the system
- We can now login and monitor the set up with this command
  ***sudo tail -f /opt/sonarqube/logs/sonar.log***
- now, copy the public IP of the SonarQube Server or VM Instance and take it to a Browser to access it on port 9000
  - **13.233.77.86:9000***
    - **Login to SonarQube***
    - Login or Username: **admin**
    - Password: ***admin***
    - ***Update your Password**
    - Old password: **admin**
    - New password: **adminadmin*
    - Confirm Password ***adminadmin***
    - Click now on "Update"
    - Immediately you click on "updates", it takes you directly to the SonarQube Dashboard
  - It takes you to the SonarQube DASHBOARD
  - {You will see it "**PASSED**" or "**FAILED**" for the build on this DASHBOARD

  # - SONARQUBE DASHBOARD -
  ** Now, create a Sonarqube Webhook, to directly hookup Sonarqube with Jenkins in order to perform it Software Composition Analysis (Static Code Analysis) test before the Build
  - So, on the SonarQube Dashboard, locate "**Aministrator**" at the top and click on it
    - Then locate "**Configuration** and click on the drop down beside it to populate "Webhook" that you will click on it
    - Click on "**Webhook**"
    - Then click on "**Create**" at the top right to create a Webhook
      - Name: **Sonarqube-webhook**
      - URL: http://172.31.0.62:8080/sonarqube-webhook/     {The IP address used here is the Public IP of the Jenkins-Server}
      - Secret:
      - Now, click on "Create" to create this Webhook

  **(5) Integrate SonarQube with Jenkins**

  So, in the SonarQube Dashboard or UI
  - Click on your Account icon or picture at the top right
  - Then click on "Security" up at the top
  - Under the "**Generate Tokesn**"
    - Name of the Token: **jenkins-sonarqube-token**
    - Type: **Global Analysis Token**
    - Expires: **No expiration**
    - Then, click on "**Generate**" to generate the token
    - Now, copy the token to store it somewhere in your Local System, as we shall use this token to create its credentials under Jenkins

  - Now, go or navigate to the Jenkins Dashboard and click on "Manage Jenkins"
  - Then locate and click on "**Credentials**"
  - Under "Stores Scoped to Jenkins" and under "Domain"
    - click on (global)
    - Then click on "Add credentials" in the middle
      - Kind: **secret text**
      - Secret: ***<paste that sonarqube token that you copied here>***
      - ID: **Jenkins-sonarqube-token**
    - Now, click on "Create"

  # Install and Configure few plugins on Jenkins.
  - So, go again to the Jenkins Dashboard and click on "Manage Jenkins"
  - Locate and click on "Available plugins"
  - In the search box, type and search for "**sonar**"
    - Check the box on "**SonarQube Scannar**"
    - Check also the box on "**SonarQube Gates**"
    - Also check the box on "**Quality Gates**"
  - Now, click on "install" at the top right
  - Scroll down and click on "**Restart Jenkins**" to restart it.
    - When installation is completed,
     - Ignore the warning signs because we have created the sonarqube Database already which is postgresql

  # Now, proceed to add this SonarQube Server to Jenkins.
  - So, go to Jenkins and click again on "Manage Jenkins"
  - Then click on "system"
    - Scroll down to locate and click on "**Add sonarqube**" under "SonarQube Installations".
    - Name: **sonarqube-server**
    - Server URL: **http://172.31.15.204:9000**  {The Ip address used here is the Private IP of the SonarQube Server}
    - Sever authentication token
      - Select: **Jenkins-sonarqube-token**    {which we have created using the API token}
      - Then click on "Apply" and click on "Save"

  # Now, proceed to add SonarQube Scannar in Jenkins
  - So, click again on "Manage Jenkins"
  - Then click on "Tools"
  - Scroll down to locate "**SonarQube Scannar Installations**"
    - Under it, click on "Add sonarqube scanner"
    - Name: **sonarqube-scanner**
    - Check this box on **Install automatically**
    - Then click on "Apply" and then click on "Save"
      
# Now, Intall Docker related Plugins in the Jenkins Server
- So, on the Jenkins Dashboard, locate and click again on "Manage Jenkins"
- Then click on "Plugins"
- Click on "Available plugins"
  - In the Search box, type "Docker" and check these boxes
  - Check the box on "**Docker 1.5**"
  - Check the box on "**Docker Common**"
  - Check the box on "**Docker Pipeline**"
  - Check the box on "**Docker API**"
  - Check the box on "**Docker-build-steps**"
  - Check the box on "**CloudBees Docker Build and push**"
  - Now, click on "Install" at the top
  - Scroll down and restart Jenkins
- Check this box on "Restart Jenkins when installation is complete and no jobs are running.

# Now, add DockerHub Credentials in your Jenkins
- So, go again to "Manage Jenkins"
- Then click on "Credentials"
- Under Stores Scoped to Jenkins", go directly under "Domains" and click on "**global**"
- Click now on "**Add Credentials**"
  - Kind: **username with password**
  - Username: **ashfaque9x**  {paste your DockerHub username here}
  - Password: **provide the Access token**
    - To generate this Access Token of your DockerHub
    - Go to your DockerHub Account and click on your "name icon" at the top right
    - Then click on "Account Settings"
    - Then click on "Security"
    - Then click on "New Access Token"
  - ID: **dockerhub**
  - Then click on "Create"

**(6) Create a Pipeline Script (Jenkinsfile) for Build $ Test Artifacts and Create CI Job on Jenkins**

- Now, go to your Github Account and click to get into that Repository that is hosting the Application Source Code. (which is register-app)
  - =github.com/Ashfaque-9x/register-app=
- So, in this Repository, create a "**Jenkinsfile**" {Here is the content of the Jenkinsfile of this CI Jobe}
          -----------------------------------------------------------------------------------------

pipeline {
    agent { label 'Jenkins-Agent' } **[the Agent or Jenkins-Agent VM Instance that was created and connected to the Master Node or Jenkins-Master is defined here]**
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }
    environment {
	    APP_NAME = "register-app-pipeline"
            RELEASE = "1.0.0"
            DOCKER_USER = "ashfaque9x"
            DOCKER_PASS = 'dockerhub'
            IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
            IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
	    JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }
    stages{
        stage("Cleanup Workspace"){
                steps {
                cleanWs()
                }
        }

        stage("Checkout from SCM"){
                steps {
                    git branch: 'main', credentialsId: 'github', url: 'https://github.com/Ashfaque-9x/register-app'
                }
        }

        stage("Build Application"){
            steps {
                sh "mvn clean package"
            }

       }

       stage("Test Application"){
           steps {
                 sh "mvn test"
           }
       }

       stage("SonarQube Analysis"){
           steps {
	           script {
		        withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') { 
                        sh "mvn sonar:sonar"
		        }
	           }	
           }
       }

       stage("Quality Gate"){
           steps {
               script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
                }	
            }

        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image = docker.build "${IMAGE_NAME}"
                    }

                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }

       }

       stage("Trivy Scan") {
           steps {
               script {
	            sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ashfaque9x/register-app-pipeline:latest --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table')
               }
           }
       }

       stage ('Cleanup Artifacts') {
           steps {
               script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
               }
          }
       }

       stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh "curl -v -k --user clouduser:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'ec2-13-232-128-192.ap-south-1.compute.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'"
                }
            }
       }
    }

    post {
       failure {
             emailext body: '''${SCRIPT, template="groovy-html.template"}''', 
                      subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed", 
                      mimeType: 'text/html',to: "ashfaque.s510@gmail.com"
      }
      success {
            emailext body: '''${SCRIPT, template="groovy-html.template"}''', 
                     subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful", 
                     mimeType: 'text/html',to: "ashfaque.s510@gmail.com"
      }      
   }
}

-----------------------------------------------------------------------------------------------------------------------------


