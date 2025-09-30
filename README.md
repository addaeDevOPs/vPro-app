# VProfile Project Manual Provisioning Setup

This guide provides step-by-step instructions to manually provision and deploy the **VProfile** application stack using **Vagrant**, **VirtualBox**, and multiple Linux-based virtual machines.

**[vPro-app Architectural Diagram](https://ibb.co/BVt4J3HD)**

## 🧰 Prerequisites

Before proceeding, ensure you have the following tools installed on your system:

1. **[Oracle VM VirtualBox](https://www.virtualbox.org/)**

2. **[Vagrant](https://developer.hashicorp.com/vagrant)**

3. **Vagrant Plugins**

   Install the required plugin:

   ```bash
   vagrant plugin install vagrant-hostmanager
   ```

4. **Git Bash** (or any UNIX-compatible terminal)

---

## 📦 VM Setup

### Steps:

1. **Clone the repository**

   ```bash
   git clone https://github.com/addaeDevOPs/vPro-app.git
   cd vPro-app
   ```

2. **Switch to the local branch**

   ```bash
   git checkout local
   ```

3. **Navigate to the provisioning directory**

   ```bash
   cd vagrant/Manual_provisioning_WinMacIntel
   ```

4. **Bring up the VMs**

   ```bash
   vagrant up
   ```

   > ⚠️ This process may take time depending on your system and internet speed.

   > 💡 If the setup is interrupted, simply run `vagrant up` again.

### Info:

* VM hostnames and your `/etc/hosts` file will be automatically updated by `vagrant-hostmanager`.

---

## 🏗️ Services & Architecture

 Service            Description                
 **MySQL**          SQL Database               
 **Memcache**       DB Caching Service         
 **RabbitMQ**       Message Queue/Broker       
 **Tomcat**         Application Server         
 **Nginx**          Web Server (Reverse Proxy) 
 **Elasticsearch**  Indexing/Search Engine     

### 🔁 Provisioning Order:

1. **MySQL**
2. **Memcache**
3. **RabbitMQ**
4. **Tomcat**
5. **Nginx**

---

## 🔧 Service-by-Service Setup

### 1. 🛢️ MySQL Setup

SSH into the database VM:

```bash
vagrant ssh db01
```

**Basic Setup:**

```bash
sudo dnf update -y
sudo dnf install epel-release -y
sudo dnf install git mariadb-server -y
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

```
sudo mysql_secure_installation
```
**[mysql secure installation image](https://ibb.co/HTb7ZZYV)**

# Database
Here,we used Mysql DB 
sql dump file:
- /src/main/resources/db_backup.sql
- db_backup.sql file is a mysql dump file.we have to import this dump to mysql db server
- > mysql -u <user_name> -p accounts < db_backup.sql

> Use `admin123` as the root password when prompted.

**Configure Database:**

```sql
mysql -u root -padmin123
CREATE DATABASE accounts;
GRANT ALL PRIVILEGES ON accounts.* TO 'admin'@'localhost' IDENTIFIED BY 'admin123';
GRANT ALL PRIVILEGES ON accounts.* TO 'admin'@'%' IDENTIFIED BY 'admin123';
FLUSH PRIVILEGES;
EXIT;
```

**Initialize Database:**

```bash
cd /tmp/
git clone -b local https://github.com/addaeDevOPs/vPro-app.git
cd vPro-app
mysql -u root -padmin123 accounts < src/main/resources/db_backup.sql
```

**Firewall Setup:**

```bash
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
```

---

### 2. 🧠 Memcache Setup

SSH into the Memcache VM:

```bash
vagrant ssh mc01
```

**Install & Configure:**

```bash
sudo dnf update -y
sudo dnf install epel-release -y
sudo dnf install memcached -y
sudo systemctl start memcached
sudo systemctl enable memcached
sudo sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
sudo systemctl restart memcached
```

**Firewall:**

```bash
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo firewall-cmd --add-port=11211/tcp
sudo firewall-cmd --runtime-to-permanent
```

---

### 3. 📩 RabbitMQ Setup

SSH into the RabbitMQ VM:

```bash
vagrant ssh rmq01
```

**Install & Configure:**

```bash
sudo dnf update -y
sudo dnf install epel-release wget -y
sudo dnf -y install centos-release-rabbitmq-38
sudo dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
sudo systemctl enable --now rabbitmq-server
```

**Configure User:**

```bash
sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
sudo rabbitmqctl add_user test test
sudo rabbitmqctl set_user_tags test administrator
sudo rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
sudo systemctl restart rabbitmq-server
```

**Firewall:**

```bash
sudo firewall-cmd --add-port=5672/tcp
sudo firewall-cmd --runtime-to-permanent
```

---

### 4. 🧩 Tomcat Setup (Application Server)

SSH into the app VM:

```bash
vagrant ssh app01
```

**Install Java & Tomcat:**

```bash
sudo dnf update -y
sudo dnf install epel-release java-17-openjdk java-17-openjdk-devel git wget -y
cd /tmp/
wget https://archive.apache.org/dist/tomcat/tomcat-10/v10.1.26/bin/apache-tomcat-10.1.26.tar.gz
tar xzvf apache-tomcat-10.1.26.tar.gz
sudo mkdir -p /usr/local/tomcat
sudo useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat
sudo cp -r apache-tomcat-10.1.26/* /usr/local/tomcat/
sudo chown -R tomcat.tomcat /usr/local/tomcat
```

**Create SystemD Service:**

```bash
sudo vi /etc/systemd/system/tomcat.service
```

Paste the following content:

```ini
[Unit]
Description=Tomcat
After=network.target

[Service]
User=tomcat
Group=tomcat
WorkingDirectory=/usr/local/tomcat
Environment=JAVA_HOME=/usr/lib/jvm/jre
Environment=CATALINA_PID=/var/tomcat/%i/run/tomcat.pid
Environment=CATALINA_HOME=/usr/local/tomcat
Environment=CATALINE_BASE=/usr/local/tomcat
ExecStart=/usr/local/tomcat/bin/catalina.sh run
ExecStop=/usr/local/tomcat/bin/shutdown.sh
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

**Start Tomcat:**

```bash
sudo systemctl daemon-reload
sudo systemctl start tomcat
sudo systemctl enable tomcat
```

**Firewall:**

```bash
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

---

### 5. 🚀 Code Build & Deployment

**Install Maven:**

```bash
cd /tmp/
wget https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.zip
unzip apache-maven-3.9.9-bin.zip
sudo cp -r apache-maven-3.9.9 /usr/local/maven3.9
export MAVEN_OPTS="-Xmx512m"
```

**Build & Deploy:**

```bash
git clone -b local https://github.com/addaeDevOPs/vPro-app.git
cd vPro-app
vim src/main/resources/application.properties  # Update backend config

/usr/local/maven3.9/bin/mvn install

sudo systemctl stop tomcat
sudo rm -rf /usr/local/tomcat/webapps/ROOT*
sudo cp target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
sudo chown -R tomcat.tomcat /usr/local/tomcat/webapps
sudo systemctl restart tomcat
```

---

### 6. 🌐 Nginx Setup

SSH into the web VM:

```bash
vagrant ssh web01
sudo -i
```

**Install & Configure Nginx:**

```bash
apt update && apt upgrade -y
apt install nginx -y

vi /etc/nginx/sites-available/vproapp
```

Paste the following configuration:

```nginx
upstream vproapp {
    server app01:8080;
}

server {
    listen 80;
    location / {
        proxy_pass http://vproapp;
    }
}
```

**Enable New Site:**

```bash
rm -rf /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
systemctl restart nginx
```

---

## ✅ Final Notes

* Access the app via the link below:

  ```bash
  http://192.168.56.11:80
  ```
