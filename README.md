# Elastic-stack-SIEM-complete-Guide
# Introdction and support mention

this is a detialed Elastic stack SIEM complete Guide, wich is a project that i have work on it 
*{ to work on it}*

# How can you benifit from this reposotry:

for take the best from this reposetory you can flow the order of modules and section wich will start by explaining the fandemantels of SIEM on genarel then, you will find a explanation of Eastic stack his componates and how it’s working, then we will explain step by how you can deploye a ELK SIEM (for practice we will call elastick stack =only⇒ “ELK” the old name of the product’s wich means the basic components of it’s wich they are Elasticsearch, Kibana, Logstash),
then will working on deffirente use cases with and deffirentes system sources.

if you want to get more deep undrstanding of the product you can merge between the officiel documentation and this reposetory.
So, Give your Cap of coffe and let’s start on.

---

# Lab ressources :

Essiantials cmoponents :

- we have work the the version 8.19 of Elastik search .
- the first version of the Lab we will install al the compneants using  VMs on vmware workstation 17.
- i have using Ubuntu 22.04 for install the compnanants on it.

system sources that i have use :

- Windows server 2019
- ubuntu  server 22.04
- Fortigate FW
- cisco SWs *{Upcomming}*
- Tripwire *{Upcomming}*
- Wallix *{Upcomming}*
- other linux distrupution *{Upcomming}*
- palo alto FW *{Upcomming}*
- WAF *{Upcomming}*
- cloud Azur *{Upcomming}*
- PGP (symantic encryption) *{Upcomming}*

---

# Architecture proposée et explications

*this will be metionned under te section of instalation Steps
and there you could also mention that we will work on an other version where the architucture will focus on how to deploye a cluster* 

Tu veux :

- **3 machines** pour un **cluster Elasticsearch** (noeuds)
- **1 machine** pour Kibana
- **1 machine** pour Logstash + Fleet Server

Ensuite, pour les sources :

- 1 machines **Windows** qui envoient via **Elastic Agent → Fleet Server → Elasticsearch**
- **Tripwire** qui envoie ses logs via **Logstash pipeline** vers Elasticsearch.

utilises Elastic Agent + ingestion par Logstash.

## phase 2 : configue étape par étape

## 🛠️ Étapes pour installer ton cluster “tolérance + performance raisonnable” (5 VMs)

### ✅ Préparation avant l’installation

