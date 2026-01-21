# Elastic Stack Lab version 1

# **1. Lab Resources & Environment** 💻

To ensure a stable and high-performance environment, this lab uses the following specifications. We are using **Elastic Stack version 8.19.**

### **Essential Components**

| **Component** | **Version** | **Role** |
| --- | --- | --- |
| **Elasticsearch** | 8.19 | Core Search & Analytics Engine |
| **Kibana** | 8.19 | Visualization & Management UI |
| **Fleet Server** | 8.19 | Centralized Agent Management |
| **Logstash** | 8.19 | Custom Data Processing Pipeline |

### **Virtualization & OS**

- **Hypervisor:** VMware Workstation 17 Pro
- **Base OS:** Ubuntu 22.04 LTS
- **Network Mode:** NAT

### **System Sources (Telemetry Data)**

These are the machines we are actively monitoring in this lab:

- **Windows Server 2019:** Active Directory & Event Logs.
- **Ubuntu Server 22.04:** System logs, auth logs, and performance metrics.
- **FortiGate Firewall (v6.x):** Network traffic and security events.

> 🚀 Upcoming Integrations:
> 
> 
> This project is designed to grow. Future updates will include:
> 
> - **Network:** Cisco Switches, Palo Alto Firewalls.
> - **Security:** Tripwire (FIM), Wallix (PAM).
> - **Cloud:** Microsoft Azure monitoring.
> - **Linux:** Various distributions (Debian, CentOS), and more …
> 
> So Follow me and waits for the upcaming 
> 

---

# **2. Lab Architecture** 🏗️

The diagram below illustrates the flow of data from our sources to the centralized Elastic Stack.

### **Core Data Pipelines** 📡

- **🛡️ Elastic Agent (Endpoint Telemetry)**
    - **Role:** Installed directly on endpoints (Windows Server, Ubuntu).
    - **Mechanism:** Securely ships logs and metrics (System, Auth, Sysmon) directly to the **Fleet Server**. This allows for centralized policy management without requiring manual configuration on individual endpoints.
- **🎮 Fleet Server (Control Plane & Listener)**
    - **Role:** Acts as the central traffic controller for all Elastic Agents.
    - **Dual Function:** In this architecture, the Fleet Server also functions as a **Syslog Listener** for agentless integrations (such as the FortiGate Firewall), allowing us to ingest logs from systems where installing an agent is not possible.
- **🏭 Logstash (Transformation Engine)**
    - **Role:** Reserved for complex data transformation and legacy systems.
    - **Use Case:** While Fleet handles standard integrations, Logstash is deployed here to parse unstructured data using custom **Grok filters** before indexing the clean data into Elasticsearch.

---

### **💡 Architectural Insight: Lab Strategy vs. Production Best Practices**

> 🧪 Lab Strategy (Version 1 - Standard)
> 
> 
> In this initial version, we have consolidated the core **ELK components** (Elasticsearch, Logstash, Kibana) onto a single machine (**VM 1**).
> 
> - **Fleet Separation:** However, we have intentionally deployed the **Fleet Server** on a **separate VM** (VM 2).
> - **Why?** This architecture highlights the evolution from the traditional "ELK Stack" to the modern "Elastic Stack." By isolating the Fleet Server, we demonstrate how modern ingestion is decoupled from the storage layer and the positive value added with the integrations & the fleet .
> 
> **🛑 Production Best Practice (Target for Version 2)**
> 
> While Version 1 focuses on functional learning, **Version 2** will simulate a true **Enterprise Cluster** adhering to production standards:
> 
> - **Clustering:** Elasticsearch nodes will be distributed across separate VMs for High Availability (HA) and Load Balancing.
> - **Dedicated Ingestion Node:** We plan to co-locate **Fleet Server** and **Logstash** on a specific "Ingestion VM." This centralizes all "Listening" and "Parsing" activities in one place (a **DMZ Concentrator** model), securing the core data cluster from direct external traffic.

---

![My Elastic Stack SIEM ](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/DG1.svg)

