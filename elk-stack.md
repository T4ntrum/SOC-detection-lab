## **A step-by-step guide to deploying an Elastic Stack for a local SIEM setup.** 
 
 ## Objective
 The goal of this guide is to set up a practical ELK stack to simulate a "production type environment". 
 
 We'll be improving our skills for a SOC role by running Atomic Red Team MITRE ATT&CK techniques.(Mock) 

 By running the attacks, we'll be 
 - Learning to validate logs and improve our detection skills.
 - Learning to observe telemetry in Kibana
 - Analyze process creation and security events


>### DISCLAIMER: I am on CachyOS and used VirtualBox for this setup. 

All sources will be credited at the bottom. 

### Setting up the server

I chose to use an Ubuntu distro for the server. (You can use other distros; it's all personal preference)

If shared clipboard is not working for you and causing problems, I'd recommend installing OpenSSH so you can SSH into the VM from your host machine. 

### Minimum requirements 

VM specs: 
- 2 CPUs
- 4 GB of RAM 
- 50 GB of disk space (Windows install is pretty bloated and you can downsize the Ubuntu deployment to save space if needed)

### My set up 

VM Specs: 
- 4 CPUs 
- 8 GBs of RAM 
- 80 GBs of disk space

### Installing ElasticSearch and Kibana

First update your Ubuntu distro:
sudo apt update && sudo apt upgrade -y

Then get the elastic PGP key:
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-9.x.list
```
After that, update again then install elasticsearch:
```bash
sudo apt update
sudo apt install elasticsearch -y
```
After that we'll have to edit the elasticsearch.yml in any text editor of your choosing (I used nano).
```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```
We'll be editing a few fields
```yml
#the name itself doesn't matter. Use whatever name you want
cluster.name: server 

xpack.security.enabled: false

#This makes it so the server can receive connections from any machine, in this case it'll be from our second VM to send logs.
network.host: 0.0.0.0 

#There's only one node in this cluster
discovery.type: single-node 
```
>**❗ IMPORTANT**: We need to either comment out or remove all mentions of `cluster.initial_master_nodes`, it conflicts with `discovery.type`.

After all that is done, we can save and exit the text editor now and start `elasticsearch` service.
```bash
sudo systemctl daemon-reload #Loads updated version of elasticsearch.yml
sudo systemctl enable elasticsearch.service
sudo systemctl start elasticsearch.service
```
To ensure the service is running properly do:

```bash
sudo systemctl status elasticsearch.service
```
After you confirmed it's running, let's curl it: 

```bash
curl http://localhost:9200
```

The curl should return JSON outputs with your cluster name. 

If it shows errors refer to your clusters logs in 
`/var/log/<cluster-name>.log`.

If no issues have arisen from this point on we can start installing Kibana.

```bash
sudo apt install kibana -y
```
>**NOTE: While installing Kibana you might freeze at 20% installation, do not terminate the install as this is normal behavior. It may take some time for Kibana to install (for me it took about 10 mins, for others it varies as it can take longer)**

Once it's installed we'll have to edit the config file using any text editor of your choice. 

```bash
sudo nano /etc/kibana/kibana.yml 
```
We'll be editing:

```yml
server.name: kibanaserver #name itself doesn't matter once again

server.host: "0.0.0.0" #this allows Kibana to listen for connections on all available network interfaces

elasticsearch.hosts:["http://localhost:9200"]
```
Now as we did with Elastic, save and exit the config file. 

```bash
sudo systemctl daemon-reload
sudo systemctl enable kibana.service
sudo systemctl start kibana.service
```
In theory you should be able to access the Kibana UI at [localhost:5601](http://localhost:5601). To access it on our host machine, we can set up a port forwarding rule to our guest port of 5601 to any available host port for our server VM. [Link to guide](https://www.activecountermeasures.com/port-forwarding-with-virtualbox/).


### Creating our Windows VM and installing winlogbeat and sysmon to send logs to our Elastic deployment.

We can use either a Windows 11 or Windows 10 ISO, I chose to use a Windows 10 ISO per my preference. 

If you're using VirtualBox the Windows account will be made for you and shouldn't have to do the other parts of the installation steps. 

>**NOTE: If asked to make an account and connect to a network in Windows 11 you can go into VirtualBox and change the network settings to NOT ATTACHED. Then press shift + F10 in the VM to enter CMD, then use the command `OOBE /BYPASSNRO`. This should bypass the connection requirement and account creation, then you should be in the Windows OS and you can change your VMs connection type back to NAT.**  

Now that the install is done, we need to download sysmon and winlogbeat. 

Once you've downloaded winlogbeat, go into your program files and create a new folder (`C:\Program Files\Winlogbeat`) and unzip the downloaded zip file in there.
Then open PowerShell as Administrator, and `cd` into `(C:\Program Files\Winlogbeat\<name of unzipped file>)` and then run, 
```ps1
cd 'C:\Program Files\Winlogbeat\<name of unzipped file>'
Set-ExecutionPolicy bypass
.\install-service-winlog.ps1
``` 
Edit the `winlogbeat.yml` with notepad as Administrator and edit the following: 
```yml
output.elasticsearch:
    hosts: [<server-vm-ip>:9200]

# add a new line after hosts
setup.kibana:
    host: "<server-vm-ip>:5601"
```

Now we need to set up a NAT Network for our VMs, so they can communicate with each other. 
In VirtualBox go to `File - Tools - Network - NAT Networks - Create`, that should work. Now let's add Port forwarding so we can access Kibana on our Host Machine. 

At the bottom of NAT Networks click `Port forwarding - Add new port forwarding rule - (name it) - TCP - Host IP 127.0.0.1 - Host Port 5601 - Guest IP (IP of server machine) - Guest Port 5601`  

Add SSH as well so you can access the VM server via OpenSSH on your Host Machine and access Elastic.
`Add new port forwarding rule - (name it) - TCP - Host IP 127.0.0.1 - Host Port 2222 - Guest IP (IP of server machine) - Guest Port 22`

To get the IP of your server VM do `ip addr show`. 

>Note: Don't access the Elastic server yet on host we still need to set up sysmon.

- Download Sysmon on the Windows VM
- Extract the ZIP folder and then open a PowerShell session as Administrator.
- Then download SwiftOnSecurity Sysmon config
- Unzip the folder and move it into the unzipped Sysmon folder.
- Run this command in the PowerShell session  `.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml`

After that we can now finish up with Winlogbeat.

First get back into the Winlogbeat folder then run this command in PowerShell 
```ps1
.\winlogbeat.exe setup --index-management -e
Start-Service winlogbeat
Set-Service -Name winlogbeat -StartupType Automatic
``` 
We'll be setting up the dashboards on Kibana.

If there's been no errors then we should now be able to access Kibana.

On the host machine run, `ssh -L 8080:localhost:5601 -p 2222 server-machine@127.0.0.1`

Then on your browser you can connect to the site by going to `http://localhost:8080`.

Once you get on to the site, we can set up the dashboard go to `Menu - Stack management - Data views - Create data view - Name: Winlogbeat - Index Pattern winlogbeat-*, @timestamp - save - click discover - data view - Winlogbeat`.

If you got this far, everything will be set up and now you'll be able to test malware on your Windows VM. 

## Sources
- [Install Elasticsearch with a Debian package](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-with-debian-package)
- [Install Kibana with Debian package](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-kibana-with-debian-package)
- [Winlogbeat quick start: installation and configuration](https://www.elastic.co/docs/reference/beats/winlogbeat/winlogbeat-installation-configuration)
- [Install Windows 11 Offline: Practical No Internet Setup Guide - Method 2](https://windowsforum.com/threads/install-windows-11-offline-practical-no-internet-setup-guide.383459/)