1. **Configurer les VMs dans VMware**
    - [ ]  Crée 5 VMs Ubuntu Server (version recommandée : 20.04 ou 22.04).
    - [ ]  Assure-toi que chaque VM peut accéder aux autres via le réseau NAT `192.168.12.0`.
    - [ ]  Attribue des adresses IP statiques et hostname par exemple :
    
    - es-node-1 : `192.168.12.20`
    - es-node-2 : `192.168.12.21`
    - es-node-3 : `192.168.12.22`
    - kibana : `192.168.12.23`
    - Logstash-Fleet : `192.168.12.24`
    
    ```bash
    sudo hostnamectl set-hostname <hostname>
    nano /etc/hosts
    # en l align o il y a 127.0.1.1 met le vouveaux hostname
    sudo reboot
    ```
    
    exampl 
    
    ![image.png](attachment:b57247a0-7b4c-44f3-858e-28d7469ae6c0:image.png)
    
    installation et activation du  service ssh et 
    
    ```bash
    sudo apt update
    sudo apt install openssh-server
    sudo systemctl enable ssh
    ```
    
    etendé la duré du session
    
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```
    
    1. définition d’une @ IP statique en 
        
        vous allez entrer au dossier netplan au vous allez trouvez le configuration réseaux de la machine, o vous devez désactivez DHCP, et définir une @ ip statique:
        
        ```jsx
        sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
        network: {config: disabled}
        sudo rm /etc/netplan/50-cloud-init.yaml
        nano /etc/netplan/01-cloud-init.yaml
        ```
        
        ```yaml
        network:
          version: 2
          renderer: networkd # Or 'systemd-networkd'
          ethernets:
            ens33: # <-- Replace with your interface name
              dhcp4: no
              addresses:
                - 192.168.12.10/24 # <-- Your desired Static IP/CIDR
              routes:
                - to: default
                  via: 192.168.12.2 # <-- Your Gateway IP
              nameservers:
                addresses: [8.8.8.8, 1.1.1.1] # <-- DNS Servers (e.g., Google/Cloudflare)
        ```
        
        ```jsx
        cat /etc/netplan/50-cloud-init.yaml
        sudo netplan appl
        ```
        

prendre une snapshot

1. **Configurer `vm.max_map_count` sur chaque nœud Elasticsearch**
2. **Installer Java (si nécessaire)**
    
    Selon la version d’Elasticsearch, Java peut être requis ou non. Si besoin :
    
    ```bash
    sudo apt update
    sudo apt install openjdk-17-jdk -y
    java -version
    ```
    
    # 👇👇👇
    
    🗒️ cloner les vm et adapter chaqu’un à la configue néssaicaire hostnames et ip
    
    ```bash
    sudo hostnamectl set-hostname <hostname>
    nano /etc/hosts
    # en l align o il y a 127.0.1.1 met le vouveaux hostname
    sudo reboot
    ```
    
    - Elasticsearch exige une valeur élevée pour `vm.max_map_count`. Exécute sur **chaque** VM ES :
        
        ```bash
        sudo sysctl -w vm.max_map_count=262144
        # Pour le rendre permanent :
        echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
        sudo sysctl -p
        
        ```
        
        - reason 💡
            
            
            تبارك الله عليك! نعم، الأوامر التي شرحتها:
            
            Bash
            
            # 
            
            `sudo sysctl -w vm.max_map_count=262144
            # ... لجعلها دائمة`
            
            **ضرورية جداً** لإنجاح وتأمين أداء **Cluster ديال Elasticsearch** (وهو المكون الرئيسي في ELK Stack).
            
            ---
            
            ## 🛑 لماذا هذا الإعداد ضروري للـ Cluster؟
            
            السبب يرجع إلى كيفية عمل محرك البحث **Lucene** (الذي يعتمد عليه Elasticsearch):
            
            1. **استخدام الذاكرة (Memory Mapping):** Elasticsearch يستخدم مكتبات Lucene التي تعتمد بشكل مكثف على تقنية تسمى **Memory-mapped files (mmap)**. هذه التقنية تسمح لـ Elasticsearch بالتعامل مع الملفات الكبيرة (مثل ملفات الـ Index) كما لو كانت جزءاً من الذاكرة الرئيسية، مما يزيد بشكل كبير من سرعة البحث والأداء.
            2. **الحد الأقصى للمناطق (`vm.max_map_count`):** المتغير `vm.max_map_count` يحدد العدد الأقصى لملفات الـ mmap التي يمكن لـ Elasticsearch فتحها في نفس الوقت.
            3. **فشل Cluster (إذا كان منخفضاً):**
                - إذا كانت القيمة الافتراضية (عادةً 65530) صغيرة جداً، فـ Elasticsearch سيفشل في فتح ملفات الـ Index اللازمة للعمل.
                - في بيئة **Cluster**، حيث يقوم كل **Node** بتخزين وإدارة عدد كبير من **Shards** و **Indices**، فإن عدد ملفات الـ mmap المطلوبة يتضاعف.
                - إذا لم يتم رفع القيمة، فإن الـ **Node** سيتوقف عن العمل فجأة عند تحميل Shards جديدة أو عند بدء التشغيل، وقد يعيق إنجاح عملية الـ Clustering بالكامل.
            
            **الخلاصة:** بدون رفع `vm.max_map_count` إلى **`262144`** على **جميع الـ Nodes** في Cluster ديالك، ستواجه مشاكل في استقرار الذاكرة والأداء، وقد تفشل عملية الـ Cluster في العمل بشكل صحيح.
            
            ---
            
            ## 📝 متطلبات أخرى ضرورية للـ Cluster
            
            بالإضافة إلى `vm.max_map_count`، يجب عليك التأكد من الإعدادات التالية على **كل Node** في الـ Cluster:
            
            1. **تخصيص الذاكرة (Heap Size):** يجب تعيين الذاكرة المخصصة لـ Java (Heap) في ملف **`jvm.options`** لتكون نصف الذاكرة المتوفرة على الـ VM (مثلاً، إذا كان لديك 16GB RAM، يجب أن تعين `Xms8g` و `Xmx8g`).
            2. **تعطيل الـ Swap:** يجب **تعطيل الـ Swap (ملف التبادل)** لمنع نظام التشغيل من نقل الـ Heap ديال Elasticsearch إلى القرص، مما يدمر الأداء.
                - (يمكن تعطيلها عبر الأمر `sudo swapoff -a` وإلغاء تعليق سطر الـ swap في `/etc/fstab`).
            3. **الـ File Descriptors:** رفع الحد الأقصى لعدد الـ File Descriptors المفتوحة إلى قيمة عالية (مثلاً 65536) في ملف `limits.conf`.

---

## 1️⃣ Installation des nœuds Elasticsearch (3 VMs)

Fais cette partie sur chacune des 3 VMs Elasticsearch (Node1, Node2, Node3).