My Elastic Stack SIEM 

---

# 3. 🛠️ steps taked for the installation of Elastic Stack

### ✅ Pre-Requirement :

Before starting, ensure your Ubuntu VM has the following minimum resources allocated in VMware:

- **CPU:** 2 Cores (Minimum per VM; more cores are recommended).
- **RAM:** 8 GB to 12 GB (Recommended for a smooth experience).
    
    > *Note:* 
    *While 4 GB may work, performance will be sluggish. If using low RAM, it is preferred to limit the Java Heap size.*
    > 
- **Storage:** 150 GBof free space at least for running ELK, fleet server, and the source systems (SSD preferred for faster indexing).

**VM 1 Configuration:** This VM will host Kibana, Elasticsearch, and Logstash.

Initial System Configuration

After installing Ubuntu Server on your virtualization tool (VMware Workstation is used in this guide), configure the following points:

> Note: i have is configured the network using NAT with the subnet 192.168.12.0/24 and the gateway 192.168.12.2.
> 
1. Setting up the Hostname
    
    ```bash
    sudo hostnamectl set-hostname ELK-server # Replace 'ELK-server' with your desired hostname
    ```
    
    Next, update the hosts file:
    
    ```bash
    sudo nano /etc/hosts
    # Locate the line containing 127.0.1.1 and replace the old hostname with the new one.
    ```
    
    Reboot to apply changes:
    
    ```bash
    sudo reboot
    ```
    
