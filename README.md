# Tp B2


## Setup des VM

```
sudo hostname gitlab
sudo nano /etc/hostname
nmcli con up enp0s8
sudo nano /etc/sysconfig/network-scripts/ifcfg-enp0s8
```
![](https://i.imgur.com/31lMgTL.png)

![](https://i.imgur.com/CoHUx7g.png)

```
sudo dnf update -y
```

```
[tomfox@bastion-ovh1fr ~]$ sudo systemctl start firewalld
[tomfox@bastion-ovh1fr ~]$ sudo firewall-cmd --permanent --list-all
public
  target: default
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 
  protocols: 
  forward: no
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules:
```

```
❯ ssh tomfox@192.168.60.10
tomfox@192.168.60.10's password: 
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Tue Sep 13 05:55:30 2022 from 192.168.60.1
[tomfox@bastion-ovh1fr ~]$
```

```
❯ ssh-copy-id -i ~/.ssh/id_rsa.pub tomfox@192.168.60.10
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/tomfox/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
tomfox@192.168.60.10's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'tomfox@192.168.60.10'"
and check to make sure that only the key(s) you wanted were added.
```

```
❯ ssh tomfox@192.168.60.10
Activate the web console with: systemctl enable --now cockpit.socket

Last failed login: Tue Sep 13 06:05:13 EDT 2022 from 192.168.60.1 on ssh:notty
There was 1 failed login attempt since the last successful login.
Last login: Tue Sep 13 06:03:39 2022 from 192.168.60.1
[tomfox@bastion-ovh1fr ~]$
```


hostname n'a pas encore changer car j'ai pas reboot le serv

```
sudo nano /etc/ssh/sshd_config
#PubkeyAuthentication yes --> PubkeyAuthentication yes
PasswordAuthentication yes --> PasswordAuthentication no
```

```
[ZenBook ~]# ssh tomfox@192.168.60.10
The authenticity of host '192.168.60.10 (192.168.60.10)' can't be established.
ED25519 key fingerprint is SHA256:w5GiCf+NJsqCljjLiJuTRlHtGtBXMTZSBNqvNmjegnw.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.60.10' (ED25519) to the list of known hosts.
tomfox@192.168.60.10: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

```
sudo dnf install epel-release -y
sudo dnf install fail2ban -y
sudo systemctl enable fail2ban.service
sudo systemctl enable fail2ban.service
```
On laisse la config de base, puis bon un fail2ban alors qu'on est en full echange de clé only je crois que c'est useless.


Ensuite j'ai dupliquer la vm pour crée mon server DNS et runner

```
❯ ssh tomfox@192.168.60.12
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Tue Sep 13 08:08:17 2022 from 192.168.60.1
[tomfox@runner ~]$ logout
Connection to 192.168.60.12 closed.
❯ ssh tomfox@192.168.60.11
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Tue Sep 13 08:07:07 2022 from 192.168.60.1
[tomfox@DNS ~]$ logout
Connection to 192.168.60.11 closed.
❯ ssh tomfox@192.168.60.10
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Tue Sep 13 08:09:52 2022 from 192.168.60.1
[tomfox@gitlab ~]$
```


## Mise en place du DNS

```
sudo dnf install bind bind-utils
sudo systemctl enable named
sudo systemctl start named
```

```
[tomfox@DNS ~]$ sudo cat /etc/named.conf

options {
        listen-on port 53 { 127.0.0.1; 192.168.60.11; };
        #listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { trusted; };

        /* 
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable 
           recursion. 
         - If your recursive DNS server has a public IP address, you MUST enable access 
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification 
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface 
        */
        recursion yes;

        dnssec-enable yes;
        dnssec-validation yes;

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        /* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
        include "/etc/crypto-policies/back-ends/bind.config";
};

acl "trusted" {
        192.168.60.10;  # Gitlab
        192.168.60.11;  # DNS
        192.168.60.12;  # Runner
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
include "/etc/named/named.conf.local";
```

J'ai ajouter 192.168.60.11 dans listen-on port.
j'ai crée acl "trusted" que j'ai ensuite mis dans allow-query
a la fin du file j'ai add include "/etc/named/named.conf.local";

```
sudo chmod 755 /etc/named
sudo mkdir /etc/named/zones
```

Je crée ces files

```
[tomfox@DNS ~]$ cat /etc/named/named.conf.local
zone "tp-ynov.lab" {
    type master;
    file "/etc/named/zones/db.tp-ynov.lab"; # zone file path
};

zone "60.168.192.in-addr.arpa" {
    type master;
    file "/etc/named/zones/db.60.168.192";  # 192.168.60.0/24 subnet
};
```

```
[tomfox@DNS ~]$ cat /etc/named/zones/db.tp-ynov.lab 
$TTL    604800
@       IN      SOA     dns.tp-ynov.lab. admin.tp-ynov.lab. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; name servers - NS records
    IN      NS      dns.tp-ynov.lab.


; name servers - A records
dns.tp-ynov.lab.          IN      A       192.168.60.11

; 192.168.60.0/24 - A records
gitlab.tp-ynov.lab.        IN      A      192.168.60.10
runner.tp-ynov.lab.        IN      A      192.168.60.12
```

```
[tomfox@DNS ~]$ cat /etc/named/zones/db.60.168.192 
$TTL    604800
@       IN      SOA     dns.tp-ynov.lab. admin.tp-ynov.lab. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; name servers - NS records
      IN      NS      dns.tp-ynov.lab.

; PTR Records
10   IN      PTR     gitlab.tp-ynov.lab.  ; 192.168.60.10
11   IN      PTR     dns.tp-ynov.lab.     ; 192.168.60.11
12   IN      PTR     runner.tp-ynov.lab.  ; 192.168.60.12
```

Suite a ça je check si tout fonctionne

```
[tomfox@DNS ~]$ sudo named-checkconf
[tomfox@DNS ~]$ sudo named-checkzone tp-ynov.lab /etc/named/zones/db.tp-ynov.lab
zone tp-ynov.lab/IN: loaded serial 3
OK
[tomfox@DNS ~]$ sudo named-checkzone 60.168.192.in-addr.arpa /etc/named/zones/db.60.168.192
zone 60.168.192.in-addr.arpa/IN: loaded serial 3
OK
```

```
sudo systemctl restart named
```

Mtn il faut mettre une règle firewall, on crée donc une zone pour tout les ip dans mon réseau priv


```
sudo firewall-cmd --zone=trusted --add-source=192.168.60.0/24 --permanent
sudo firewall-cmd --zone=trusted --add-service=dns --permanent
sudo firewall-cmd --reload
```

il nous manque plus qu'a setup notre dns sur les server

```
sudo nvim /etc/resolv.conf
search tp-ynov.lab 
nameserver 192.168.60.11
```

```
[tomfox@DNS ~]$ nslookup gitlab.tp-ynov.lab
Server:         192.168.60.11
Address:        192.168.60.11#53

Name:   gitlab.tp-ynov.lab
Address: 192.168.60.10

[tomfox@DNS ~]$ nslookup dns.tp-ynov.lab
Server:         192.168.60.11
Address:        192.168.60.11#53

Name:   dns.tp-ynov.lab
Address: 192.168.60.11

[tomfox@DNS ~]$ nslookup runner.tp-ynov.lab
Server:         192.168.60.11
Address:        192.168.60.11#53

Name:   runner.tp-ynov.lab
Address: 192.168.60.12
```

Ceci marche sur tout les serveur

```
[tomfox@runner ~]$ ping google.com
PING google.com (142.250.178.142) 56(84) bytes of data.
64 bytes from par21s22-in-f14.1e100.net (142.250.178.142): icmp_seq=1 ttl=63 time=19.3 ms
64 bytes from par21s22-in-f14.1e100.net (142.250.178.142): icmp_seq=2 ttl=63 time=21.1 ms
^C
--- google.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 19.302/20.212/21.123/0.921 ms
[tomfox@runner ~]$ curl ip.me
92.103.174.138
```

Les serveur ont une résolutions externes a google.com ou autre


## Gitlab

(j'ai plus output)

```
sudo dnf install postfix -y
sudo systemctl enable postfix
sudo systemctl start postfix
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
sudo dnf install gitlab-ce -y
sudo gitlab-ctl reconfigure
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

Après avoir changer le root password

![](https://i.imgur.com/XsVqUUg.png)

(Je n'utilise pas gitlab.tp-ynov.lab car je suis sur le wifi ynov et du coup trop chiant de changer les dns)

## Monitoring (Grafana/Prometheus/Node Exporter)

### Install grafana

```
sudo nvim /etc/yum.repos.d/grafana.repo
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
[tomfox@DNS ~]$ sudo dnf update
grafana                                                                                                                                                                                                     1.4 kB/s | 454  B     00:00    
grafana                                                                                                                                                                                                      15 kB/s | 1.7 kB     00:00    
Importing GPG key 0x24098CB6:
 Userid     : "Grafana <info@grafana.com>"
 Fingerprint: 4E40 DDF6 D76E 284A 4A67 80E4 8C8C 34C5 2409 8CB6
 From       : https://packages.grafana.com/gpg.key
Is this ok [y/N]: y
grafana                                                                                                                                                                                                     8.1 MB/s |  12 MB     00:01    
Last metadata expiration check: 0:00:03 ago on Wed 14 Sep 2022 04:47:04 AM EDT.
Dependencies resolved.
Nothing to do.
Complete!
[tomfox@DNS ~]$ sudo dnf repolist
repo id       repo name
appstream     Rocky Linux 8 - AppStream
baseos        Rocky Linux 8 - BaseOS
epel          Extra Packages for Enterprise Linux 8 - x86_64
epel-modular  Extra Packages for Enterprise Linux Modular 8 - x86_64
extras        Rocky Linux 8 - Extras
grafana
```

```
[tomfox@DNS ~]$ sudo dnf install grafana -y
[...]
Installed:
  grafana-9.1.5-1.x86_64                              libICE-1.0.9-15.el8.x86_64                                       libSM-1.2.3-1.el8.x86_64                                  libXcursor-1.1.15-3.el8.x86_64                          
  libXfixes-5.0.3-7.el8.x86_64                        libXi-1.7.10-1.el8.x86_64                                        libXinerama-1.1.4-1.el8.x86_64                            libXmu-1.1.3-1.el8.x86_64                               
  libXrandr-1.5.2-1.el8.x86_64                        libXt-1.1.5-12.el8.x86_64                                        libXxf86misc-1.0.4-1.el8.x86_64                           libXxf86vm-1.1.4-9.el8.x86_64                           
  libfontenc-1.1.3-8.el8.x86_64                       libmcpp-2.7.2-20.el8.x86_64                                      mcpp-2.7.2-20.el8.x86_64                                  urw-base35-bookman-fonts-20170801-10.el8.noarch         
  urw-base35-c059-fonts-20170801-10.el8.noarch        urw-base35-d050000l-fonts-20170801-10.el8.noarch                 urw-base35-fonts-20170801-10.el8.noarch                   urw-base35-fonts-common-20170801-10.el8.noarch          
  urw-base35-gothic-fonts-20170801-10.el8.noarch      urw-base35-nimbus-mono-ps-fonts-20170801-10.el8.noarch           urw-base35-nimbus-roman-fonts-20170801-10.el8.noarch      urw-base35-nimbus-sans-fonts-20170801-10.el8.noarch     
  urw-base35-p052-fonts-20170801-10.el8.noarch        urw-base35-standard-symbols-ps-fonts-20170801-10.el8.noarch      urw-base35-z003-fonts-20170801-10.el8.noarch              xorg-x11-font-utils-1:7.5-41.el8.x86_64                 
  xorg-x11-server-utils-7.7-27.el8.x86_64            

Complete!
```

```
[tomfox@DNS ~]$ sudo systemctl enable --now grafana-server.service 
[tomfox@DNS ~]$ sudo firewall-cmd --add-port=3000/tcp --permanent
[tomfox@DNS ~]$ sudo firewall-cmd --reload
```

![](https://i.imgur.com/iWL7iu5.png)


![](https://i.imgur.com/SRQBap1.png)

Ok maintenant nous avons besoin d'une db time based pour notre dashboard grafana nous allons utilisé Prometheus

### Install Prometheus

```
[tomfox@DNS ~]$ sudo groupadd --system prometheus
[tomfox@DNS ~]$ sudo useradd -s /sbin/nologin --system -g prometheus prometheus
[tomfox@DNS ~]$ sudo mkdir /var/lib/prometheus
[tomfox@DNS ~]$ sudo mkdir /etc/prometheus
[tomfox@DNS ~]$ wget https://github.com/prometheus/prometheus/releases/download/v2.37.1/prometheus-2.37.1.linux-amd64.tar.gz
[tomfox@DNS ~]$ tar xvf prometheus-2.37.1.linux-amd64.tar.gz
prometheus-2.37.1.linux-amd64/
prometheus-2.37.1.linux-amd64/consoles/
prometheus-2.37.1.linux-amd64/consoles/index.html.example
prometheus-2.37.1.linux-amd64/consoles/node-cpu.html
prometheus-2.37.1.linux-amd64/consoles/node-disk.html
prometheus-2.37.1.linux-amd64/consoles/node-overview.html
prometheus-2.37.1.linux-amd64/consoles/node.html
prometheus-2.37.1.linux-amd64/consoles/prometheus-overview.html
prometheus-2.37.1.linux-amd64/consoles/prometheus.html
prometheus-2.37.1.linux-amd64/console_libraries/
prometheus-2.37.1.linux-amd64/console_libraries/menu.lib
prometheus-2.37.1.linux-amd64/console_libraries/prom.lib
prometheus-2.37.1.linux-amd64/prometheus.yml
prometheus-2.37.1.linux-amd64/LICENSE
prometheus-2.37.1.linux-amd64/NOTICE
prometheus-2.37.1.linux-amd64/prometheus
prometheus-2.37.1.linux-amd64/promtool
[tomfox@DNS ~]$ cd prometheus-2.37.1.linux-amd64/
[tomfox@DNS prometheus-2.37.1.linux-amd64]$ sudo cp prometheus promtool /usr/local/bin/
[tomfox@DNS prometheus-2.37.1.linux-amd64]$ sudo cp -r consoles/ console_libraries/ /etc/prometheus/
[tomfox@DNS prometheus-2.37.1.linux-amd64]$ sudo cp prometheus.yml /etc/prometheus/
[tomfox@DNS prometheus-2.37.1.linux-amd64]$ sudo chown -R prometheus:prometheus /etc/prometheus
[tomfox@DNS prometheus-2.37.1.linux-amd64]$ sudo chown -R prometheus:prometheus /var/lib/prometheus
[tomfox@DNS prometheus-2.37.1.linux-amd64]$ sudo chown prometheus:prometheus /usr/local/bin/promtool 
[tomfox@DNS prometheus-2.37.1.linux-amd64]$ sudo chown prometheus:prometheus /usr/local/bin/prometheus 
[tomfox@DNS ~]$ sudo cat /etc/systemd/system/prometheus.service
[sudo] password for tomfox: 
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
[tomfox@DNS ~]$ sudo systemctl enable --now prometheus
[tomfox@DNS ~]$ sudo firewall-cmd --add-port=9090/tcp --permanent
[tomfox@DNS ~]$ sudo firewall-cmd --reload
```
(J'ouvre le port juste pour voir si tout fonctionne bien je le fermerais a la fin car grafana et prometheus tourne sur la même machine)

![](https://i.imgur.com/HFtFQRk.png)

La on est good mtn go mettre prometheus dans grafana
(Dsl léo je vais mettre qlq scren)

![](https://i.imgur.com/QxlQoSn.png)

![](https://i.imgur.com/JRGCg5z.png)

j'importe un dashboard default pour voir si tout marche bien

![](https://i.imgur.com/tphKyhQ.png)

Ok cool mtn je vais utiliser node_exporter pour avoir plus d'info et un meilleurs dashboard

### Install Node Exporter

```
[tomfox@DNS ~]$ sudo groupadd --system node_exporter
[sudo] password for tomfox: 
[tomfox@DNS ~]$ sudo useradd -s /sbin/nologin --system -g node_exporter node_exporter
[tomfox@DNS ~]$ wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
[tomfox@DNS ~]$ tar xvfz node_exporter-1.3.1.linux-amd64.tar.gz
[tomfox@DNS ~]$ sudo cp node_exporter-1.3.1.linux-amd64/node_exporter /usr/local/bin/
[tomfox@DNS ~]$ sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
[tomfox@DNS ~]$ cat /etc/systemd/system/node_exporter.service
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --collector.cpu --collector.meminfo --collector.loadavg --collector.filesystem

[Install]
WantedBy=multi-user.target
[tomfox@DNS ~]$ sudo systemctl enable --now node_exporter.service 
```

```
[tomfox@DNS ~]$ cat /etc/prometheus/prometheus.yml 
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: 'dns'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']

```
On ajoute notre node dans la conf de prometheus

![](https://i.imgur.com/rb6qAlz.png)

On vois mtn des metrics pour le port 9100

(http://192.168.60.11:9100/metrics)

![](https://i.imgur.com/EfxQocW.png)

Prometheus va scrape tous ces metrics

sur grafana je vais utiliser ce dashboard https://grafana.com/grafana/dashboards/12486-node-exporter-full/

![](https://i.imgur.com/1Mp4g2n.png)

Voila on peut monitorer notre server dns, il nous manque plus qu'a installer node_exporter sur nos deux autre server, ainsi notre prometheus qui run sur le server dns ira scrape les data des node explorer des deux autre serveur sur le port 9100 (on va setup a la fin une règle au firewall comme quoi uniquement le server dns pourra accéder au port 9100 des autre server)

### Node sur tout les serv


Sur les autre server répeter l'étape install Node Exporter puis ceci:
```
[tomfox@runner ~]$ sudo firewall-cmd --new-zone=prometheus --permanent
[tomfox@runner ~]$ sudo firewall-cmd --reload
[tomfox@runner ~]$ sudo firewall-cmd --zone=prometheus --add-source=192.168.60.11 --permanent
[tomfox@runner ~]$ sudo firewall-cmd --zone=prometheus --add-port=9100/tcp --permanent
[tomfox@runner ~]$ sudo firewall-cmd --reload
```

(A noté que je dit les autre pour l'install node exporter sauf que gitlab a déjà node exporter d'intégrer donc il faut juste faire les commande pour le firewall)

suite à ça il faut changer la config de prometheus

```
[tomfox@DNS ~]$ cat /etc/prometheus/prometheus.yml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: 'infra'
    scrape_interval: 5s
    static_configs:
      - targets: ['dns.tp-ynov.lab:9100','gitlab.tp-ynov.lab:9100','runner.tp-ynov.lab:9100']
```

A la fin du file on vois qu'on a add nos deux server dans targets

qlq petit screen ;)

![](https://i.imgur.com/4MdObAj.png)

![](https://i.imgur.com/ajiCHpR.png)

![](https://i.imgur.com/6OoZQxS.png)

![](https://i.imgur.com/jfeAIOZ.png)

![](https://i.imgur.com/D3frO8L.png)

Voila tout est fini pour le monitoring