1. **Ajouter le dépôt Elasticsearch**
    
    ```bash
    wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
    sudo apt-get install apt-transport-https
    echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
    sudo apt update
    
    ```
    
2. **Installer Elasticsearch**
    
    ```bash
    sudo apt install elasticsearch -y
    
    ```
    
3. **Activer et démarrer Elasticsearch**
    
    ```bash
    sudo /bin/systemctl daemon-reload
    sudo /bin/systemctl enable elasticsearch.service
    sudo systemctl start elasticsearch.service
    sudo systemctl status elasticsearch.service
    ```
    
    then change the password by : `/usr/share/elasticsearch/bin/elasticsearch-reset-password -i -u elastic`
    
    ⇒ elkcluster
    
    vivrifier si elasticsearch work by : `curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic https://localhost:9200`
    
4. **Configurer `elasticsearch.yml`**
    
    make copy of the yml file 
    
    `cp /etc/elasticsearch/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml.backup`
    
    Édite `/etc/elasticsearch/elasticsearch.yml` sur chaque nœud. Par exemple :
    
    - **Node1 (`192.168.12.20`)** :
        
        ```yaml
        #--------------------first step------------------------
        # modify only this lignes and left all by default intill the next step
        node.name: es-node-1
        network.host: 192.168.12.20 #pourquelle être accissible avec cette @ et pas seulement en localhost
        http.port: 9200
        ```
        
        allow port 9200 on FW on tcp & udp then reload.
        
        ```bash
        sudo ufw allow 9200/tcp
        sudo ufw allow 9200/udp
        sudo ufw status
        # 1. تعطيل (لحظي)
        sudo ufw disable
        
        # 2. تفعيل مجدداً (مع تحميل الإعدادات الجديدة)
        sudo ufw enable
        ```
        
        ```yaml
        # ------------------next step -----------------------
        cluster.name: mon-cluster
        transport.port: 9300
        
        discovery.seed_hosts: ["192.168.12.20", "192.168.12.21", "192.168.12.22"]
        cluster.initial_master_nodes: ["es-node-1", "es-node-2", "es-node-3"]
        
        node.roles: [ "master", "data" ]
        
        ```
        
    - **Node2 (`192.168.12.11`)** :
        
        ```yaml
        cluster.name: mon-cluster
        node.name: es-node-2
        network.host: 192.168.12.21
        http.port: 9200
        transport.port: 9300
        
        discovery.seed_hosts: ["192.168.12.20", "192.168.12.21", "192.168.12.22"]
        # ne pas répéter initial_master_nodes si tu as déjà démarré tout le cluster, mais pour le bootstrap initial c’est ok
        cluster.initial_master_nodes: ["es-node-1", "es-node-2", "es-node-3"]
        
        node.roles: [ "master", "data" ]
        
        ```
        
    - **Node3 (`192.168.12.12`)** :
        
        ```yaml
        cluster.name: mon-cluster
        node.name: es-node-3
        network.host: 192.168.12.22
        http.port: 9200
        transport.port: 9300
        
        discovery.seed_hosts: ["192.168.12.20", "192.168.12.21", "192.168.12.22"]
        cluster.initial_master_nodes: ["es-node-1", "es-node-2", "es-node-3"]
        
        node.roles: [ "master", "data" ]
        
        ```
        
    
    > Cette configuration met 3 nœuds master-eligible + data, ce qui est recommandé pour un petit cluster avec tolérance aux pannes. (Discuss the Elastic Stack)
    > 
5. **Vérifier le cluster**
    
    Depuis l’un des nœuds (ou un client) :
    
    ```bash
    curl http://192.168.12.20:9200/_cluster/health?pretty
    curl http://192.168.12.20:9200/_cat/nodes?v
    
    ```
    

---

## 2️⃣ Installation de **Kibana** (1 VM)

Sur la VM Kibana (`192.168.12.23`).

1. **Ajouter le dépôt Kibana**
    
    ```jsx
    wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
    sudo apt-get install apt-transport-https
    echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
    ```
    
    ```bash
    sudo apt update
    sudo apt install kibana -y
    sudo /bin/systemctl daemon-reload
    sudo /bin/systemctl enable kibana.service
    sudo systemctl start kibana.service
    sudo systemctl status kibana.service
    ```
    