2. Installing and Activating SSH
    
    Using an SSH session is more practical than working directly in the VMware CLI, as it offers flexibility for copy-pasting and scrolling. I recommend using **MobaXterm**.
    
    > note : it’s possible that you have install it with steps of initial the OS ubuntu server, so check if you have befor starting on install it.
    > 
    
    ```bash
    sudo apt update
    sudo apt install openssh-server
    sudo systemctl enable ssh
    ```
    
    Also, perform a system upgrade; it’s recommended:
    
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```
    
    `-y`  for tell yes for all confirmations needed.
    
3. Assigning a Static IP Address
    
    > NB :
    > 
    > 
    > We must configure this before the next restart; otherwise, `cloud-init` may revert the network configuration to DHCP.
    > 
    > First, disable cloud-init's network capabilities:
    > 
    > ```bash
    > sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
    > # Add the following line:
    > network: {config: disabled}
    > ```
    > 
    > Then you can modify the configuration file : 
    > 
    > ```bash
    > sudo nano /etc/netplan/50-cloud-init.yaml
    > ```
    > 
    
    *adjust interface name `ens33` and IPs as needed*
    
    ```yaml
    network:
      version: 2
      renderer: networkd
      ethernets:
        ens33:
          dhcp4: no
          addresses:
            - 192.168.12.10/24 # Your desired Static IP
          routes:
            - to: default
              via: 192.168.12.2 # Your Gateway IP
          nameservers:
            addresses: [8.8.8.8, 1.1.1.1]
    ```
    
    Check and apply the configuration:
    
    ```bash
    cat /etc/netplan/01-cloud-init.yaml
    ```
    
    ```bash
    sudo netplan apply
    ip a
    ```
    
    - Then check in the next restart if the ip address is still the same as you configured it.
4. Installing Java
    
    Currently, **OpenJDK 17 is** sufficiently compatible with Elastic Stack components.
    
    ```bash
    sudo apt update
    sudo apt install openjdk-17-jdk -y
    java -version
    ```
    

---

## 1️⃣ Installation of Elasticsearch

*We are working on VM 1, where all components will be installed.*

1. Add the Elasticsearch Repository and GPG Key
    
    ```bash
    wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
    sudo apt-get install apt-transport-https
    echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
    sudo apt update
    ```
    
    ### explaination
    
    ## 🔹 Command 1
    
    ```bash
    wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
    
    ```
    
    ### 🔍 What this command does (deep explanation)
    
    This command **downloads and installs the official GPG signing key** used by Elastic to sign its APT packages.
    
    ### Breakdown:
    
    - `wget`
        
        → A command-line tool used to download files from the web.
        
    - `q`
        
        → Quiet mode (no output shown).
        
        This is useful for scripting and automation.
        
    - `O -`
        
        → Writes the downloaded content to **standard output (stdout)** instead of saving it to a file.
        
    - `https://artifacts.elastic.co/GPG-KEY-elasticsearch`
        
        → The **official public GPG key** used by Elastic to cryptographically sign their packages.
        
    - `|` (pipe)
        
        → Redirects the output of `wget` directly into the next command.
        
    - `sudo gpg --dearmor`
        
        → Converts the ASCII-armored GPG key into a **binary format** (`.gpg`), which APT requires.
        
    - `o /usr/share/keyrings/elasticsearch-keyring.gpg`
        
        → Saves the converted key in the system keyring directory.
        
    
    ### ✅ Why this step is mandatory
    
    APT will **refuse to install packages** from a repository unless their signatures can be verified.
    
    This step ensures:
    
    - Package **authenticity**
    - Package **integrity**
    - Protection against **tampered or malicious repositories**
    
    ---
    
    ## 🔹 Command 2
    
    ```bash
    sudo apt-get install apt-transport-https
    
    ```
    
    ### 🔍 What this command does
    
    This installs support for downloading APT packages over **HTTPS**.
    
    ### Explanation:
    
    - `apt-transport-https`
        
        → Enables secure package downloads via HTTPS.
        
    
    ### ⚠️ Important note (shows mastery)
    
    - On **modern Ubuntu versions (20.04+)**, this package is often installed by default.
    - It is still included for **backward compatibility** and **best practice**, especially in enterprise documentation.
    
    ---
    
    ## 🔹 Command 3
    
    ```bash
    echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
    
    ```
    
    ### 🔍 What this command does
    
    This command **adds the Elastic APT repository** to the system.
    
    ### Breakdown:
    
    - `echo "deb ..."`
        
        → Outputs a new APT repository definition.
        
    - `signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg`
        
        → Restricts trust to **only this specific GPG key**, improving security.
        
        (Best practice recommended by Debian/Ubuntu.)
        
    - `https://artifacts.elastic.co/packages/8.x/apt`
        
        → Official Elastic package repository for version **8.x**.
        
    - `stable main`
        
        → Specifies the distribution and repository section.
        
    - `| sudo tee /etc/apt/sources.list.d/elastic-8.x.list`
        
        → Writes the repository configuration into a **dedicated file**.
        
        This is cleaner and more maintainable than editing `/etc/apt/sources.list`.
        
    
    ### ✅ Why this matters
    
    This step tells APT:
    
    > “Elastic packages exist here, trust them using this exact key.”
    > 
    
    ---
    
    ## 🔹 Command 4
    
    ```bash
    sudo apt update
    
    ```
    
    ### 🔍 What this command does
    
    This refreshes the local package index.
    
    ### Explanation:
    
    - APT contacts all configured repositories
    - Downloads the **latest metadata**
    - Makes Elastic packages (`elasticsearch`, `kibana`, `logstash`) available for installation
    
    Without this step, APT would not recognize the newly added repository.
    
    > The Elastic APT repository configuration and GPG key installation must be performed on every machine where Elastic components are installed.
    > 
    > 
    > If multiple components (Elasticsearch, Kibana, Logstash) are installed on the **same host**, this step is required **only once**.
    > 
    > However, in a **distributed architecture** where each component is deployed on a separate machine, the repository configuration must be repeated **independently on each host**.
    > 
    
2. Install **Elasticsearch**
    
    ```bash
    sudo apt install elasticsearch -y
    ```
    
3. Enable and Start Elasticsearch
    
    ```bash
    # for enable that elasticsearch service start with the system
    sudo /bin/systemctl daemon-reload
    sudo /bin/systemctl enable elasticsearch.service
    ```
    
    ```bash
    # then start the service
    sudo systemctl start elasticsearch.service
    # this for cheking if the service hasbeen started
    sudo systemctl status elasticsearch.service
    ```
    
    - **Password Setup:** To change the `elastic` superuser (`-u`), password interactively (`-i`),:
    
    ```bash
    /usr/share/elasticsearch/bin/elasticsearch-reset-password -i -u elastic
    ```
    
    > Note: If you omit the -i flag, a random password will be generated automatically.
    > 
