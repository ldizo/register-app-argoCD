# register-app-argoCD
In this project, we shall build acomplete CI/CD Pipeline using Jenkins, then we integrate the Application SCM GitHub Repository, Maven, Docker, EKS Cluster, Trivy, ArgoCD Slack and the 2nd GitHub repository (which host those Manifest Files) to the Jenkins Pipeline.
Such that, when a Developer makes any changes to the Application source code and pushes it to the GitHub Application Source code Repository, it automatically triggers the Continuous Integration (CI) Jenkins Job. This will immediately pull the Application code from that GitHub Repository and the build process (by Maven) will automatically get initiated. Then Static Code Analysis (by SonarQube) will proceed to perform quality test on the Application. In the Jenkinsfile, it instructs Jenkins to futher package and build the image using the Dockerfile and further push it to our defined DockerHub Repository. After that this created Docker Image will go through a scanning process by Trivy-scan to eliminate any vulnerabilities. This action ends the CI Job.
When everything is automated, the Continuous Deployment (CD) Job still defined in Jenkins as the 2nd Job get trigerred immediately. It will first of all update the the Build Number (the Release Number) in the Deployment Yaml File within the 2nd GitHub Repository that is hosting all the Kubernetes Manifest Files. From that 2nd GitHub Repository, ArgoCD which is installed in the K8s cluster will automaticall pull the changes noticed in that 2nd Repo and it will automatically deploy the resources inside the EKS Cluster. 
And once that CD Job is completed, it will send a notification on Slack to the entire team or it can send an email to team members concerned.