2. **Configurer `kibana.yml`**
    
    Ouvre `/etc/kibana/kibana.yml` et mets :
    
    but befor taking an backup by  : 
    
    `cp /etc/kibana/kibana.yml /etc/kibana/kibana.yml.backup`
    
    the  `nano /etc/kibana/kibana.yml`
    
    ```yaml
    #------------------------first step-------------------
    server.port: 5601
    server.host: "192.168.12.23"
    #=====================system: elasticsearch====================
    elasticsearch.hosts: ["https://192.168.12.20: 9200"]
    elasticsearch.username: "kibana_system"
    elasticsearch.password: "elkcluster"
    #======================system: elasticsearch(opt)
    elasticsearch.ssl.certificateAuthorities: [ "/etc/kibana/certs/http_ca.crt"]
    ```
    
    where going to copy the certificatesfrom elasticsearch into kibana because we are going to use https for communication.
    
    crée le dosier premierement en kibana : 
    
    - `mkdir /etc/kibana/certs/`
    
    du la machine node 1 
    
    ```bash
    # 1. نسخ الملف إلى مجلد المستخدم العادي haji في الخادم 192.168.12.23
    scp /etc/elasticsearch/certs/http_ca.crt haji@192.168.12.23:/home/haji/
    
    # 2. تسجيل الدخول إلى الخادم الهدف ونقل الملف
    ssh haji@192.168.12.23
    # (داخل الخادم 192.168.12.23)
    sudo mv /home/haji/http_ca.crt /etc/kibana/certs/
    ```
    
    changer  le mode pass prébuilt in  du kibana parce que on vas autoriser d’authtifier par mots de pass au lieux du token enrollement par : 
    
    `/usr/share/elasticsearch/bin/elasticsearch-reset-password -i -u kibana_system`  ⇒ à : elkcluster
    
    en la machine kibana
    
    vous pouver verfifer avec  `cat`  si le fichier à été bien transmet et que le fichier yml de configuration et bien enregisterer
    
     and then autoriser le port 5601 en tcp et aussi udp par parfeu
    
    ```bash
    sudo ufw allow 5601/tcp
    sudo ufw status
    # 1. تعطيل (لحظي)
    sudo ufw disable
    
    # 2. تفعيل مجدداً (مع تحميل الإعدادات الجديدة)
    sudo ufw enable
    ```
    
    ```bash
    sudo systemctl restart kibana.service
    sudo systemctl status kibana.service
    ```
    
    after you get into the kibana interface you see wornning notification pup up you will fixe it by set a public address for kibana in the yml file 
    
    ```yaml
    server.publicBaseUrl: "http://192.168.12.23:5601"
    ```
    
    redémarer le systme aprer de sauvgarder les modification que vous fait en le fichier yml
    
3. **Accéder à Kibana**
    1. **Accéder à Kibana**
        
        Dans ton navigateur → `http://192.168.12.13:5601`
        
        puis s’authantifier par user name : `elastic` et password: `elkcluster`
        
    

```yaml
#------------------------next step-------------------
elasticsearch.hosts: ["http://192.168.12.10:9200","http://192.168.12.11:9200","http://192.168.12.12:9200"]

```

---

## 3️⃣ Installation de **Logstash + Fleet Server** (1 VM)

Sur ta VM dédiée (par exemple `192.168.12.14`).

1. **Installer Logstash**
    
    ```bash
    wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg
    sudo apt-get install apt-transport-https
    echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
    sudo apt update
    sudo apt install logstash -y
    ```
    
2. **Installer Elastic Agent / Fleet Server**
    - Télécharge l’Elastic Agent (version compatible avec ton Elasticsearch).
    - Suis la documentation de Fleet Server pour l’installer comme serveur : tu devras configurer `fleet.yml` pour pointer vers ton cluster ES, générer des certificats si tu veux TLS, etc. (voir docs Elastic : Fleet Server)
    - tester avec WD server et ses intégration et down one by one of the nodes to test all of them , surtout la node 1
    
    💡🚨 
    
    - pour bien monitorer elasticsearch i faut installler :
        - [ ]  il faut installer un agent en elasticsearchs nodes (policiers du Debians) pour bien monitorer les nodes d’une manier avancé (essayer de trouver  👉👍une méthode de ignorer certification ‼️‼️comme celle utiliser en  windows issues ou si non on vas êtres besoin crée une certification du fleet server  (commme celle crée déjat crée par  ES http_ca.crt) ❓❓)
        - [ ]  ajouter l’intégrartoin du elasticsearch à la policier de sans agent.
        
        chek l’état de l’intégration tous doit étres en vers
        