4. **Configure `elasticsearch.yml`**
    
    First, create a backup of the configuration file:
    
    ```bash
    cp /etc/elasticsearch/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml.backup
    ```
    
    Edit the file:
    
    ```bash
    sudo nano /etc/elasticsearch/elasticsearch.yml
    ```
    
    Ensure the following settings are active in the network section:
    
    ```yaml
    #-------------------- Network ------------------------
    network.host: 192.168.12.10 # Allows access from network IPs, not just localhost
    http.port: 9200
    ```
    
    > Note:
    > 
    > - If you encounter communication issues between Kibana and Elasticsearch, you may use 0.0.0.0 as the host, but any way if you have nicely configured your netwowrk it’s should work with the specifique ip address
    > - Other parameters, including SSL/TLS communication settings, may already be present and enabled, as the Elastic Stack adopts a secure-by-default configuration model.
    
    **Firewall Configuration:** Allow port 9200 (TCP/UDP):
    
    ```bash
    sudo ufw allow 9200/tcp
    sudo ufw allow 9200/udp
    sudo ufw reload
    ```
    
    Restart the service to apply changes
    
    ```bash
    
    sudo systemctl restart elasticsearch.service
    sudo systemctl status elasticsearch.service # check if the service corectly started
    ```
    
5. Verification
    
    ```bash
    curl -k -u elastic https://192.168.12.10:9200
    ```
    
    *You will be prompted to enter the password generated during the installation.*
    
    **Expected Result:**
    
    If the installation is successful, the server will respond with a JSON object similar to this:
    
    JSON
    
    ```json
    {
      "name" : "node-1",
      "cluster_name" : "elasticsearch",
      "cluster_uuid" : "Fv5_ABCD1234...",
      "version" : {
        "number" : "8.19.0",
        "build_flavor" : "default",
        "build_type" : "deb",
        "lucene_version" : "9.x.x",
        "minimum_wire_compatibility_version" : "7.17.0",
        "minimum_index_compatibility_version" : "7.0.0"
      },
      "tagline" : "You Know, for Search"
    }
    ```
    

---

## 2️⃣ Installing Kibana

1. Install and Start Service
    
    ```bash
    sudo apt install kibana -y
    sudo systemctl daemon-reload
    sudo systemctl enable kibana.service
    sudo systemctl start kibana.service
    ```
    

> Note: Kibana cannot start if Elasticsearch is not running or if communication is blocked.
> 
1. Configure `kibana.yml`
    
    Backup the configuration file:
    
    ```bash
    cp /etc/kibana/kibana.yml /etc/kibana/kibana.yml.backup
    ```
    
    Then check if these two lines are activated in `kibana.yml` :
    
    ```yaml
    server.port: 5601
    server.host: "192.168.12.10" # Enables remote access
    ```
    
2. Security Configuration (Keys & Token)
    
    Generate encryption keys:
    
    ```bash
    cd /usr/share/kibana/bin
    sudo ./kibana-encryption-keys generate # this command will generatte 3 encryption keys that  we will ad it to the keystore
    ```
    
    Add the three generated keys to the Kibana keystore
    A*fter typing these commands, you will be asked to enter the kys generated by the previeurse cammand:*
    
    ```bash
    ./kibana-keystore add xpack.encryptedSavedObjects.encryptionKey 
    ./kibana-keystore add xpack.reporting.encryptionKey
    ./kibana-keystore add xpack.security.encryptionKey
    ```
    
    **Firewall Configuration:** Allow port 5601:
    
    ```bash
    sudo ufw allow 5601/tcp
    ```
    
    Restart Kibana:
    
    ```bash
    sudo systemctl restart kibana.service
    sudo systemctl status kibana.service # check if it's has been good startted
    ```
    
