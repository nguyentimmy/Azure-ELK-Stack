# 🦌 Azure ELK Stack — Deployment & Monitoring Walkthrough

## 🎯 Project Overview
A three-part walkthrough for deploying an ELK (Elasticsearch, Logstash, Kibana) monitoring stack on Microsoft Azure using Ansible, then using Kibana to hunt through logs and metrics. Built as a purple-team lab: stand up the monitoring, generate attacker activity, and verify the stack detects it.
> Here is a optional pre-requisite, full walk through on setting up two "DVWA" servers using the recommended security settings. If you want a full purple team lab, I do highly recommend going through this guide below. This will provide real time monitoring on those "DVWA" servers.
➡️ [Azure Cloud Security Walkthrough](https://github.com/nguyentimmy/Azure-Virtualization-Security/blob/main/README.md)

**Stack:** Microsoft Azure · Ansible · Docker · ELK (sebp/elk) · Filebeat · Metricbeat · Kibana

## 🧰 Azure Resources, Settings & Security Measures

| Resource / Setting | Function | Security Measures Applied |
| --- | --- | --- |
| **Resource Group** | Logical container holding every resource for the lab | Keeps all lab assets grouped for isolation and easy teardown |
| **Virtual Network — RedTeam VNet** | Original private network hosting the jump box and DVWA web servers | Internal-only address space; foundation for network segmentation |
| **Virtual Network — ELK VNet** | Separate VNet in a different region hosting the ELK monitoring server | Isolates monitoring infrastructure from the workload network |
| **Network Security Group (NSG)** | Firewall controlling inbound/outbound traffic to the VMs | Default-deny baseline with explicit allow-rules layered above |
| **ELK VM** *(Monitoring, 4GB+ RAM)* | Runs the Elastic Stack container; collects logs + metrics | Public IP restricted so only the admin IP reaches Kibana on port 5601 |
| **Ansible container** (`cyberxsecurity/ansible`) | Infrastructure-as-code tool that configures the ELK + web VMs | Provisions consistently via playbook; dedicated SSH key; scoped `remote_user` |
| **Load Balancer** | Distributes inbound web traffic across DVWA 1 & DVWA 2 | Restricts inbound access; health probe monitors backend availability |

---

## 🛡️ Security Principles Demonstrated

- **Network segmentation** — monitoring (ELK) and workload (DVWA) live in separate VNets and regions, connected only by scoped peering.
- **Default-deny networking** — the NSG blocks everything first; access is granted only by explicit, scoped exception.
- **Least-privilege access** — Kibana (port 5601) and SSH are restricted to specific source IPs, not left open.
- **Secure remote administration** — the jump box model funnels all admin access through one hardened, monitored entry point.
- **Key-based authentication only** — password auth is disabled; VMs are reached with SSH keys distributed via Ansible.
- **Infrastructure as code** — Ansible provisions the ELK server and Beats consistently and repeatably, reducing configuration drift.
- **Centralized monitoring & detection** — Filebeat and Metricbeat forward logs and metrics to ELK, giving a single pane to detect SSH brute-force, CPU stress, and web-request floods.

---

# 🦌 Part 1 — ELK Installation
---
## 🌐 Step 1 — Create a New VNet

Make sure that you are logged into your personal Azure account, where your cloud security unit VMs are located.

- Create a new vNet located in the same resource group you have been using.

	- Make sure this vNet is located in a _new_ region and not the same region as your other VM's.

   ![](Step-By-Step-Guide/Part%201/Resources/vNet-images/vNet.png)

Here we are adding it to the `(US) West US` region because all the other resources are in the `(US) East US` region. 

  - Note that _which_ region you select is not important as long as it's a different US region than your other resources.

    ![](Step-By-Step-Guide/Part%201/Resources/vNet-images/vNet-ip.png)

- Leave the rest of the settings at default.
	
  - Notice, in this example, that the IP Addressing has automatically created a new network space of `10.2.0.0/16`. If your network is different (10.1.0.0 or 10.3.0.0) it is ok as long as you accept the default settings. Azure automatically creates a network that will work.

   ![](Step-By-Step-Guide/Part%201/Resources/vNet-images/vNet-Final.png)

- Create a Peer connection between your vNets. This will allow traffic to pass between your vNets and regions. This peer connection will make both a connection from your first vNet to your Second vNet _And_ a reverse connection from your second vNet back to your first vNet. This will allow traffic to pass in both directions.

- Navigate to 'Virtual Network' in the Azure Portal. 

- Select your new vNet to view it's details. 

- Under 'Settings' on the left side, select 'Peerings'.

- Click the `+ Add` button to create a new Peering.

   ![](Step-By-Step-Guide/Part%201/Resources/vNet-images/Peerings-side.png)

- Make sure your new Peering has the following settings:

	- A unique name of the connection from your new vNet to your old vNet.
		- Elk-to-Red would make sense

	- Choose your original RedTeam vNet in the dropdown labeled 'Virtual Network'. This is the network you are connecting to your new vNet and you should only have one option.

	- Name the resulting connection from your RedTeam Vnet to your Elk vNet.
		- Red-to-Elk would make sense

- Leave all other settings at their defaults.

  ![](Step-By-Step-Guide/Part%201/Resources/vNet-images/Peering1.png)

  ![](Step-By-Step-Guide/Part%201/Resources/vNet-images/Peerings-final.png)

## 💻 Step 2 — Create a New VM

Set up a new virtual machine to run ELK.

- SSH into your Jump-Box using `ssh username@jump.box.ip`

- Check for your Ansible container:
 
  ```bash
  sysadmin@Jump-Box-Provisioner:~$ sudo docker ps
  CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
  ```

- Locate the container name:

  ```bash
  sysadmin@Jump-Box-Provisioner:~$ sudo docker container list -a
  CONTAINER ID        IMAGE                          COMMAND             CREATED             STATUS                      PORTS               NAMES                     
  4d16db8c80d6        cyberxsecurity/ubuntu:bionic   "bash"              3 days ago          Exited (0) 3 days ago    											 romantic_noyce
  ```

- Start the container:

  ```bash
  sysadmin@Jump-Box-Provisioner:~$ sudo docker container start romantic_noyce
  romantic_noyce
  sysadmin@Jump-Box-Provisioner:~$
  ```

- Connect to the Ansible container:

  ```bash
  sysadmin@Jump-Box-Provisioner:~$ sudo docker container attach romantic_noyce
  root@6160a9be360e:~#
  ```

- Copy the SSH key from the Ansible container on your jump box:

  ```bash
    # cat ~/.ssh/id_rsa.pub 
  ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDUfoIGFxTFyZXWV0QuCCmPKxsvGhnW/sKwGrOZ/K7nozKxsaRSCSG/oLGbugTyi9+fRY9wYWCmK/HLpjOaTEi8iU+ydvGM8nTloD/dIlje9PClUCxFQjql2XyQz32FqDjHV8rCZA+Pz+9ozc7BogQwLLg/0c4beQYbVQPKs1QGHf31YuXs6hAraJMXCx7VsDJHQwfv1kScE2s+yGeUJMt0ny3xaED8y2Pn+mBF2Tw7HLT+HPkmvXcuCkLxo6gY3ad+EH9Ko0r2AEFvtZTcFyGfIDLcS6jo+GUlKuCLGRAzeKNhq+D78fHf8Vt4qvUSIywP9HHnvnqfUCVKXsKxZGGl root@6160a9be360e

  ```

- Configure a new VM using that SSH key.
    - Make sure this VM has at least 4 GB of RAM.
    - Make sure it has a public IP address.
    - Make sure it is added to your new vNet and create a new Security Group for it.

- Solutions:
  ![](Step-By-Step-Guide/Part%201/Resources/virtual-machine-1.png)
  ![](Step-By-Step-Guide/Part%201/Resources/virtual-machine-networking.png)

## 📦 Step 3 — Download & Configure the Container
In this step, you had to:
- Add your new VM to the Ansible `hosts` file.
- Create a new Ansible playbook to use for your new ELK virtual machine.
    - The header of the Ansible playbook can specify a different group of machines as well as a different remote user (in case you did not use the same admin name):

      ```bash
      - name: Config elk VM with Docker
        hosts: elk
        remote_user: azadmin
        become: true
        tasks:
      ```
    
    - Before you can run the elk container, we need to increase the memory:

    ```yaml
    - name: Use more memory
      sysctl:
        name: vm.max_map_count
        value: '262144'
        state: present
        reload: yes
    ```
    - This is a system requirement for the ELK container. More info [at the `elk-docker` documentation](https://elk-docker.readthedocs.io/#prerequisites).
    - The playbook should then install the following services:
      - `docker.io`
      - `python3-pip`
      - `docker`, which is the Docker Python pip module.

## 🚀 Step 4 — Launch & Expose the Container 

After Docker is installed, download and run the `sebp/elk:761` container.
  - The container should be started with these published ports:
    - `5601:5601` 
    - `9200:9200`
    - `5044:5044`

Your Ansible output should resemble the output below and not contain any errors:

```bash
root@6160a9be360e:/etc/ansible# ansible-playbook elk.yml

PLAY [Configure Elk VM with Docker] ****************************************************

TASK [Gathering Facts] *****************************************************************
ok: [10.1.0.4]

TASK [Install docker.io] ***************************************************************
changed: [10.1.0.4]

TASK [Install python3-pip] *************************************************************
changed: [10.1.0.4]

TASK [Install Docker module] ***********************************************************
changed: [10.1.0.4]

TASK [Increase virtual memory] *********************************************************
changed: [10.1.0.4]

TASK [Increase virtual memory on restart] **********************************************
changed: [10.1.0.4]

TASK [download and launch a docker elk container] **************************************
changed: [10.1.0.4]

TASK [Enable service docker on boot] **************************************
changed: [10.1.0.4]

PLAY RECAP *****************************************************************************
10.1.0.4                   : ok=1    changed=7    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

- SSH from your Ansible container to your ELK machine to verify the connection before you run your playbook.

- After the ELK container is installed, SSH to your container and double check that your `elk-docker` container is running.

Run `sudo docker ps`

```bash
sysadmin@elk:~$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                                              NAMES
842caa422ed8        sebp/elk            "/usr/local/bin/star…"   3 hours ago         Up 3 hours          0.0.0.0:5044->5044/tcp, 0.0.0.0:5601->5601/tcp, 0.0.0.0:9200->9200/tcp, 9300/tcp   elk
sysadmin@elk:~$
```

Solutions:
  - [Ansible Configuration File](Step-By-Step-Guide/Part%201/Resources/ansible.cfg)
  - [Ansible Hosts File](Step-By-Step-Guide/Part%201/Resources/hosts)
  - [ELK Playbook](Step-By-Step-Guide/Part%201/Resources/install-elk.yml)

## 🔑 Step 5 — Identity & Access Management
 
This ELK web server runs on port `5601`. Create an incoming rule for your security group that allows TCP traffic over port `5601` from your IP address.

Verify that you can load the ELK stack server from your browser at `http://[your.VM.IP]:5601/app/kibana`.

Solutions:
Sending traffic to the entire ELK-NET is fine here because there are no other resources besides the ELK server.

![](Step-By-Step-Guide/Part%201/Resources/Security_group1.png)

You can also choose to send traffic _only_ to the ELK server by changing "Virtual Network" to the IP of your ELK Server.

If everything is working correctly, you should see this webpage:

![](Step-By-Step-Guide/Part%201/Resources/Kibana_Home.png)

# 📊 Part 2 — Filebeat & Metricbeat Installation 

On Day 2, you had to install the Filebeat service on your existing ELK server.

Below are the solution configuration files for setting up your Filebeat configuration and playbook:
- [Filebeat Configuration](Step-By-Step-Guide/Part%202/config_files/filebeat-configuration.yml)
- [Filebeat Playbook](Step-By-Step-Guide/Part%202/config_files/filebeat-playbook.yml)

Feel free to complete this project again using these configuration files for additional practice. If you are working from home after working in the classroom, you will need to update the IP addresses to your IP addresses. 

Let's take a closer look at the steps you needed to complete:

## 📥 Step 1 — Install Filebeat on the DVWA Container

First, make sure that our ELK server container is up and running.
- Navigate to http://[your.VM.IP]:5601/app/kibana. Use the public IP address of the ELK server that you created.

- If you do not see the ELK server landing page, open a terminal on your computer and SSH into the ELK server.

  - Run `docker container list -a` to verify that the container is on.

  - If it isn't, run `docker start elk`.

Install Filebeat on your DVWA VM:
- Open your ELK server homepage.
    - Click on **Add Log Data**.
    - Choose **System Logs**.
    - Click on the **DEB** tab under **Getting Started** to view the correct Linux Filebeat installation instructions.

## ⚙️ Step 2 — Create the Filebeat Configuration File

Next, create a Filebeat configuration file and edit this file so that it has the correct settings to work with your ELK server.

Open a terminal and SSH into your jump box:
- Start the Ansible container.
- SSH into the Ansible container.

Copy the provided configuration file for Filebeat to your Ansible container: [Filebeat Configuration File Template](Step-By-Step-Guide/Part%202/config_files/filebeat-configuration.yml).

 - Note that when text is copy and pasted from the web into your terminal, formatting differences are likely to occur that will corrupt this configuration file.

 - Using `curl` is a better way to avoid errors and we have the file hosted for public download [HERE](https://gist.githubusercontent.com/slape/5cc350109583af6cbe577bbcc0710c93/raw/eca603b72586fbe148c11f9c87bf96a63cb25760/Filebeat)

 - Run: `curl https://gist.githubusercontent.com/slape/5cc350109583af6cbe577bbcc0710c93/raw/eca603b72586fbe148c11f9c87bf96a63cb25760/Filebeat >> /etc/ansible/filebeat-config.yml`

 ```bash
root@6160a9be360e:/etc/ansible# curl https://gist.githubusercontent.com/slape/5cc350109583af6cbe577bbcc0710c93/raw/eca603b72586fbe148c11f9c87bf96a63cb25760/Filebeat > filebeat-config.yml
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 73112  100 73112    0     0   964k      0 --:--:-- --:--:-- --:--:--  964k
 ```

Once you have this file on your Ansible container, edit this file as specified in the Filebeat instructions (the specific steps are also detailed below). 

Edit the configuration in this file to match the settings described in the installation instructions for your server.

- **Hint:** Instead of using Ansible to edit individual lines in the `/etc/filebeat/filebeat-config.yml` configuration file, it is easier to keep a copy of the entire configuration file (preconfigured) with your Ansible playbook and use the Ansible `copy` module to copy the preconfigured file into place.

- Because we are connecting your webVM's to the ELK server, we need to edit the file to include your ELK server's IP address. 

  - Note that the default credentials are `elastic:changeme` and should not be changed at this step.

Scroll to line #1106 and replace the IP address with the IP address of your ELK machine.

```bash
output.elasticsearch:
hosts: ["10.1.0.4:9200"]
username: "elastic"
password: "changeme"
```

Scroll to line #1806 and replace the IP address with the IP address of your ELK machine.

```
setup.kibana:
host: "10.1.0.4:5601"
```
Save this file in  `/etc/ansible/files/filebeat-config.yml`.

After you have edited the file, your settings should resemble the below. Your IP address may be different, but all other settings should be the same, including ports.

  ```
  output.elasticsearch:
  hosts: ["10.1.0.4:9200"]
  username: "elastic"
  password: "changeme"

  ...

  setup.kibana:
  host: "10.1.0.4:5601"
  ```

## 📜 Step 3 — Create the Filebeat Installation Play
Create another Ansible playbook that accomplishes the Linux Filebeat installation instructions.

- The playbook should:
  - Download the `.deb` file from [artifacts.elastic.co](https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.4.0-amd64.deb).
  - Install the `.deb` file using the `dpkg` command shown below:
    - `dpkg -i filebeat-7.4.0-amd64.deb`
  - Copy the Filebeat configuration file from your Ansible container to your WebVM's where you just installed Filebeat.
    - You can use the Ansible module `copy` to copy the entire configuration file into the correct place.
    - You will need to place the configuration file in a directory called `files` in your Ansible directory.
  - Run the `filebeat modules enable system` command.
  - Run the `filebeat setup` command.
  - Run the `service filebeat start` command.
  - Enable the Filebeat service on boot.

Solution:

  - [Filebeat Installation Play](Step-By-Step-Guide/Part%202/config_files/filebeat-playbook.yml)

  - After entering your information into the Filebeat configuration file and Ansible playbook, you should have run: `ansible-playbook filebeat-playbook.yml`.


```bash
root@1f08425a2967:/etc/ansible# ansible-playbook filebeat-playbook.yml

PLAY [installing and launching filebeat] *******************************************************

TASK [Gathering Facts] *************************************************************************
ok: [10.0.0.4]
ok: [10.0.0.5]
ok: [10.0.0.6]


TASK [download filebeat deb] *******************************************************************
[WARNING]: Consider using the get_url or uri module rather than running 'curl'.  If you need to
use command because get_url or uri is insufficient you can add 'warn: false' to this command
task or set 'command_warnings=False' in ansible.cfg to get rid of this message.

changed: [10.0.0.4]
changed: [10.0.0.5]
changed: [10.0.0.6]

TASK [install filebeat deb] ********************************************************************
changed: [10.0.0.4]
changed: [10.0.0.5]
changed: [10.0.0.6]

TASK [drop in filebeat.yml] ********************************************************************
ok: [10.0.0.4]
ok: [10.0.0.5]
ok: [10.0.0.6]

TASK [enable and configure system module] ******************************************************
changed: [10.0.0.4]
changed: [10.0.0.5]
changed: [10.0.0.6]

TASK [setup filebeat] **************************************************************************
changed: [10.0.0.4]
changed: [10.0.0.5]
changed: [10.0.0.6]

TASK [start filebeat service] ******************************************************************
[WARNING]: Consider using the service module rather than running 'service'.  If you need to use
command because service is insufficient you can add 'warn: false' to this command task or set
'command_warnings=False' in ansible.cfg to get rid of this message.

changed: [10.0.0.4]
changed: [10.0.0.5]
changed: [10.0.0.6]

TASK [enable service filebeat on boot] **************************************************************************
changed: [10.0.0.4]
changed: [10.0.0.5]
changed: [10.0.0.6]

PLAY RECAP *************************************************************************************
10.0.0.4                  : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
10.0.0.5                  : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
10.0.0.6                   : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
## ✅ Step 4 — Verify Installation & Playbook 

Next, you needed to confirm that the ELK stack was receiving logs. Navigate back to the Filebeat installation page on the ELK server GUI.
- Verify that your playbook is completing Steps 1-4.
- On the same page, scroll to **Step 5: Module Status** and click **Check Data**.
- Scroll to the bottom and click on **Verify Incoming Data**.

Solution:

- If the ELK stack was successfully receiving logs, you would have seen: 

![](Step-By-Step-Guide/Part%202/Images/data_success.png)

## 📈 Step 5 — Create a Play to Install Metricbeat

To update your Ansible playbook to install Metricbeat:

From the homepage of your ELK site:
- Click **Add Metric Data**.
- Click **Docker Metrics**.
- Click the **DEB** tab under **Getting Started** for the correct Linux instructions.

- Download the [Metricbeat `.deb` file](https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.4.0-amd64.deb).

- Use `dpkg` to install the `.deb` file.
- Update and copy the provided [Metricbeat config file](https://gist.githubusercontent.com/slape/58541585cc1886d2e26cd8be557ce04c/raw/0ce2c7e744c54513616966affb5e9d96f5e12f73/metricbeat).
- Run the `metricbeat modules enable docker` command.
- Run the `metricbeat setup` command.
- Run the `metricbeat -e` command.
- Enable the Metricbeat service on boot.

To verify that your play works as expected, on the Metricbeat installation page in the ELK server GUI, scroll to **Step 5: Module Status** and click **Check Data**.

---

# 🗺️ Part 3 — Automated ELK Stack Deployment & Kibana Exploration

## 📦 Automated ELK Stack Deployment

This document contains the following details:
- Description of the Topology
- ELK Configuration
  - Beats in Use
  - Machines Being Monitored
- How to Use the Ansible Build
- Access Policies

### 🗺️ Description of the Topology
This repository includes code defining the infrastructure below. 

![](Step-By-Step-Guide/Part%203/Diagram/Images/Solved.png)

The main purpose of this network is to expose a load-balanced and monitored instance of DVWA, the "D*mn Vulnerable Web Application"

Load balancing ensures that the application will be highly **available**, in addition to restricting **inbound access** to the network. The load balancer ensures that work to process incoming traffic will be shared by both vulnerable web servers. Access controls will ensure that only authorized users — namely, ourselves — will be able to connect in the first place.

Integrating an ELK server allows users to easily monitor the vulnerable VMs for changes to the **file systems of the VMs on the network**, as well as watch **system metrics**, such as CPU usage; attempted SSH logins; `sudo` escalation failures; etc.

The configuration details of each machine may be found below.

| Name     |   Function  | IP Address | Operating System |
|----------|-------------|------------|------------------|
| Jump Box | Gateway     | 10.0.0.4   | Linux            |
| DVWA 1   | Web Server  | 10.0.0.5   | Linux            |
| DVWA 2   | Web Server  | 10.0.0.6   | Linux            |
| ELK      | Monitoring  | 10.0.0.8   | Linux            |

In addition to the above, Azure has provisioned a **load balancer** in front of all machines except for the jump box. The load balancer's targets are organized into the following availability zones:
- **Availability Zone 1**: DVWA 1 + DVWA 2
- **Availability Zone 2**: ELK

### 🖥️ ELK Server Configuration
The ELK VM exposes an Elastic Stack instance. **Docker** is used to download and manage an ELK container.

Rather than configure ELK manually, we opted to develop a reusable Ansible Playbook to accomplish the task. This playbook is duplicated below.


To use this playbook, one must log into the Jump Box, then issue: `ansible-playbook install_elk.yml elk`. This runs the `install_elk.yml` playbook on the `elk` host.

### 🔐 Access Policies
The machines on the internal network are _not_ exposed to the public Internet. 

Only the **jump box** machine can accept connections from the Internet. Access to this machine is only allowed from the IP address `64.72.118.76`
- **Note**: _Your answer will be different!_

Machines _within_ the network can only be accessed by **each other**. The DVWA 1 and DVWA 2 VMs send traffic to the ELK server.

A summary of the access policies in place can be found in the table below.

| Name     | Publicly Accessible | Allowed IP Addresses |
|----------|---------------------|----------------------|
| Jump Box | Yes                 | 64.72.118.76         |
| ELK      | No                  | 10.0.0.1-254         |
| DVWA 1   | No                  | 10.0.0.1-254         |
| DVWA 2   | No                  | 10.0.0.1-254         |

### ⚙️ ELK Configuration

Ansible was used to automate configuration of the ELK machine. No configuration was performed manually, which is advantageous because...

- _TODO: What is the main advantage of automating configuration with Ansible?_

The playbook implements the following tasks:
- _TODO: In 3-5 bullets, explain the steps of the ELK installation play. E.g., install Docker; download image; etc._
- ...
- ...


The following screenshot displays the result of running `docker ps` after successfully configuring the ELK instance.

- _TODO_: Update the image file path with the name of your screenshot of docker ps output:

  ![Docker ps output showing the running ELK container](Step-By-Step-Guide/Part%203/Diagram/Images/docker_ps_output.png)


The playbook is duplicated below.

```yaml
---
# install_elk.yml
- name: Configure Elk VM with Docker
  hosts: elkservers
  remote_user: elk
  become: true
  tasks:
    # Use apt module
    - name: Install docker.io
      apt:
        update_cache: yes
        name: docker.io
        state: present

      # Use apt module
    - name: Install pip3
      apt:
        force_apt_get: yes
        name: python3-pip
        state: present

      # Use pip module
    - name: Install Docker python module
      pip:
        name: docker
        state: present

      # Use command module
    - name: Increase virtual memory
      command: sysctl -w vm.max_map_count=262144

      # Use sysctl module
    - name: Use more memory
      sysctl:
        name: vm.max_map_count
        value: "262144"
        state: present
        reload: yes

      # Use docker_container module
    - name: download and launch a docker elk container
      docker_container:
        name: elk
        image: sebp/elk:761
        state: started
        restart_policy: always
        published_ports:
          - 5601:5601
          - 9200:9200
          - 5044:5044
```

### 🎯 Target Machines & Beats
This ELK server is configured to monitor the DVWA 1 and DVWA 2 VMs, at `10.0.0.5` and `10.0.0.6`, respectively.

We have installed the following Beats on these machines:
- Filebeat
- Metricbeat
- Packetbeat

These Beats allow us to collect the following information from each machine:
- **Filebeat**: Filebeat detects changes to the filesystem. Specifically, we use it to collect Apache logs.
- **Metricbeat**: Metricbeat detects changes in system metrics, such as CPU usage. We use it to detect SSH login attempts, failed `sudo` escalations, and CPU/RAM statistics.
- **Packetbeat**: Packetbeat collects packets that pass through the NIC, similar to Wireshark. We use it to generate a trace of all activity that takes place on the network, in case later forensic analysis should be warranted.

The playbook below installs Metricbeat on the target hosts. The playbook for installing Filebeat is not included, but looks essentially identical — simply replace `metricbeat` with `filebeat`, and it will work as expected.

```yaml
---
- name: Install metric beat
  hosts: webservers
  become: true
  tasks:
    # Use command module
  - name: Download metricbeat
    command: curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.4.0-amd64.deb

    # Use command module
  - name: install metricbeat
    command: dpkg -i metricbeat-7.4.0-amd64.deb

    # Use copy module
  - name: drop in metricbeat config
    copy:
      src: /etc/ansible/files/metricbeat-config.yml
      dest: /etc/metricbeat/metricbeat.yml

    # Use command module
  - name: enable and configure docker module for metric beat
    command: metricbeat modules enable docker

    # Use command module
  - name: setup metric beat
    command: metricbeat setup

    # Use command module
  - name: start metric beat
    command: service metricbeat start
```

### ▶️ Using the Playbooks
In order to use the playbooks, you will need to have an Ansible control node already configured. We use the **jump box** for this purpose.

To use the playbooks, we must perform the following steps:
- Copy the playbooks to the Ansible Control Node
- Run each playbook on the appropriate targets

The easiest way to copy the playbooks is to use Git:

```bash
$ cd /etc/ansible
$ mkdir files
# Clone Repository + IaC Files
$ git clone https://github.com/yourusername/project-1.git
# Move Playbooks and hosts file Into `/etc/ansible`
$ cp project-1/playbooks/* .
$ cp project-1/files/* ./files
```

This copies the playbook files to the correct place.

Next, you must create a `hosts` file to specify which VMs to run each playbook on. Run the commands below:

```bash
$ cd /etc/ansible
$ cat > hosts <<EOF
[webservers]
10.0.0.5
10.0.0.6

[elk]
10.0.0.8
EOF
```

After this, the commands below run the playbook:

 ```bash
 $ cd /etc/ansible
 $ ansible-playbook install_elk.yml elk
 $ ansible-playbook install_filebeat.yml webservers
 $ ansible-playbook install_metricbeat.yml webservers
 ```

To verify success, wait five minutes to give ELK time to start up. 

Then, run: `curl http://10.0.0.8:5601`. This is the address of Kibana. If the installation succeeded, this command should print HTML to the console.


---

## 🔎 Exploring Kibana

1. Start by adding the sample web log data to Kibana.

    - You can import it by clicking **Try our sample data**.

        ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/Welcome.png)

    - Or you can import it from the homepage by clicking on **Load a data set and a Kibana dashboard** under **Add sample data**.

        ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/add-data.png)

    - Click **Add Data** under the **Sample Web Logs** data pane.

        ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/sampledata.png)

    - Click **View Data** to pull up the dashboard.

        ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/view-data.png)

2. Answer the following questions:

    - In the last 7 days, how many unique visitors were located in India?

       - **Example Answer:** 253

        ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/India.png)

        ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/india-unique.png)

    - In the last 24 hours, of the visitors from China, how many were using Mac OSX?

       - **Example Answer:** 7

        ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/china-osx.png)

    - In the last 2 days, what percentage of visitors received 404 errors? How about 503 errors?

        - **Example Answer:** 404: 6.667% and 503: 13.333%

         ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/404-503.png)

    - In the last 7 days, what country produced the majority of the traffic on the website?

        - **Example Answer:** China

          ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/most-traffic.png)

          ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/most-traffic2.png)

    - Of the traffic that's coming from that country, what time of day had the highest amount of activity?

        - **Example Answer:** 12 p.m. and 1 p.m. (hours 12 and 13)

         ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/hour-day.png)

    - List all the types of downloaded files that have been identified for the last 7 days, along with a short description of each file type (use Google if you aren't sure about a particular file type).

        - **Example Answer:**

            - **gz:** `.gz` files are compressed files created using the gzip compression utility.

            - **css:** `.css` files can help define font, size, color, spacing, border and location of HTML information on a webpage. They are downloaded with their `.html` counterparts and rendered by the browser.

            - **zip:** A lossless compression format. A `.zip` file may contain one or more files or directories that have been compressed.

            - **deb:** A file with the `.deb` file extension is a Debian (Linux) Software Package file. These files are installed when using the `apt` package manager.

            - **rpm:** `.rpm` file formats are a Red Hat Software Package file. RPM stands for Red Hat Package Manager.

         ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/files-downloaded.png)

3. Look at the chart that shows Unique Visitors Vs. Average Bytes.

    ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/visitors-vs-bytes.png)

    - Locate the time frame in the last 7 days with the most amount of bytes (activity).

    - In your own words, is there anything that seems potentially strange about this activity?

        **Example Answer:** (Your results may be different.) In our example, it seems strange that _one_ visitor is using a number of bytes that is considerably higher than all other usage.

         ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/outstanding-traffic.png)

4. Filter the data by this event.

     ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/filtered-by-visit.png)

    - What is the timestamp for this event?
      
        - **Example Answer:** The time filter shows Sep 13, 2020 @ 21:00 -> Sep 14, 2020 @ 00:00. The time stamp is 22:55.

     ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/Time-Stamp.png)

     ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/time-stamp2.png)

    - What kind of file was downloaded?

       - **Example Answer:** An RPM file

     ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/file-downloaded.png)
        
    - From what country did this activity originate?

        - **Example Answer:** India

     ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/india2.png)    
        
    - What HTTP response codes were encountered by this visitor?

        - **Example Answer:** 200 OK

     ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/404-png.png)

5. Switch over to the Kibana Discover page to see more details about this activity.

    ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/Discover.png)

    - What is the source IP address of this activity?

        - **Example Answer:** `35.143.166.159`
    
    - What are the geo coordinates of this activity?

        - **Example Answer:** `{ "lat": 43.34121, "lon": -73.6103075 }`

     ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/ip-geo.png)    
    
    - What OS was the source machine running?

        - **Example Answer:** Windows 8
    
    - What is the full URL that was accessed?

        - **Example Answer:** https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-6.3.2-i686.rpm
    
    - From what website did the visitor's traffic originate?

        - **Example Answer:** Facebook

       ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/url-fb.png)

6. Finish your investigation with a short overview of your insights.

    - What do you think the user was doing?

        - **Example Answer:** This event appears to be a user downloading a Linux package from the website being monitored. 

    - Was the file they downloaded malicious? If not, what is the file used for?

        - Linux packages aren't typically malicious, but they could be. Depending on the website, this could be harmless traffic from a sysadmin performing an update.

    - Was there anything that seems suspicious about this activity? 
    - Is any of the traffic you inspected potentially outside of compliance guidelines?

        - **Example Answer:** The main concern is the referral link from Facebook, as it's probably not within compliance to post package update links on Facebook.

        - This user could be further investigated and monitored for suspicious activity.


---

## 🧪 Kibana Continued — Generating & Detecting Activity

### 📋 Scenario 

In this activity, you played the role of a cloud architect and were tasked with setting up an ELK server to gather logs for the Incident Response team.

Before you hand over the server to the IR team, your senior architect has asked that you verify the ELK server is working as expected and pulling both logs and metrics from the pen-testing web servers.

You had three tasks: 

1. Generate a high amount of failed SSH login attempts and verify that Kibana is picking up this activity.

2. Generate a high amount of CPU usage on the pen-testing machines and verify that Kibana picks up this data.

3. Generate a high amount of web requests to your pen-testing servers and make sure that Kibana is picking them up.

### 🔨 SSH Barrage

Task: Generate a high amount of failed SSH login attempts and verify that Kibana is picking up this activity.

<details>
<summary> Solution Guide: SSH Barrage </summary>

<br>

##### SSH Barrage Solutions

1. Start by logging into your jump-box. 

	- Run: `ssh username@ip.of.web.vm`

	- You should receive an error:

		```bash
		sysadmin@Jump-Box-Provisioner:~$ ssh sysadmin@10.0.0.5
		sysadmin@10.0.0.5: Permission denied (publickey).
		```

	- This error was also logged and sent to Kibana. 

2.  Run the failed SSH command in a loop to generate failed login log entries.

    ```bash
    # Creates 1000 login attempts on the 10.0.0.5 server.
    sysadmin@Jump-Box-Provisioner:~$ for i in {1..1000}; do ssh sysadmin@10.0.0.5; done
    ```

    - Syntax Breakdown:
        - `for` begins the for loop.
        - `i in` creates a variable named `i` that will hold each number `in` our list.
        - `{1..1000}` creates a list of 1000 numbers, each of which will be given to our `i` variable.
        - `;` separates the portions of our `for` loop when written on one line.
        - `do` indicates the action taken each loop.
        - `ssh sysadmin@10.0.0.5` is the command run by `do`.
        - `;` separates the portions of our for loop when it's written on one line.
        - `done` closes the `for` loop.


3. Search through the logs in Kibana to locate your generated failed login attempts.

    ```bash
    # IMPORTANT: This loop will continue to run until you stop it using `CTRL + C` 
    # It will create thousands of login attempts on the 10.0.0.5 server.
    sysadmin@Jump-Box-Provisioner:~$ while true; do ssh sysadmin@10.0.0.5; done
    ```

    - Syntax Breakdown:
        - `while` begins the while loop.
        - `true` will always be equal to `true` so this loop will never stop, unless you force quit it.
        - `;` separates the portions of our `while` loop when it's written on one line.
        - `do` indicates the action taken each loop.
        - `ssh sysadmin@10.0.0.5` is the command run by `do`.
        - `;` separates the portions of our `for` loop when it's written on one line.
        - `done` closes the `for` loop.

    - Search through the logs in Kibana to locate your generated failed login attempts.

    - You should find a section of logs that look like this:

        ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/Log-Auth.png)

4. **Bonus**: Create a nested loop that generates SSH login attempts across all 3 of your VM's.


    ```bash
    sysadmin@Jump-Box-Provisioner:~$ while true; do for i in {5..7}; do ssh sysadmin@10.0.0.$i; done; done
    ```
     - **Important:** This loop will continue to run until you stop it using `CTRL + C` and will create thousands of login attempts.

     - Syntax Breakdown:
        - `i in` creates a variable named `i` that will hold each number `in` our list.
        - `{5..7}` creates a list of numbers (5, 6 and 7), each of which will be given to our `i` variable.
        - `do` indicates the action taken each loop.
        - `ssh sysadmin@10.0.0.$i` is the command run by `do`. It is passing in the `$i` variable so the `wget` command will be run on each server.


     - Note that the brace expansion (`{5..7}`) will only work if the IP addresses of your servers end in `5`, `6`, or `7`. If their IP numbers are not in sequence, we can list them explicitly:

	```bash
	# Note `for i in 5 8 12`
	sysadmin@Jump-Box-Provisioner:~$ while true; do for i in 5 8 12; do ssh sysadmin@10.0.0.$i; done; done
	```

    - **Note**: This loop will continue to run until you stop it using `CTRL + C` and will create thousands of login attempts.


</details>

### 🔥 Linux Stress

Task: Generate a high amount of CPU usage on the pen-testing machines and verify that Kibana picks up this data.

<details>

<summary> Solution Guide: Linux Stress </summary>

<br>

#### Solutions

1. From your Jump-Box, start up your `Ansible` container and attach to it.

        ```bash
        $ sudo docker container list -a   # obtain the container name

        $ sudo docker start container_name

        $ sudo docker attach container_name
        ```

2. SSH from your Ansible container to one of your WebVM's.

        ```bash
        $ ssh username@ip.of.web.vm
        ```

3. Run `sudo apt install stress` to install the stress program.

4. Run `sudo stress --cpu 1` and allow stress to run for a few minutes. 

5. View the Metrics page for that VM in Kibana. 

    - Are you able to see the CPU usage increase?

    - **Answer:** Yes. Leave the stress test running and continue to refresh the Kibana metrics page for that VM. You should see a jump in CPU usage similar to the below image:

        ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/CPU-Spike.png)


6. Run the `stress` program on all three of your VMs and take screen shots of the data generated on the metrics page of Kibana.

    - **Solution:** You should be able to create screen shots similar to the below:

        ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/VM-cpu-1.png)

        ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/VM-CPU-2.png)


</details>


### 🌊 wget-DoS

Task: Generate a high amount of web requests to your pen-testing servers and make sure that Kibana is picking them up.

<details>

<summary> Solution Guide: wget-DoS </summary>


1. Login to your Jump-Box

2. Run `wget ip.of.web.vm`

	```bash
	sysadmin@Jump-Box-Provisioner:~$ wget 10.0.0.5
	--2020-05-08 15:44:00--  http://10.0.0.5/
	Connecting to 10.0.0.5:80... connected.
	HTTP request sent, awaiting response... 302 Found
	Location: login.php [following]
	--2020-05-08 15:44:00--  http://10.0.0.5/login.php
	Reusing existing connection to 10.0.0.5:80.
	HTTP request sent, awaiting response... 200 OK
	Length: 1523 (1.5K) [text/html]
	Saving to: ‘index.html’

	index.html            100%[=======================>]   1.49K  --.-KB/s    in 0s      

	2020-05-08 15:44:00 (179 MB/s) - ‘index.html’ saved [1523/1523]
	```

3. Run `ls` to view the file you downloaded from your web vm to your jump-box:

	```bash
	sysadmin@Jump-Box-Provisioner:~$ ls
	index.html
	```

4. Run the `wget` command in a loop to generate a ton of web requests.
	
	- You can use a bash `for` or `while` loop, directly on the command line, just as you did with the `SSH` command.

	- **for loop:**

		```bash
		sysadmin@Jump-Box-Provisioner:~$ for i in {1..1000}; do wget 10.0.0.5; done
		```
		**Important:** This loop will create 1000 web requests on the 10.0.0.5 server and 1000 downloaded files on your jump-box.

		Syntax Breakdown:
		- `{1..1000}` creates a list of 1000 numbers, each of which will be given to our `i` variable.
		- `;` separates the portions of our `for` loop when it's written on one line.
		- `do` is what each iteration of the loop will do.
		- `do wget 10.0.0.5` is the command run by with each loop.


	- **while loop:**

		```bash
		sysadmin@Jump-Box-Provisioner:~$ while true; do wget 10.0.0.5; done
		```

		**Important:** This loop will continue to run until you stop it using `CTRL + C` and will create thousands of web requests on the 10.0.0.5 server as well as files on your jump-box.


5. Open the `Metrics` page for the web machine you attacked and answer the following questions:
	
	- Which of the VM Metrics was affected the most from this traffic?

	- **Answer:** The Load and Networking Metrics were hit:

	  ![](Step-By-Step-Guide/Part%203/Exploring-Kibana/Images/load-net.png)

6. **Bonus**: Notice that your `wget` loop creates a lot of duplicate files on your jump-box.

	-  Write a command to delete _all_ of these files at once.

		```bash
		sysadmin@Jump-Box-Provisioner:~$ rm *
		```

	-  Find a way to run the `wget` command without generating these extra files.
		- Look up the flag options for `wget` and find the flag that lets you choose a location to save the file it downloads. 

		**Answer:** From the man pages:

		```bash
			-O file
			--output-document=file
					The documents will not be written to the appropriate files, but all will
					be concatenated together and written to file.  If - is used as file,
					documents will be printed to standard output, disabling link conversion.
					(Use ./- to print to a file literally named -.)
		```

	- Save that file to the linux directory known as the 'void' or the directory that doesn't save anything.

		- **Answer:** The directory known as the 'void' that doesn't save anything is `/dev/null`

		- Full command: `while true; do wget 10.0.0.5 -O /dev/null; done`


7. **Bonus**:  Write a nested loop that sends your `wget` command to all 3 of your web VM's over and over.

	```bash
	sysadmin@Jump-Box-Provisioner:~$ while true; do for i in {5..7}; do wget -O /dev/null 10.0.0.$i; done; done
	```

	- **Important:** This loop will continue to run until you stop it using `CTRL + C` and will create thousands of web requests.

	- Syntax Breakdown:
		- `i in` create a variable named `i` that will hold each number `in` our list.
		- `{5..7}` creates a list of numbers, (5, 6 and 7) each of which will be given to our `i` variable.
		- `;` separates the portions of our `for` loop when it's written on one line.
		- `do wget 10.0.0.$i` is the command run by `do`. Notice that here we are passing in our `$i` variable so the `wget` command will be run on each server.
	

	- Or:

		```bash
		sysadmin@Jump-Box-Provisioner:~$ while true; do for i in 5 8 12; do wget -O /dev/null 10.0.0.$i; done; done
		```

	- **Important:** This loop will continue to run until you stop it using `CTRL + C` and will create thousands of web requests.

</details>