**![Screenshot 2024-11-16 at 7 29 08 AM](https://github.com/user-attachments/assets/ec7a597a-8f85-4afa-88eb-cf85d9e47704)**

# [--- Implementation] ---
We shall use the Jenkins Master-Client Architecture, so that the main Jenkins server will not be overloaded.

# (1) Install and configure the Jenkins-Master server and the Jenkins-Agent server
 
Create a Jenkins Master Server in the Console. so
- Locate and click on "Launch Instance"
  - Name: **Jenkins-master**
  - OS image: **Ubuntu**
  - Amazon Machine Image (AMI): **Ubuntu server 22.04 LTS**  `Your GitHub Account`
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
***sudo apt upgrade***
- Open and rename the Hostname here.
***`vi /etc/hostname`***
- Now, Erase everything in there and type this
***`Jenkins-Master`***
- Now, save and quite.
***`:wq!`***
- Now, reboot the system.
***`sudo init 6`***
  
- Now, go to the console and open the Firewall Rule.
- So, go to "Security Group"
- click on "create a Security Group"
- Click on "Edit inbound rules"
- Click on "Add rule"
- Type: ***Custom***, TCP Protocol: ***TCP***, Port range: ***8080***, Source: ***Anywhere ipv4***
- Then click on "create" to create the Firewall Rule

- Proceed to install Java on this Jenkins VM. Use this command to install java.
***`sudo apt install openjdk-17-jre`***
- Now, check to confirm that Java is succesfully installed.
***`java -version`***
  
- It should show you the version of openjdk thats running in the system
  
- Now, proceed to install Jenkins. You can go to the Jenkins Documentation page for Ubuntu/Debian and copy the code under "Weekly release"
  ***`sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins`***

- now, Enable the Jenkins service to start at boot.
***`sudo systemctl enable jenkins`***
- Then, Start Jenkins as a service.
***`sudo systemctl start jenkins`***
- Now, Check the status of jenkins to ensure that its running and Active.
***`systemctl status jenkins`***
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
***`sudo apt update`***
- Then, proceed to upgrade the system.
***`sudoapt upgrade`***
- Open and rename the Hostname here.
***`vi /etc/hostname*`**
- Now, Erase everything in there and type this
***`Jenkins-Agent`***
- Now, save and quite.
***`:wq!`***
- Now, reboot the system.
***`sudo init 6`***
- it should now appear as follows
  
  **ubuntu@Jenkins-Agent:~$**

- Also install Java in the Agent. so run this command
***`sudo apt install openjdk-17-jre`***
- Now, check to confirm that Java is succesfully installed.
***`java -version`***
  
- It should show you the version of openjdk thats running in the system
- Now, install DOCKER inside this Jenkins Agent Server.
- so do
***`sudo apt-get install docker.io`***
-Do you want to continue? **y**

- Now, give full rights or permissions to the current User inside the Jenkins-Agent on Docker. Use this command
***`sudo usermod -aG docker $USER`***
- Now, reboot the system. So do
***`sudo init 6`***
- Still in the Jenkins-Agent, we shall connect the Jenkins-Agent to the Jenkins-Master fot them to be connected together.
- So, still in the Jenkins-Agent, open the ssh configuration File.
***`sudo vi /etc/ssh/sshd_config`***
- inside the configuration File, scroll down to locate and uncomment "Public Key Authentication" to turn it to "yes". So uncomment it to appear as follows;
***pubKeyAuthentication yes***
- Then continue to scroll down again to locate and uncomment "AuthorizedKeysFile" to appear as follows
***AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2***
- Now, save and quite.
***`:wq!`***

- Now, navigate to the Jenkins-Master server, which is **Ubuntu@Jenkins-Master:~$**
- Proceed to open the configuration file of this Master server. So do
***`sudo vi /etc/ssh/sshd_config`***
- In here, also scroll down to locate and uncomment "Public Key Authentication" to turn it to "yes". So uncomment it to appear as follows;
***pubKeyAuthentication yes***
- Then continue to scroll down again to locate and uncomment "AuthorizedKeysFile" to appear as follows
***AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2***
- Now, save and quite.
***`:wq!`***
- At this point, while still inside the Jenkins-Master, reload the ssh service. So do
***`sudo service sshd reload`***
- At this point, also go to the Jenkins-Agent and reload the ssh service as well. So in the Agent, also do
***`sudo service sshd reload`***
- Now, go to the Jenkins-Master server and generate the ssh key. So in the Master Server, do
***`ssh-keygen`***
- It says "Enter File in which to save the key (/home/ubuntu/.ssh/id_rsa):
***press ENTER on your keyboard***
- It further says "Enter passphrase (empty for no passphrase):
***press ENTER on your keyboard again***
- It also says "Enter same passphrase again"
***press ENTER again on your keyboard***
- The key has been generated
- Now do ***`pwd`*** to ensure that you are inside /home/ubuntu
- Now, go into .ssh directory. so do
***`cd .ssh/`***
- Inside this .ssh directory, do **`ls`**
- You will see "authorized_keys"
- You will also see both your private key **id_rsa** and the public key **id_rsa.pub**
- Now, open the public key. So do
***`cat id_rsa.pub`***
- Now, copy the complete key from "**ssh** all the way to the end **-master**"

- Now, go into the Jenkins-Agent server where you have **ubuntu@Jenkins-Agent:~$**
- Here, do ***`pwd`*** to see that you are in **/home/ubuntu**
- Then, go to the **.ssh** Directory. so do
***`cd .ssh/`***
- Inside this Directory, do **`ls`**
- You will see a File called **authorized_keys**
- Now, get into this "**authorized_keys**" and paste that public key of the Jenkins-Master server here
  "If you cat authorized_key File, you will see both the private and the public key"
- Now, get inside the Public Key and you go below that public key and paste the public key of the Jenkins-Master server that you copied
- Then save and quit
***`:wq!`***
- Now, cat the authorized_key file to see those 2 public keys aligned there. So do
***`cat authorized_keys`***
- you will see a "READONLY" public key content of the Agent first above it, then you see the public key content of the Master second directly below it

# (2) Access the Jenkins-Master Server and configure Jenkins to integrate the Agent to the Master Node

- **So, copy the Public IP of the Jenkins-Master and take it to a Browser to open it with port 8080**
  - **`172.31.0.62:8080`**
- Now, unlock Jenkins. so
  - Copy the path in red and take it to the Master Node to generate the needed password. so
  - While in the Master Node, do ***`sudo cat <paste that path here>`***
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
    - Now, go to the Jenkins-Master (i.e ubuntu@Jenkins-Master:~) and do ***`/.ssh`*** to populate the Private Key to copy it. Copy the Private Key of this Jenkins-Master from ***---BEGIN OPENSSH PRIVATE KEY*** right up to ***---END OPENSSH PRIVATE KEY---***
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
  
# (3.0) Integrate Maven to Jenkins and Add GitHub Credentials to Jenkins

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
   - Version: "**`jdk-17.0.5+8`**" {This version, we shall use for jdk}
   - Then click on "Apply" and then click on "Save"

# (3.1) Proceed to add our Github Credentials to Jenkins {Use this approach if your GitHub Account is private}

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
    - Repository URL: **`https://github.com/Ashfaque-9x/register-app`** {replace this with your Application Source code Repo git repo URL}
    - Credentials: "Select your github credentials" **`Ashfaque-9x/`***(github)**
    - Branch Specifier: "****`/main`***"
    - Script path: **`Jenkinsfil`e**
    - Check the box on "Lightweight checkout"
    - Click now on "Apply" and then you click on "Save"
    - Now go up and click on **Build now**  (No build trigger yet for now)

# (4) Install and Configure Sonarqube

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
  ***`sudo apt update`***
- Also upgrade the system. So do
  ***`sudo apt upgrade`***
- Now, add the Postgresql Repository by running this command
  ***`sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list`***
- Then run this command as well still adding postgresql.
  ***`wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null`***
- Now, install Postgresql. Use this command to do so.
  ***`sudo apt update`*** to first update the system
- Then proceed to install postgresql by running this command
  ***`sudo apt-get -y install postgresql postgresql-contrib`***
- As postgresql hass been installed, enable it. so do
  ***`sudo systemctl enable postgresql`***
- Now, create a password for SonarQube using this command.
  ***`sudo passwd postgres`***
  - New password: **`sonar`** {you have to remember this password}
  - Retype new password: **`sonar`**
- Now, run this command to change the user to "postgres"
  ***`su - postgres`***
  - Password: ***`sonar`***
- Now, create a user for sonarQube. So do
  ***`createuser sonar`***
- Run this command as well. So do
  ***`psql`***
- As the User is now postgres, run this command to alter the user with an encrypted password
  ***`ALTER USER sonar WITH ENCRYPTED password 'sonar';`***
- Now, create a Database for SonarQube with this command
  ***`CREATE DATABASE sonarqube OWNER sonar;`***
- Then proceed to grant privileges to the Database. So do
  ***`grant all privileges on DATABASE sonarqube to sonar;`***
- Now, run this command
  ***`\q`***
- Then exit from the postgres user.
- So do ***`exit`***
  Now, the Database for SonarQube has been created succesfully.
- So, proceed to add the Adoptium Repository. Start by doing
  ***`sudo bash`***
- Then run this command
  ***`wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc`***
- Now, run this long command too
  ***`echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list`***
- Now proceed to install Java17 on this SonarQube VM.
- But first of all update the system.
  ***`apt update`***
- Install tumerin 17 with jdk. So do
  ***`apt install temurin-17-jdk`***
- Do you want to proceed? **y**
- Now update it alternatives. So do
  ***`update-alternatives --config java`***
- Then run this command
  ***`/usr/bin/java --version`***
  You will see the version of Java "openjdk 17.0.8.1" which has been installed.
- Now, exit.
  ***`exit`***

- Now, we need to do some Linux Kernel turning. But first of all, we will increase the limits. So run this command to increase the limit in the Conf file
  ***`sudo vi /etc/security/limits.conf`***
  - At the last end of the configuration file where you see **# End of File**, go to the next line and add this command
    - ***`sonarqube   -   nofile   65536`***
    - ***`sonarqube   -   nproc    4096`***
  - Now, save and quit. So do ***`:wq!`***
  - Proceed to increase the mapped memory regions. So open this file by doing
    - ***`sudo vim /etc/sysctl.conf`***
  - Now, go to the tail end of this config file and insert this without commenting it
    - ***`vm.max_map_count = 262144`***
  - Now, save and quit. So do ***`:wq!`***
  - Now, reboot the system. So do
    ***`sudo init 6`***

Now, actually start the SonarQube installation proper. So first of all download the packages with this command
   ***`sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.0.65466.zip`***
- Now, install the unzip in your system by doing
  ***`sudo apt install unzip`***
- Now, unzip SonarQube. So do
  ***`sudo unzip sonarqube-9.9.0.65466.zip -d /opt`***
- Now run this command
  ***`sudo mv /opt/sonarqube-9.9.0.65466 /opt/sonarqube`***
- Then you create a User group and set permissions for that User. But first of all run this command to add the group user. So do
  ***`sudo groupadd sonar`***
-Then also run  this command
***`sudo useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar`***
- Now, change ownership by running this command. so do
  ***`sudo chown sonar:sonar /opt/sonarqube -R`***
- Then proceed to update SonarQube properties with the database credentials. So open this File by doing
  ***`sudo vi /opt/sonarqube/conf/sonar.properties`***
  - Inside this configuration File, locate and add these values to appear as follows
    - ***sonar.jdbc.username=sonar***    {Note uncomment it}
    - ***sonar.jdbc.password=sonar***    {Note uncomment it}
  - Now, still inside this configuration file, locate the sonar jdbc postgresql url and change it to appear as follows
    - ***sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube***
  - Now, save and quite. So do ***`:wq!`***
- Now, proceed to create Service for SonarQube. Start by creating and vi into the sonar.service file. So do
  ***`sudo vim /etc/systemd/system/sonar.service`***
  - Then, paste this content inside that sonar.service file
    ***`[Unit]
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
     WantedBy=multi-user.target`***
  -  Now, save and quite. So do ***`:wq!`***
- At this point, start sonarQube and enable Service. So do
  ***`sudo systemctl start sonar`***
- Then you enable sonarqube by doing
  ***`sudo systemctl enable sonar`***
- Now, check the status to Sonar to enure that sonarqube.service is running. So do
  ***`sudo systemctl status sonar`***
  It should say "Active (running)
  - So Sonarqube.Service is running in the system
- We can now login and monitor the set up with this command
  ***`sudo tail -f /opt/sonarqube/logs/sonar.log`***
- now, copy the public IP of the SonarQube Server or VM Instance and take it to a Browser to access it on port 9000
  - **`13.233.77.86:9000`***
    - **Login to SonarQube***
    - Login or Username: **`admin`**
    - Password: ***`admin`***
    - ***Update your Password**
    - Old password: **`admin`**
    - New password: **`adminadmin`**
    - Confirm Password ***`adminadmin`***
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
      - Name: **`Sonarqube-webhook`**
      - URL: **`http://172.31.0.62:8080/sonarqube-webhook/`**     {The IP address used here is the Public IP of the Jenkins-Server}
      - Secret:
      - Now, click on "Create" to create this Webhook

  # (5) Integrate SonarQube with Jenkins

  So, in the SonarQube Dashboard or UI
  - Click on your Account icon or picture at the top right
  - Then click on "Security" up at the top
  - Under the "**Generate Tokesn**"
    - Name of the Token: **`jenkins-sonarqube-token`**
    - Type: **`Global Analysis Token`**
    - Expires: **`No expiration`**
    - Then, click on "**Generate**" to generate the token
    - Now, copy the token to store it somewhere in your Local System, as we shall use this token to create its credentials under Jenkins

  - Now, go or navigate to the Jenkins Dashboard and click on "Manage Jenkins"
  - Then locate and click on "**Credentials**"
  - Under "Stores Scoped to Jenkins" and under "Domain"
    - click on (global)
    - Then click on "Add credentials" in the middle
      - Kind: **secret text**
      - Secret: ***<paste that sonarqube token that you copied here>***
      - ID: **`Jenkins-sonarqube-token`**
    - Now, click on "Create"

  # Install and Configure few plugins on Jenkins.
  - So, go again to the Jenkins Dashboard and click on "Manage Jenkins"
  - Locate and click on "Available plugins"
  - In the search box, type and search for "**`sonar`**"
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
    - Name: **`sonarqube-server`**
    - Server URL: **`http://172.31.15.204:9000`**  {The Ip address used here is the Private IP of the SonarQube Server}
    - Sever authentication token
      - Select: **Jenkins-sonarqube-token**    {which we have created using the API token}
      - Then click on "Apply" and click on "Save"

  # Now, proceed to add SonarQube Scannar in Jenkins
  - So, click again on "Manage Jenkins"
  - Then click on "Tools"
  - Scroll down to locate "**SonarQube Scannar Installations**"
    - Under it, click on "Add sonarqube scanner"
    - Name: **`sonarqube-scanner`**
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
  - Username: **`ashfaque9x`**  {paste your DockerHub username here}
  - Password: **provide the Access token**
    - To generate this Access Token of your DockerHub
    - Go to your DockerHub Account and click on your "name icon" at the top right
    - Then click on "Account Settings"
    - Then click on "Security"
    - Then click on "New Access Token"
  - ID: **dockerhub**
  - Then click on "Create"

# (6) Create a Pipeline Script (Jenkinsfile) for Build $ Test Artifacts and Create CI Job on Jenkins

- Now, go to your Github Account and click to get into that Repository that is hosting the Application Source Code. (which is register-app)
  - =github.com/Ashfaque-9x/register-app=
- So, in this Repository, create a "**`Jenkinsfile`**" {Here is the content of the Jenkinsfile of this CI Jobe}
          -----------------------------------------------------------------------------------------

pipeline {
    agent { label 'Jenkins-Agent' } 
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

# (7) Setup a Bootsrap Server for EKSctle and Setup Kubernetes using EKSctl

- Here, we are going to create an EKS Bootstrap Server. So, go to the Console and create an Instance
  - Click on "Launch Instance"
  - Name: **`EKS-Bootstrap-Server`**
  - OS Image: **ubuntu**
  - Architecture: **64-bit(x86)**
  - Instance type: **t2-micro**
  - Keypair: **Jenkins-vm-keypair**
  - Now, click on "Launch Instance" ro create the Instance
- Copy the Public IP and log into the instance to have this
  - **[ubuntu@ip-172-31-34-39:~$]**

- Now, update the system. So do **`sudo apt update`**
- Secondly, upgrade the system. So do **`sudo apt upgrade`**
- Do you want to continue? **Y**
- Now, open the Host File and rename it. so do **`sudo vi /etc/hostname`**
  - Here, delete everything in the file and type "`EKS-Boostrap-Server`"
  - Then Save and Quit **`:wq!`**
- Procceed to reboot the system. So do **`sudo init 6`**
- So you now have it appear as
  - **[ubuntu@EKS-boostrap-Server:~$]**

- Now, change to a root user. So do **`sudo su`**
  - So you now have it appears as
  - **[root@EKS-Boostrap-Server:~$]**

**Install AWS CLI in this Server**
- So, firstly, get to the root or home directory. So do **`cd ~`**
- Now, check your present Working directory to ensure that you are on "root". So do **`pwd`**
  - You should see it appear as **`/root`**
- Now, download AWS cli using this command.
  **`curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"`**
- Install the unzip using this command
  **`apt install unzip`**
- Now, unzip the downloaded aws CLI
  **`unzip awscliv2.zip`**
- Now, run this command to actually install the AWS cli
  **`sudo ./aws/install`**
- Proceed to check and confirm that AWS CLI is installed and its running. So do
  **`/usr/local/bin/aws --version`**
  - You should now see the version  of AWS running in the system
  - You can as well do **`aws --version`** to see thesame thing

**Install kubectl in this Boostrap Server**
- So, first of all downloadkubectl using this command.
  **`curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.27.1/2023-04-19/bin/linux/amd64/kubectl`**
- Now, do **`ll`** or **`ls`**
 - You will see **`awscliv2.zip`** and kubectl there but its not having executable permissions. So we have to give it executable permissions. So
 - Run this command to give kubectl executable permissions. So do
   **`chmod +x ./kubectl`**
- Now, do **`ll`** or **`ls`**
  - You realize that "kubectl" has turn to "green colour". So it now have executable permissions.
- Since our Executable files should be in a "/bin" Directory, we should move "kubectl" to the "/bin" Directory. So do
  - **`mv kubectl /bin`**
  - Now, check to confirm that its now installed. So do
    - **`kubectl version --output=yaml`**
    - You will see something appear as "clientversion:builddata, compiler etc" that are installed in this server
    - Ignore that warning sign that says "The connection to the server localhost:8080 was refused. did you specify ...." Ignore it becuase we are not connected to any cluster.

-**MY ADDENDUM**

- **`You will first of all set up "Cloud SDK" in your local Laptop" for GCP Setup (For AWS, You will need to setup AWS CLI in your Local)`**
- **`[You will need to install kubectl in Kubectl in your local or laptop, (so as to be able to interact with the cluster from your Local`.** "Windows setup is different from mac setup"



**Proceed to install EKSctl in this EKS-Boostrap-Server**
- So, download the package in the "tmp" Folder. Run this command
  **`curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp`**
- Now, go to the tmp Folder. so
  **`cd /tmp`**
- Now, do **`ll`** or **`ls`**
  - You will see that "eksctl" is in green. Its already having executable permissions here.
- Now, move the eks file from the "tmp" Folder to the "/bin" Folder. This is because all executable files are present in this "/bin" Folder. So do
 - **`sudo mv /tmp/eksctl /bin`**
- Now, check the eksctl version running. So do **`eksctl version`**
  - You should see the version thats running in there
Now, move or come out from the root. so do **`cd ~`**

**Now, create an IAM Role to assign to the EKS Bootstrap Server**
- So, go to the Console and go to "IAM"
- Click on "Role" and click on "Create role"
- Click on "AWS Service"
- Use Case
  - Service or use case: **EC2**
  - Check the box on "EC2"
- Click on "Next"
- Permissions policy
  - Here, check the box on "AdministratorAccess"
- Click now on "Next"
- Role name: **`eks_role`**
- Now, scroll down and click on "Create role".
  - Now, the Role has been created

- Procceed. So, go back to EC2 Instance of the EKS-Boostrap-Server and click to check the box beside this "EKS-Boostrap-Server"
- Then click on "Action" at the top
- Then you click on "Security"
  - Click here on "Modify IAM Role"
  - In the modify IAM Role page that pops up, Inside the box, use the drop down to select "eksctl-role" A role that we just created.
  - Then click on "Update IAM role"

# (8) Create the EKS Cluster

- To create this EKS Cluster, go to the Boostrap Server and set it up there.   ***{Since we are creating or setting up the cluster inside this VM Instance}***
- Go to the Terminal of this Server that appears as
  - **[root@EKS-Boostrap-Server]**
- Here, pass this command to create a Virtual Cluster in there. So run this command
  # `eksctl create cluster --name virtualtechbox-cluster \`
 - Pass the region here. **`--region ap-south-1 \`**
 - Now, Pass the node-type. **`--node-type t2.small \`**
 - Then proceed to give the Node Count. **`--nodes 3 \`**
   - **The EKS cluster creation will take some time to be set up**
   - Now, verify and confirm that nodes are up and running. So do **`kubectl get nodes`**
      - You should see 3 Nodes which are ready inside this Cluster

# (9) ArgoCD installation on the EKS Cluster and add EKS Cluster to Argo CD

**So the next task is Argo CD Installation and it Configuration**
- So, first of all, on the Boostrap Server, terminal, where the cluster is running, we shall create a Namespace there for ArgoCD.
- Thus, in this Terminal **root@EKS-Boostrap-Server:~$**
  - Create a Namespace for ArgoCD. So do
  - ***`kubectl create namespace argocd`***
  - Now, in this namespace, apply the ArgoCD yaml configuration file. This is the command for that ArgoCD yaml file to get applied here in this Namespace. So pass this command
  - ***`kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`***   {Here, we used an auto-generated ArgoCD yaml file. Do not change anything on this file as it is autogenerated. Any change on it will make ArgoCD to fail}
 
  - Now, we can check to see the Pods that are created in the ArgoCD Namespace by default. So do
  - ***`kubectl get pods -n argocd`***  {These are the pods that are running inside ArgoCD BY DEFAULT, in our ArgoCD Namespace. "1 is still initializing"}
  - Now, to interact with the API Server, we need to deploy the ArgoCD CLI first. So run this command to deploy the ArgoCD CLI.
  - ***`curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64`***
  - Now, give ArgoCD the executable permissions. So do
  - ***`chmod +x /usr/local/bin/argocd`***
  - Now, we need to expose the ArgoCD Server to the External World by creating a LoadBalancer. So do
  - ***`kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'`***   {It will take some time to get created in the background}
  - Now, check to see if the LoadBalancer has been created. So do
  - ***`kubectl get svc -n argocd`***  {You will see the LoadBalancer there with its IP and it External IP or URL that you can use to access it}.
  - However, before Accessing the LoadBalancer, we need to extract the password for the ArgoCD. To get the Password for this ArgoCD, do
  - ***`kubectl get secret argocd-initial-admin-secret -n argocd -o yaml`***   {it will generate a password which appears as **echo WXV-----==I base64 --decode**}
    
  - Now, we need to decode it. So, copy the password that is generated above after running the command and paste it here
  - ***`echo WXV-----==I base64 --decode`*** Hit enter to decode the password
  - The decoded password appears as follows
  - ***Sd6eqrtGVK9TowKM***root@EKS-Boostrap-Server:~#   {Here is the decoded password in Bold}
  - Now, copy the External IP or the long URL of the LoadBalancer and take it to a new Browser and hit enter
  - On the page that pops up, click on "Advance"
  - Then input the following requested information
    - Username: **`admin`**
    - Password: **`Sd6eqrtGVK9TowKM`**     {This is that decoded password. So go copy it above and paste here}
    - Now, hit "enter"
      - ArgoCD UI comes up
     
      - # ----------------ARGO CD UI---------------
     
      - Now, we have to sign or log into this Argo CD. To log into the Argo CD Cluster;
      - Go and locate "User info" at the left and click on it
      - Then click on "**UPDATE PASSWORD**
      - On the "Update Account Password" page that pops up, input these information
        - Current Password: ***`Sd6eqrtGVK9TowKM`**  {Paste the decoded password here}
        - New Password: ***`admin`***   {type your own personal chosened password here}
        - Confirm new Password: ***`admin`**    {Confirm your new password}
     - Then click on "SAVE NEW PASSWORD"
     - Now, click on "Log out" at the top right to log out
     - Then Login with the new password. So input this information
       - Username: ***`admin`***
       - Password: ***`admin`***
     - Click now on "`Sign in`"
- Now, go back to the EKS-Boostrap-Server terminal
- **root@EKS-Boostrap-Server:~$**

# (10) Add the EKS Cluster to Argo CD.

- First of all, we have to login to AgroCD from the CLI. So, in the terminal, run this command
- ***`argocd login <Paste the LoadBalancer URL here> --username admin`***
- {So the whole command will look like this ***`argocd login a2255bb2bb33f438d9addf8840d294c5-785887595.ap-south-1.elb.amazonaws.com --username admin`*** }
- It says "Warning server certificate had error. Certificate is valid for Localhost". Reply (Y/N)?:
- Type: **Y**
- Password: **`admin`** {Provide the Password of your ArgoCD Account}
- Login Successful {So we have succesfully login to ArgoCD Cluster using the CLI}

- Now, if you go to Argo CD Dashboard and click on "**Settings** at the left,
- Then you click on "Cluster"
  - You will see only 1 Cluster (which is the default Cluster of ArgoCD)

- Now, come back to the Boostrap-Server and list the Cluster. You will see only that same default ArgoCD Cluster.
- To see that, do ***`argocd cluster list`***  {You will see the default Cluster here as well

- Now, we need to extract the name of our EKS Cluster. To see the details of our EKS Cluster, do
- ***`kubectl config get-contexts`***   {Its shows you the CURRENT NAME: **`i-03b9doffo409f48e7@virtualtechbox-cluster.ap-south-1.eksctl.io`**. NAMESPACE: **`Vitualtechbox-cluster.app-south`**. CLUSTER:  }
- Now, we need to add this our EKS Cluster to ArgoCD. For that to be added, run this command
- ***`argocd cluster <paste the CURRENT NAME of our cluster here> --name <Pass the name of our cluster that we are giving here>`***
- So the whole command will appear like this ***`argocd cluster add i-08b9d0ff0409f48e7@virtualtechbox-cluster.ap-south-1.eksctl.io --name virtualtechbox-eks-cluster`***
- It says "Warning. This will create a Service Account". Do you want to continue [Y/N]?
- Type: **Y**
- It will actually create a Service Account for ArgoCD Manager. Then it also create a Role Binding
- Now, list the Cluster to check and confirm that our EKS Cluster is now available in ArgoCD. So do
- ***`argocd cluster list`***   {You will see both the default cluster and our cluster that we have just created}
- Now, go to the ArgoCD DASHBOARD and refresh. {You will now see 2 Clusters there. The default Cluster and our own Ckuster}

# (11) Configure ArgoCD to Deploy pods on EKS and automate ArgoCD Deployment Job using GitOps GitHub Repository.

- **Requirements:**
  - (i) Have another GitHub Repository known as **`github.com/Asfaque-9x/gitops-register-app`**   {which is having Those Manifest Files for Kubernetes. Note: This Repository is in GitHub}
  - (ii) Then we shall need to connect this 2nd Repository which is **`github.com/Asfaque-9x/gitops-register-app`** hosting our Manifest Files to ArgoCD Cluster

-Now, we are going to connect our ArgoCD to the 2nd GitHub Repository
- So, Go to ArgoCD DASHBOARD and click on "Settings" at the left so as to connect the repository.
- Then click on "Repositories"
- Then click on "+ Connect Repo"
- Then locate "**VIA SSH**" and click on the drop down below it.
  - Then you select "VIA HTTPS"
  - Type: **git**
  - Project: **default**
  - Repository URL: **`<Paste that 2nd GitHub Repo URL here>`** {which is **`https://github.com/Asfaque-9x/gitops-register-app`** }
  - Username: **Paste your GitHub Account Username here** {Ashfague-9x}
  - Password: **Paste the Personal Access Token for that your GitHub Account here**
  - Then you click on "Connect"
  - Now, our 2nd Repository has been added to ArgoCD DASHBOARD
  - This Repository {**`gitops-register-app`**} have its URL as **`https://github.com/Asfaque-9x/gitops-register-app`** have our Manifest Files which are the "deployment.yaml" and the "service.yaml"}
  - **deployment.yaml**
    - apiVersion: apps/v1
kind: Deployment
metadata:
  name: register-app-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: register-app
  template:
    metadata:
      labels:
        app: register-app
    spec:
      containers:
        - name: register-app
          image: ashfaque9x/register-app-pipeline:1.0.0-9
          resources:
            limits:
              memory: "256Mi"
              cpu: "500m"
          ports:
            - containerPort: 8080
              
- **Service.yaml**
  -apiVersion: v1
kind: Service
metadata:
  name: register-app-service
  labels:
    app: register-app 
spec:
  selector:
    app: register-app

  ports:
    - port: 8080
      targetPort: 8080

  type: LoadBalancer

# Now, we have to deploy the Application in the EKS Cluster through Argo CD
- So, to deploy the Application in the EKS Cluster through ArgoCd, go to the ArgoCD DASHBOARD.
- Under "Applications"
  - Click at the top on "NEW APP"
  - Application Name: **`register-app`***
  - Project Name: **`default`**
  - SYNC POLICY: Automatic
  - Check the box on **PRUNE RESOURCES**
  - Check the box on **SELF HEAL**
  - Source:
    - Repository: **`https://github.com/Asfaque-9x/gitops-register-app`** {Paste the URL of the 2nd Repo here. This Repository contains the Manifest Files for K8s Application}
    - Revision: **HEAD**
    - Path: **`./`**
  - Destination:
    - Cluster URL: {Select your cluster URL in the drop down using the drop down menure} it appears something like **`https://FoBB3378900D8FoBec---ap-south-1.eks.amazon.aws.com`**
    - Namespace: **`default`**  {It will deloy the resource in our kubernetes cluster in the default Namespace}
    - Now, click on **create**
- The Application has been deployed in the EKS Cluster. So the Deployment is happening perfectly as seen in the ArgoCD DASHBOARD

<img width="1431" alt="Screenshot 2024-11-20 at 5 37 56 PM" src="https://github.com/user-attachments/assets/2fa4c307-277f-4800-b544-f7c9afc4bedd">

- The Deployment is also happening perfectly in our EKS Cluster throught the Argo CD. So if we go to our Boostrap-Server terminal and do ***`kubectl get pods`***
- you will see 2 pods running inside the default Namespace of our EKS Cluster.
- To get the External DNS Name of this Application, do ***`kubectl get svc`***   {You will see the EXTERNAL-IP of the Loadbalancer appears in the default Namespace through the Service Manifest File
- Now, copy this entire EXTERNAL-IP of this LoadBalancer and put in a new Browser:8080
- # Application UI of Tomcat Appears

- And if we paste the EXTERNAL-IP:8080/webapp/ "enter"
- # New User Registration for DevOps Learning pops up
  {This is our Application}
- Now, we need to automate this deployment process.

# (12) Automating the Deployment Process

- To automate this Deployment process,
- Go to the Jenkins DASHBOARD and there we shall create a Continuous Deployment (CD) Job there. So
- In the Jenkins DASHBOARD, click on "+ New Item"
  - Item Name: **`gitOps-register-app-cd`**
  - Click on "Pipeline"
  - Then click on "O.K"
  - Under "General",
    - Check the box on **Discard old builds**
  - Max number of builds to keep: **2**
    - This project is Parameterized
    - Click now on "**Add Parameter**"
    - Then click on "**String Parameter**"
    - Name: **`IMAGE_TAG`**        {Because our deployment will always change the image tag in the GitOps Repository}
    - Build Triggers
    - Check the box on **Trigger builds remotely (e.g from Script)**
    - Authentication Token: Name it as **`gitops-token`**
    - Pipeline:
    - Definition: Select: **Pipeline script from SCM**
    - SCM: **Git**
    - Repositories:
    - Repository url: **`https://github.com/Asfaque-9x/gitops-register-app`**  {Paste the URL of the 2nd GitHub Repo here that is hosting those Manifest Files}
    - Credentials: **`Ashfague-9x/`***(github)
    - Branch Specifier: ***`/main`**
    - Check the box on **Lightweight**
    - Then click on "Apply" and click on "Save"
    - Now, the triggered CD Pipeline stage need to be added in the Jenkinsfile of the Application source code Repo and save it by commiting the changes (if only you did not add that last stage or trigger the CD Stage).

# (13) Create a Jenkinsfile in the GitOps Repository
- The next thing to do is to create a Jenkinsfile in our gitOps Repository. This is because our CD Job has been configured with that Repository. And that Repository must have a Jenkinsfile as our CD Job has been configured as a Declarative Job which will to pull the Jenkinsfile from the GitOps Repository to deploy it in the EKS Cluster via ArgoCD. For that;
- Go to the GitOps Repository which has those Manifest Files for K8s and click on "Add File"
- Then click on "Create new file" in this Repository in the main Branch
- Name: **Jenkinsfile**
- pipeline {
    agent { label "Jenkins-Agent" }
    environment {
              APP_NAME = "register-app-pipeline"
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
               steps {
                   git branch: 'main', credentialsId: 'github', url: 'https://github.com/Ashfaque-9x/gitops-register-app'
               }
        }

        stage("Update the Deployment Tags") {
            steps {
                sh """
                   cat deployment.yaml
                   sed -i 's/${APP_NAME}.*/${APP_NAME}:${IMAGE_TAG}/g' deployment.yaml
                   cat deployment.yaml
                """
            }
        }

        stage("Push the changed deployment file to Git") {
            steps {
                sh """
                   git config --global user.name "Ashfaque-9x"
                   git config --global user.email "ashfaque.s510@gmail.com"
                   git add deployment.yaml
                   git commit -m "Updated Deployment Manifest"
                """
                withCredentials([gitUsernamePassword(credentialsId: 'github', gitToolName: 'Default')]) {
                  sh "git push https://github.com/Ashfaque-9x/gitops-register-app main"
                }
            }
        }
      
    }
}

- **DONE**

- # -----Testing-------

  # (14) Verify the CI/CD Pipeline by doing a test commit on the GitHub Repository (from the Application Source Code Repository**

- Everything has now been set up. Now, verify our CI/CD Pipeline. But before we verify the CI/CD Pipeline, we should set the trigger in our CI Job, so that it will trigger the entire Pipeline once any change is made that lands on our Application Source Code GitHub Repository. To set up such Trigger;
- Go to the Jenkins DASHBOARD and locate and click on our "**`register-app-ci`**" job
- Then click on "Configure" at the left
- Under "**Build Triggers",
- locate and check the box on "**Poll SCM**"
- In the box under "**Schedule**", put 5 stars there to appear as "*****" {These stars means that "every minutes", which means that, every minutes, it will monitor this Repository. And any change, it will trigger the Build Job.
- Pipeline:
  - Definition: **Pipeline script from scm**
  - SCM: **Git**
- Repositories:
  - Repository URL: **`https://github.com/Asfaque-9x/register-app`**  {This is the URL of the Application Source Code Repository or that of 1st Repo}
  - Credentials: Ashfague-9x/****(github)
- Branches to build
  - Branch specifier: ***`/main`**
  - Now, click on "Apply" and then click on "Save"
- Now, we have opened the CI Job and the CD Job on a 2 different tabs, so as to see clearly how the CI Job will be triggered or react when a trigger takes effect, and also to see how a CD Job will reat simultaneousely when the triggers happens.

# (i) ------------ First Test --------------

- Now, use your local terminal and connect or cd into the Application Source Code remote Repository and **ensure that you are in the "main Branch" of that Repository that is hosting the Source Code**
- { That Branch should have the **src** folder, the **pom.xml** File, and the **Jenkinsfile** all together in that Repository.}
- Now, inject another File called **index.jsp** file. So, inside that Directory, do
  - ***`vi index.jsp`***
  - Paste this content in there at the tail end
  - **<h1> Happy Learning. </h1>
  - Now, save and quite ***`:wq!`***

- Now, commit it to the remote Repository. So do
  - ***g`it add .`***
  - ***`git commit -m "changed the index file"`***
  - ***`git push origin main`***

- **`As soon as the changes get pushed to the Application Source Code Repository, Our Jenkins CI Job get triggered automatically, and the CI Build job starts immediately. (See the Jenkins CI Job Dashboard)`**
  - **NOW, THE C.I JOB HAS BEEN COMPLETED SUCCESFULLY** --

- Lets now proceed to the CD Jenkins Job Dasboard.
- **`As soon as the CI Job gets completed, it also triggers the CI Job automatically and its own Build too get started immediately. (Go to the other tab of the CD jENKINS job and see it as its happening`**

- --------- **Now, go to DockerHub and Refresh** -------------------
- You will see that, the image "**a`shfague9x/register-app-pipeline`**" has just been pushed there.
- Click on that image name. You will see that it has a "**Latest**" Tag which is **1.0.0-8** (Note the "**8** here as it is the latest Build # which is showing on the Build Jenkins DASHBOARD beside the build.
- The previous one is **1.0.0-7**

- ---------- **Now, go to the gitOps Repository and Refresh as well** ----------------------
- Click on the "**deployment.yaml**" File
- You will see that the image in this yaml Manifest File has been automatically updated to the latest tag with "**8**" at the end

- ---------- **Now, go to Argo CD DASHBOARD and Refresh as well** --------------------------
- Then in the box or page, click on the name "**vitualtechbox-eks-cluster**"
- Destination: **`virtualtechbox-eks-cluster`**               {click on this name, - which is the app}
- Then click above on "APP DETAILS"
- Scroll down to image. You will see the image there carrying **1.0.0-8**
- So the App is using the "Latest Release", which is the **"8"** Release.

- ----------- **Now, click on the Pod** ---------------
- You will see also that the Pod there has been automatically updated to the latest Release ("Which is **8**")

-  ---------- **Now, go to the Application Brouser and Refresh as well** ---------------------
-  You will see that
-  "**`Happy Learning`**" comes up as the latest changes by the Latest Release.
  
# (ii) ------------ Second Test --------------
- We want to test it again one more time. So
- Still go to the Terminal or Local terminal of the Application Source Code and
- Open again  the **index.jsp** file
- Then under it, add this there
- **`<h1> Happy Learning. See You Again. </h1>`**
- Now, save the File and quite ***`:wq!`***

- Now, commit it to the remote Repository. So do
  - ***`git add .`***
  - ***`git commit -m "Display message changed on the main page"`***
  - ***`git push origin main`***

- --------------------- **Now, go to the CI Job of the Jenkins DASHBOARD tap and Refresh** -------------------------
- You will see that the **Build #9** has been automatically triggered and it goes into effect immediately upon the push to the Git Repository.
- CI Job completed succesfully
- Now, go to the CD part

- -------------------- **Now, go to the CD Job of the Jenkins DASHBOARD tap and refresh** ------------------------
- You will also see that it has been automatically triggered, upon the trigger of the CI Job.
- And its own Build has automatically started as well.
- CD Job also completed Succesfully

-  ------------------ **Now, go to DockerHub and Refresh again** ------------------------------------------------
-  You will see the image that has just been pushed there. Click on that image
-  You will see that the "TAG **1.0.0-9** has been added

-  ------------------ **Now, go to ArgoCD DASHBOARD again and Refresh** ------------------------------------------
- Now, in the box or page, click on the name "**vitualtechbox-eks-cluster**"
- Destination: **`virtualtechbox-eks-cluster`**               {click on this name, - which is the app}
- Then click above on "APP DETAILS"
- Scroll down to 'IMAGES'. You will see that, the latest image there has been automatically  updated carrying **1.0.0-9**
- So the App is using the "Latest Release", now which is the **"9"** Release.

- -------------------- **Now, click on Pod** ----------------------------------
- You will see the changes appear there as well as
- **`Happy Learning. See You Again`**

- So, by this way, you can set up a Declarative Pipeline using **Jenkins**, **Docker**, **ArgoCD**, and **EKS**. Any change on that drops on the GitHub Application Repository, it will atomatically trigger the C.I Job in Jenkins. And once the CI Job is completed, CD Job will be automatically triggered. And upon completion of the CD Job, we will have the changes applied to our Application at all levels (i.e in DockerHub, GitOps Manifest Files in ArgoCD and right down to the pod and to the Application at the tail end.

# --------- Challenges faced ----------
# --------- Challenge Number (1) --------
- The major challenge I faced running this Pipeline was at the level of generating the ArgoCD intial Token which comes with base 64 encoded. The right way is to first generate the token using imperative way, Then you proceed to decode it as it comes in an encoded form (base 64 encoded), which then permit us to use the decoded password and the ArgoCD user to bring up the ArgoCD UI.
- If you dont follow this approach in order to extract that token, before procceeeding to decode it to use it, it will consistently fail.
- Bringing up the ArgoCD UI involves a 2 steps setup which needs the decoded token in all of those steps to be able to to gain Access to the ArgoCD Succesfully.
  - (1) At the first level, the decoded token is used to access the ArgoCD UI             and
  - (2) At the second level, the decoded token is used to usher you in as a user into your own ArgoCD Environment which is running within the Cluster.

# ---------- Challenge Number (2) -----------
- The defualt IAM Role that comes with the Cluster is insufficient to make the Cluster to interact well with Argo CD and other services, especially when it comes to microservice Application that has multiple components. So without an additional IAM Role with full permission attached to the Cluster will limit the eks environment to interact with other services. So and additional IAM Role is created and attached to the cluster before things really work well