- So now you can git into the Web interface.
by  taping in the browser this url→ `http://192.168.12.10:5601`
- you have now generate the token for access, (we wil work by token enrollement instead of certifactoin .crt for secure communication between this tow srvices.)
    
    first reset the password of the kibana_system user : 
    `/usr/share/elasticsearch/bin/elasticsearch-reset-password -i -u kibana_system`, and do it also for elastic user. 
    
    Then    
    
    ```bash
    /usr/share/elasticsearch/bin
    # the run this command
    sudo ./elasticsearch-create-enrollment-token -- scope kibana
    # its will output for you an token that you will use it for enrollement.
    ```
    
    - In the web interface, enter the enrollment token, then you will be askedto neter an OTP code ( that enforces the security process) wich you can generate it by this commandes :
        
        
        and then you will be asked to log in with your credentials of   `elastic` user
        
        ```bash
        cd /usr/share/kibana/bin/
        ./kibana-verification-code # wich will generate a code of sex numbers
        ```
        
    

> after you get into the kibana interface you will see a wornning notification pup up. you will fixe it by set a public address for kibana in the yml file
> 
> 
> ![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image.png)
> 
> ```yaml
> server.publicBaseUrl: "http://192.168.12.10:5601"
> ```
> 

⇒ So from now you can discover the Kibana UI however you want,

- you find as mentioned on the home page divided into 3 principal sections: analytics, observability, security, and the management section, where we will manage the fleet and do other things.

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%201.png)

# 3️⃣ Fleet Server & Logstash

## Fleet

In this section, we describe the installation and configuration of the **Fleet Server**, the central component responsible for managing Elastic Agents and collecting logs from source systems.

---

## 1. Installing Fleet Server

The installation of the **Fleet Server** is straightforward and fully guided through the **Kibana** interface.

First, navigate to the **Fleet** section in Kibana and click on **Add Fleet Server**.

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%202.png)

You will be prompted to provide:

- The **Fleet Server name** (in our case: `fleet-server`)
- The **Fleet Server URL**, which must include such i have do  `https://192.168.12.12:8220`:
    - the `https` protocol
    - the server IP address.
    - the configured listening port `8220`
    
    ![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%203.png)
    

Once this information is validated, Kibana automatically generates a set of commands to be executed on the target machine that will host the Fleet Server.

It is important to select the command corresponding to the **operating system** in use.

In our architecture, the Fleet Server is deployed on a **dedicated and isolated Ubuntu machine**, ensuring better stability and security, on next versions we will install it with logstash on the same VM as it the best practice for a centrilzed listenning placement ( for example in case of new system source inagration added or the other since if an integration do not respond to your needs and you wants to come back into the custom way with logstash pipline, instead of change the syslog destination on all the system source you can handl it  only from kibana interface becauseit’s will still send the logs to the same machin so you will need to define only in this machine will listenning to is logs fleet or logstash  ).

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%204.png)

- After successfully executing the commands, a ✅ symbol appears in the Kibana interface, confirming that communication between the Fleet Server and the Elastic Stack has been successfully established.

---

## 2. Installing Fleet Agent

To install an **Elastic Agent** on a source system, click on **Add agents** next to **Add Fleet Server**.

Then, create a new **agent policy** and assign it a meaningful name.

Keep the first option (in section 2) enabled to allow **agent enrollment via the Fleet Server**, which provides centralized management and automatic updates.

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%205.png)

Next, select the target operating system.

In our case, the agent is installed on a **Windows Server** so we will chose windows.

> **Note:** To temporarily bypass certificate verification during installation (lab environment), add the `--insecure` option at the end of the last command. Example:
> 

```
.\elastic-agent.exe install --url=https://192.168.12.12:8220 --enrollment-token=RUViY25ab0JHQ1dMOUJJZTdQLUw6VGI3YlplLTBiR29KRUhWOVBqMG5Ldw== --insecure
```

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%206.png)

As with the Fleet Server installation, a visual confirmation ✅ appears when the agent installation is successful.

