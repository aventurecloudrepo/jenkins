This document is an intellectual property of Aventure Cloud LLC, Grand Rapids, Michigan, USA and restricts the distribution by whatsoever means like print, electronic media etc., The document usage is restricted to certain persons given access via Google Classroom of Aventure Cloud LLC and prevent unauthorized access and usage, failing which they will be prosecuted in federal court by the U.S. Attorney’s Office.

#### Install AZ CLI module if not already installed

Windows

```
winget install -e --id Microsoft.AzureCLI
```

Ubuntu

```
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

Connect to Azure: AZ Login

#### Create a test directory called jenkins-server

#### Switch to the test directory

#### Create a file named cloud-init-jenkins.txt

#### Paste the following code into the new file:

    #cloud-config
    package_upgrade: true
    runcmd:
      - sudo apt install openjdk-11-jre -y
      - curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
      - echo 'deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/' | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
      - sudo apt-get update && sudo apt-get install jenkins -y
      - sudo service jenkins restart

#### Create a resource group

```
az group create --name jenkins-rg --location eastus
```

#### Create a virtual machine

When running on Windows replace the backslashs ( \ ) with backticks ( ` ) at the end of each line

```
az vm create \
--resource-group jenkins-rg \
--name jenkins-vm \
--image canonical:ubuntu-24_04-lts:server:24.04.202411030 \
--admin-username "azureuser" \
--generate-ssh-keys \
--public-ip-sku Standard \
--custom-data cloud-init-jenkins.txt\
--size Standard_B2s
```

#### Verify the creation (and state) of the new virtual machine

```
az vm list -d -o table --query "[?name=='jenkins-vm']"
```

#### As Jenkins runs on port 8080, open port 8080 on the new virtual machine

When running on Windows replace the backslashs ( \ ) with backticks ( ` ) at the end of each line

```
az vm open-port \
--resource-group jenkins-rg \
--name jenkins-vm \
--port 8080 \
--priority 1010
```

#### Get the public IP address for the virtual machine

When running on Windows replace the backslashs ( \ ) with backticks ( ` ) at the end of each line

```
az vm show \
--resource-group jenkins-rg \
--name jenkins-vm -d \
--query [publicIps] --output tsv
```

#### SSH into the virtual machine

```
ssh azureuser@<ip_address>
```

#### Verify that Jenkins is running by getting the status of the Jenkins service

```
sudo systemctl status jenkins
```
You may get error like this:

    azureuser@jenkins-vm:~$ sudo systemctl status jenkins
    × jenkins.service - Jenkins Continuous Integration Server
         Loaded: loaded (/usr/lib/systemd/system/jenkins.service; enabled; preset: enabled)
         Active: failed (Result: exit-code) since Tue 2024-12-17 08:11:03 UTC; 1min 21s ago
        Process: 14528 ExecStart=/usr/bin/jenkins (code=exited, status=1/FAILURE)
       Main PID: 14528 (code=exited, status=1/FAILURE)
            CPU: 412ms
    
    Dec 17 08:11:03 jenkins-vm systemd[1]: jenkins.service: Scheduled restart job, restart counter is at 5.
    Dec 17 08:11:03 jenkins-vm systemd[1]: jenkins.service: Start request repeated too quickly.
    Dec 17 08:11:03 jenkins-vm systemd[1]: jenkins.service: Failed with result 'exit-code'.
    Dec 17 08:11:03 jenkins-vm systemd[1]: Failed to start jenkins.service - Jenkins Continuous Integration Server.
    Dec 17 08:11:03 jenkins-vm systemd[1]: jenkins.service: Start request repeated too quickly.
    Dec 17 08:11:03 jenkins-vm systemd[1]: jenkins.service: Failed with result 'exit-code'.
    Dec 17 08:11:03 jenkins-vm systemd[1]: Failed to start jenkins.service - Jenkins Continuous Integration Server.

That's because Jenkins requires Java 17 or above.
Refer to this URL: https://www.jenkins.io/blog/2024/06/11/require-java-17/
So how did i know that?
I executed this command at my linux terminal:

    journalctl -u jenkins.service
And the first few lines in the output gives the clue:

    Dec 17 08:10:59 jenkins-vm systemd[1]: Starting jenkins.service - Jenkins Continuous Integration Server...
    Dec 17 08:10:59 jenkins-vm jenkins[14356]: Running with Java 11 from /usr/lib/jvm/java-11-openjdk-amd64, which>
    Dec 17 08:10:59 jenkins-vm jenkins[14356]: Supported Java versions are: [17, 21]
    Dec 17 08:10:59 jenkins-vm jenkins[14356]: See https://jenkins.io/redirect/java-support/ for more information.
Supported Java versions are: [17, 21]!
So let us now uninstall Java11 (we sued it in cloud-init script) and install Java21:

    sudo apt-get install -y openjdk-21-jre
    sudo apt-get remove -y openjdk-11-jre
Finally restart Jenkins and verify if it's running:

    sudo systemctl restart jenkins
    sudo systemctl status jenkins
You should see something like this:

    ● jenkins.service - Jenkins Continuous Integration Server
         Loaded: loaded (/usr/lib/systemd/system/jenkins.service; enabled; preset: enabled)
         Active: active (running) since Tue 2024-12-17 08:29:14 UTC; 7s ago
       Main PID: 15269 (java)
          Tasks: 44 (limit: 4023)
         Memory: 401.5M (peak: 406.8M)
            CPU: 14.093s

#### Get the autogenerated Jenkins password

```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

#### Using the IP address, open the following URL in a browser: **http://<ip_address>:8080**