3. **Configurer un pipeline Logstash**
    
    prend du snapsohte ou vms ⇒ puis tester avec ce que j’ai du fortigate ⇒revenir ver les derniers snapshots ⇒ (et test  fortigat avec la méthode du fleet ) ⇒demader du fatima le fichier du config si ms collégues e ont pas le trouvez
    
    Crée un fichier de pipeline, par exemple `/etc/logstash/conf.d/agent-pipeline.conf` :
    
    ```
    # ==============================================================================
    # 1. INPUT STAGE: Receive Logs from FortiGate
    # ==============================================================================
    input {
      udp {
        host => "192.168.12.10"
        port => 5144            # FortiGate Syslog Port
    #    codec => plain
      }
    }
    
    # ==============================================================================
    # 2. FILTER STAGE: Parse and Process FortiGate Logs
    # ==============================================================================
    filter {
      grok {
        match => {"message" => "%{SYSLOG5424PRI}%{GREEDYDATA:message}" }
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
      add_field => { "logdate" => "%{date} %{time}"}
      }
    
      date {
        match => [ "logdate", "yyyy-MM-dd HH:mm:ss" ]
    #    timezone => "America/edmonton"
        target => "@timestamp"
      }
      mutate {
        remove_field => ["logdate", "date", "time"] # ✅ CORRECT SYNTAX
        convert => { "rcvdbyte" => "integer" }
        convert => { "sentbyte" => "integer" }
     }
    }
    # ==============================================================================
    # 3. OUTPUT STAGE: Send Logs to Elasticsearch
    # ==============================================================================
    output {
      stdout {
        codec => rubydebug
      }
      elasticsearch {
        hosts => ["https://192.168.12.10"]
        index => "firewall-%{+YYYY.MM.dd}"
        user => "elastic"
        password => "FP+eESzDMGixNOd95ra7"
        ssl => true
        cacert => "/etc/logstash/http_ca.crt"
      }
    
    }
    ```
    
4. **Démarrer Logstash**
    
    ```bash
    sudo systemctl enable logstash --now
    sudo systemctl status logstash
    
    ```
    
5. **Configurer Fleet Server dans l’Elastic Agent**
    - Lors de l’“enrollment” des agents, indique l’adresse de ton Fleet Server (`192.168.12.14`).
    - Vérifie que les agents s’enregistrent et envoient des logs.

---

## 4️⃣ Test & validation de ton cluster

1. **Valider le cluster ES**
    - Fais `curl _cluster/health` → doit être “green” ou “yellow” (avec 3 nœuds data + master).
    - Regarde `_cat/nodes?v` pour vérifier que les 3 nœuds sont bien dans le cluster.
2. **Valider Logstash ingestion**
    - Configure un agent Elastic sur ta machine Windows ou Linux test.
    - Envoie quelques logs → vérifier dans Kibana (index `agent-logs-*`).
    - Vérifie dans Logstash les logs de pipeline (erreurs, parsing…).
3. **Valider Kibana**
    - Crée un “Index Pattern” dans Kibana / “Data View” pour les index créés par Logstash.
    - Va dans Discover pour voir les documents.
    - Crée des visualisations simples (par ex. nombre de logs par type) pour vérifier que tout marche.
4. **Test de tolérance**
    - Arrête un des nœuds Elasticsearch (par exemple ES-Node3) :
        
        ```bash
        sudo systemctl stop elasticsearch
        
        ```
        
    - Vérifie avec `/_cluster/health` que le cluster reste en vie (quorum).
    - Redémarre le nœud et assure-toi qu’il réintègre le cluster sans problème.

---

## 5️⃣ Conseils & meilleures pratiques

- **Surveille les ressources** : Assure-toi que chaque VM a assez de RAM. Si les nœuds ES sont limités, ajuste les `Xms` / `Xmx` dans `/etc/elasticsearch/jvm.options`.
- **Backups** : Même dans un lab, fais des snapshots de ta VM ou des backups de tes données ES si tu veux garder des things importantes.
- **Logs & Monitoring** : Active la journalisation d’Elasticsearch (`/var/log/elasticsearch`) et surveille les erreurs.
- **Sécurité** : Si tu veux rendre ton cluster plus sécurisé, active TLS entre nœuds (`xpack.security.transport.ssl.*`) et l’authentification.
- **Nettoyage** : Si tu réinitialises souvent, n’oublie pas de vider le dossier `data` (`/var/lib/elasticsearch/`) avant de redémarrer un nœud pour éviter des conflits de cluster. Certains ont eu des erreurs quand ils ont réutilisé des données sans nettoyage. ([Reddit](https://www.reddit.com/r/elasticsearch/comments/yx7m4f?utm_source=chatgpt.com))

---

Si tu veux, je peux te donner **un script bash automatisé** que tu peux exécuter sur chaque VM Elasticsearch pour configurer ton cluster automatiquement (avec tes IP). Tu veux que je le prépare ?