The agent then becomes visible in the Fleet interface, displaying detailed status information.

In our case, since the **WD Server is powered off in that moment**, the agent appears with an **offline** status.

> Notifications may also indicate the availability of **policy updates**. These updates must be applied from the Fleet interface and will then be automatically deployed by the Fleet Server.
> 

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%207.png)

Fleet Servers and agents can be monitored in more detail using **prebuilt dashboards**, which provide a global overview of their status and performance.

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%208.png)

**Note:**

Additional dashboards are available in the **Dashboards** section, including:

- **Analytics**: a wide collection of generic dashboards
- **Security**: dashboards focused on security monitoring and detection

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%209.png)

In addition to existing dashboards, **custom dashboards** can be created using the dedicated button in the top-left corner of the interface.

This approach is particularly useful for:

- non-standard data sources
- logs that are not automatically normalized
- specific monitoring or detection requirements

It requires a solid understanding of log structure in order to build meaningful visualizations.

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%2010.png)

---

## 3. Integrations

> **Note:** The more integrations are added, the more ready-to-use dashboards become available for analysis and visualization.
> 

---

## 1. Use Case 1: Source Systems with Elastic Agent

An integration can be added from two different locations within Kibana.

Click on **Add integration**, search for the desired integration, and select it.
or you can go to te sepicicfic agentpolicy that you want and you find a button for adding integration your need.

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%2011.png)

In our case, we search for example **Windows** integration.

As shown, the integration is already associated with an agent policy. A notification also indicates that an update is available and must be applied from the Fleet interface.

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%2012.png)

> After clicking **Add Windows**, you must define:
> 
> - the types of logs to collect
> - the log sources
> 
> In our use case, special attention is given to **Sysmon**. enabled it if not , while the other options are kept at their default values.
> 

Once this step is completed, Windows logs are properly normalized, indexed, and ready for analysis.

Verification can be performed from the **Discover** section by selecting the appropriate data view (e.g., `logs-*`) and using KQL queries.

Examples:

```
source.ip: "192.168.12.128"
```

or Displays all logs originating from the WD Servers.

```
host.os.name: "Windows Server 2019 Standard Evaluation"
```

Displays logs generated by systems running this operating system.

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%2013.png)

> **Note:** Logs are structured as fields and values, which greatly facilitates search and correlation.
> 

---

## 2. Use Case 2: Source Systems Without Elastic Agent

We take the **FortiGate firewall** as the first example. Additional devices will be integrated later, God willing.

Some devices, especially network equipment, do not support Elastic Agent installation.

In such cases, a specific configuration is required.

First, add the **FortiGate integration** to the Fleet Server policy.

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%2014.png)

The Fleet Server then acts as a **Syslog listener**, configured on port `9004` with the all address by puting `0.0.0.0` in case of if you have only one equipement on the same port you can add his ip address for minimise the logs gathered.

After saving the configuration, proceed with the FortiGate firewall configuration.

Parameters used:

- Fleet Server IP: `192.168.12.12`
- Port: `9004`

FortiGate CLI configuration:

```
config log syslogd setting
    set status enable
    set server "192.168.12.12"
    set mode udp
    set port 9004
    set format default
end

```

Once the configuration is completed, logs should be successfully transmitted to the Elastic Stack.

In case of issues, it is recommended to verify the entire pipeline:

- FortiGate → Fleet Server
- Fleet Server → Elasticsearch
- Elasticsearch → Kibana

It is also possible that logs are correctly indexed but not visible in Kibana due to the absence of an appropriate **data view**.

The dashboards provided by the integration offer a global overview.

![Capture d'écran 2026-01-07 230544.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/Capture_dcran_2026-01-07_230544.png)

In our case, they display authentication failures, without advanced detection rules at this stage.

# Logstash Configuration

### 1. Installing Logstash

In this lab, Logstash is installed on the same virtual machine as Elasticsearch and Kibana (VM1).

```bash
sudo apt install logstash -y
```

Enable and start the Logstash service:

```bash
sudo systemctl enable logstash.service
sudo systemctl start logstash.service
```

Verify that the service is running:

```bash
sudo systemctl status logstash.service
```

### 2. Configuring a Logstash Pipeline

This section demonstrates the complexity of log ingestion **before native Elastic integrations were available**.

Prior to Elastic Stack version **8.11**, there was **no native FortiGate integration**, which required the manual creation of Logstash pipelines.

This situation still applies to other products such as **Tripwire**, **Wallix**, and various legacy systems.

As a result, a dedicated pipeline configuration file must be created **for each device type**.

Below is an example for a FortiGate firewall:

```
/etc/logstash/conf.d/fortigate-pipeline.conf
```

Each pipeline follows the standard **input → filter → output** structure.

---

### 3. FortiGate Logstash Pipeline Example

### Input Stage

Receives syslog messages from the FortiGate firewall over UDP.

```
input {
  udp {
    host => "192.168.12.10"
    port => 5144   # FortiGate Syslog Port
  }
}
```

### Filter Stage

Parses, normalizes, and enriches the log data.

```
filter {
  grok {
    match => { "message" => "%{SYSLOG5424PRI}%{GREEDYDATA:message}" }
    overwrite => [ "message" ]
  }

  mutate {
    remove_field => ["@timestamp", "path", "host", "@version", "log", "event"]
  }

  kv {
    field_split => " "
  }

  mutate {
    remove_field => ["message"]
    add_field => { "logdate" => "%{date} %{time}" }
  }

  date {
    match => [ "logdate", "yyyy-MM-dd HH:mm:ss" ]
    target => "@timestamp"
  }

  mutate {
    remove_field => ["logdate", "date", "time"]
    convert => {
      "rcvdbyte" => "integer"
      "sentbyte" => "integer"
    }
  }
}
```

### Output Stage

Sends processed logs to Elasticsearch and prints them to the console for debugging.

```yaml
output {
  stdout { # used for see the output on the cli for verfication only
    codec => rubydebug
  }

  elasticsearch { #this is the real destination
    hosts => ["https://192.168.12.10"]
    index => "firewall-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "********"
    ssl => true
    cacert => "/etc/logstash/http_ca.crt"
  }
}
```

---

### 4. Applying the Pipeline

After creating or modifying the pipeline configuration file, restart Logstash to apply the changes:

```bash
sudo systemctl restart logstash
```

If all steps are configured correctly, FortiGate logs will begin appearing in **Kibana**, structured according to the fields defined in the Logstash pipeline.

---

---

---

# 4️⃣ Testing the ELK SIEM

To validate that the SIEM is functioning correctly, the following steps were performed:

1. **Define detection rules**
2. **Simulate an attack**
3. **Verify alert generation**

### 1. Detection Rules

Detection rules are configured in:

```
Security → Detection rules (SIEM)
```

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%2015.png)

Elastic provides multiple methods for defining rules:

- Custom query
- Threshold-based rules
- Machine Learning rules
- Event correlation rules
- Imported rules
- Pre-built rules

Custom rules can be written using **event correlation logic** or KQL/DSL queries.

### 2. Rule Implementation

For this project, three detection rules were used:

1. **Custom rule** (manually created)
    - Triggers an alert for every failed login attempt on the field server.
2. **Pre-built SSH brute-force rule**
    - Detects repeated failed SSH authentication attempts.
3. **Privileged account brute-force rule**
    - Detects brute-force attempts targeting privileged accounts such as `root` or administrators.

---

### 3. Attack Simulation

To test the detection logic, a **brute-force attack** was simulated using the **Hydra** tool.

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%2016.png)

The resulting alerts confirmed that log ingestion, parsing, correlation, and alerting were functioning as expected.

![image.png](Elastic%20Stack%20Lab%20version%201%20(2%20most%20organized)/image%2017.png)

---

## Future Enhancements

Planned improvements for this project include:

- Integration with additional security tools and log sources
- Automated response actions (SOAR)
- Enhanced correlation rules across multiple systems