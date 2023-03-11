# fully-automated-pipeline-infra-scripts
USE THE SCRIPTS BELOW TO PROVISION INFRASTRUCTURE REQUIRED AS NEEDED:

1. Jenkins:

Create an Amazon Linux 2 VM instance and call it "Jenkins"
Instance type: t2.Medium
Security Group (Open): 8080, 9100 and 22 to 0.0.0.0/0
Key pair: Select or create a new keypair
Attach Jenkins server with IAM role having "AdministratorAccess"
Enter user data below
After launching this Jenkins server, attach a tag as Key=Application, value=jenkins

Userdata:

#!/bin/bash
# Hardware requirements: AWS Linux 2 with mimum t2.medium type instance & port 8080(jenkins), 9100 (node-exporter) should be allowed on the security groups
# Installing Jenkins
sudo yum update –y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum upgrade -y
sudo amazon-linux-extras install java-openjdk11 -y
sudo yum install jenkins -y
sudo echo "jenkins ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
sudo systemctl enable jenkins
sudo systemctl start jenkins

# Installing Git
sudo yum install git -y



-----------------------------------------------------------------------------------

2. Install Maven an Java 8 on Jenkins Server

#Steps To Install Apache Maven and Java 8 on your EC2 instance

#Connect to your Amazon EC2 instance with an SSH client.

#Install Apache Maven on your EC2 instance. First, enter the following to add a repository with a Maven package.


sudo wget https://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo -O /etc/yum.repos.d/epel-apache-maven.repo

#Enter the following to set the version number for the packages.

sudo sed -i s/\$releasever/6/g /etc/yum.repos.d/epel-apache-maven.repo

#Then you can use yum to install Maven.

sudo yum install -y apache-maven

#The Gremlin libraries require Java 8. Enter the following to install Java 8 on your EC2 instance.

sudo yum install java-1.8.0-devel

#Enter the following to set Java 8 as the default runtime on your EC2 instance.

sudo /usr/sbin/alternatives --config java

#When prompted, enter the number for Java or just hit enter.
#Enter the following to set Java 8 as the default compiler on your EC2 instance.

sudo /usr/sbin/alternatives --config javac

#When prompted, enter the number for Java 8(or whatever version that got installed).
#Verify your maven version with the command below
mvn -v

#Project Preparation

#Create the .m2 directory in the home directory of your current user 
mkdir ~/.m2

#Create the Settings file inside of the ~/.m2 directory cd ~/.m2/ mv demo/settings.xml ~/.m2/ , We can leave out this step till later.

----------------------------------------------------------------------------------------

3. SonarQube:

Create an Create an Ubuntu 20.04 VM instance and call it "SonarQube"
Instance type: t2.medium
Security Group (Open): 9000, 9100 and 22 to 0.0.0.0/0
Key pair: Select or create a new keypair
Enter user data below

#!/bin/bash
cp /etc/sysctl.conf /root/sysctl.conf_backup
cat <<EOT> /etc/sysctl.conf
vm.max_map_count=262144
fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
EOT
cp /etc/security/limits.conf /root/sec_limit.conf_backup
cat <<EOT> /etc/security/limits.conf
sonarqube   -   nofile   65536
sonarqube   -   nproc    409
EOT

sudo apt-get update -y
sudo apt-get install openjdk-11-jdk -y
sudo update-alternatives --config java

java -version

sudo apt update
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -

sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
sudo apt install postgresql postgresql-contrib -y
#sudo -u postgres psql -c "SELECT version();"
sudo systemctl enable postgresql.service
sudo systemctl start  postgresql.service
sudo echo "postgres:admin123" | chpasswd
runuser -l postgres -c "createuser sonar"
sudo -i -u postgres psql -c "ALTER USER sonar WITH ENCRYPTED PASSWORD 'admin123';"
sudo -i -u postgres psql -c "CREATE DATABASE sonarqube OWNER sonar;"
sudo -i -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;"
systemctl restart  postgresql
#systemctl status -l   postgresql
netstat -tulpena | grep postgres
sudo mkdir -p /sonarqube/
cd /sonarqube/
sudo curl -O https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.3.0.34182.zip
sudo apt-get install zip -y
sudo unzip -o sonarqube-8.3.0.34182.zip -d /opt/
sudo mv /opt/sonarqube-8.3.0.34182/ /opt/sonarqube
sudo groupadd sonar
sudo useradd -c "SonarQube - User" -d /opt/sonarqube/ -g sonar sonar
sudo chown sonar:sonar /opt/sonarqube/ -R
cp /opt/sonarqube/conf/sonar.properties /root/sonar.properties_backup
cat <<EOT> /opt/sonarqube/conf/sonar.properties
sonar.jdbc.username=sonar
sonar.jdbc.password=admin123
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
sonar.web.host=0.0.0.0
sonar.web.port=9000
sonar.web.javaAdditionalOpts=-server
sonar.search.javaOpts=-Xmx512m -Xms512m -XX:+HeapDumpOnOutOfMemoryError
sonar.log.level=INFO
sonar.path.logs=logs
EOT

cat <<EOT> /etc/systemd/system/sonarqube.service
[Unit]
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
WantedBy=multi-user.target
EOT

systemctl daemon-reload
systemctl enable sonarqube.service
#systemctl start sonarqube.service
#systemctl status -l sonarqube.service
apt-get install nginx -y
rm -rf /etc/nginx/sites-enabled/default
rm -rf /etc/nginx/sites-available/default
cat <<EOT> /etc/nginx/sites-available/sonarqube
server{
    listen      80;
    server_name sonarqube.groophy.in;

    access_log  /var/log/nginx/sonar.access.log;
    error_log   /var/log/nginx/sonar.error.log;

    proxy_buffers 16 64k;
    proxy_buffer_size 128k;

    location / {
        proxy_pass  http://127.0.0.1:9000;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;
              
        proxy_set_header    Host            \$host;
        proxy_set_header    X-Real-IP       \$remote_addr;
        proxy_set_header    X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto http;
    }
}
EOT
ln -s /etc/nginx/sites-available/sonarqube /etc/nginx/sites-enabled/sonarqube
systemctl enable nginx.service
#systemctl restart nginx.service
sudo ufw allow 80,9000,9001/tcp

echo "System reboot in 30 sec"
sleep 30
reboot
---------------------------------------------------------------------------------
4. Nexus Repository Manager

Create an Amazon Linux 2 VM instance and call it "Nexus"
Instance type: t2.medium
Security Group (Open): 8081, 9100 and 22 to 0.0.0.0/0
Key pair: Select or create a new keypair

Sample install link: https://devopscube.com/how-to-install-latest-sonatype-nexus-3-on-linux/

User data:

#!/bin/bash
yum install java-1.8.0-openjdk.x86_64 wget -y   
mkdir -p /opt/nexus/   
mkdir -p /tmp/nexus/                           
cd /tmp/nexus/
NEXUSURL="https://download.sonatype.com/nexus/3/latest-unix.tar.gz"
wget $NEXUSURL -O nexus.tar.gz
EXTOUT=`tar xzvf nexus.tar.gz`
NEXUSDIR=`echo $EXTOUT | cut -d '/' -f1`
rm -rf /tmp/nexus/nexus.tar.gz
rsync -avzh /tmp/nexus/ /opt/nexus/
useradd nexus
chown -R nexus.nexus /opt/nexus 
cat <<EOT>> /etc/systemd/system/nexus.service
[Unit]                                                                          
Description=nexus service                                                       
After=network.target                                                            
                                                                  
[Service]                                                                       
Type=forking                                                                    
LimitNOFILE=65536                                                               
ExecStart=/opt/nexus/$NEXUSDIR/bin/nexus start                                  
ExecStop=/opt/nexus/$NEXUSDIR/bin/nexus stop                                    
User=nexus                                                                      
Restart=on-abort                                                                
                                                                  
[Install]                                                                       
WantedBy=multi-user.target                                                      

EOT

echo 'run_as_user="nexus"' > /opt/nexus/$NEXUSDIR/bin/nexus.rc
systemctl daemon-reload
systemctl start nexus
systemctl enable nexus


OR INSTALL STEP BY STEP AS BELLOW:

Instructions:

a. Remove older version of java and install latest version:

yum remove java* -y
sudo yum install java-1.8.0-openjdk.x86_64 -y

b. Download and setup nexus stable version:
cd /opt
wget https://sonatype-download.global.ssl.fastly.net/nexus/3/nexus-3.0.2-02-unix.tar.gz
tar -zxvf nexus-3.0.2-02-unix.tar.gz
mv /opt/nexus-3.0.2-02 /opt/nexus
i
c. As a good security practice, it is not advised to run nexus service as root. so create new user called nexus and grant sudo access to manage nexus services:

sudo adduser nexus
visudo \\ nexus ALL=(ALL) NOPASSWD: ALL
sudo chown -R nexus:nexus /opt/nexus

d. Open /opt/nexus/bin/nexus.rc file, uncomment run_as_user parameter and set it as following:

vi /opt/nexus/bin/nexus.rc
run_as_user="nexus" (file should have only this line)

e. Add nexus as a service at boot time:

sudo ln -s /opt/nexus/bin/nexus /etc/init.d/nexus

f. Login as a nexus user and start service:

su - nexus
service nexus start

g. Login nexus server from browser on port 8081:

http://<Nexus_server>:8081

h. Use default credentials to login:

username : admin
password : admin123


---------------------------------------------------------------------------

5. Prometheus

Create Amazon Linux 2 VM instance and call it "Prometheus"
Instance type: t2.micro
Security Group (Open): 9090 and 22 to 0.0.0.0/0
Key pair: Select or create a new keypair
Attach Jenkins server with IAM role having "AmazonEC2ReadOnlyAccess"

Sample install link: https://www.fosstechnix.com/how-to-install-prometheus-on-amazon-linux-2/

Ssh and run the following commands:

# Copy and paste all the following commands end with sudo nano command

sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir /etc/prometheus && sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus && cd /tmp/
wget https://github.com/prometheus/prometheus/releases/download/v2.31.1/prometheus-2.31.1.linux-amd64.tar.gz
tar -xvf prometheus-2.31.1.linux-amd64.tar.gz
cd prometheus-2.31.1.linux-amd64
sudo mv console* /etc/prometheus
sudo mv prometheus.yml /etc/prometheus
sudo chown -R prometheus:prometheus /etc/prometheus
sudo mv prometheus /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo nano /etc/systemd/system/prometheus.service


# Copy the following content and paste then save 

[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target

# Continue with the following command 

sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus

# Access your server with the link below

http://server-IP-or-Hostname:9090
-------------------------------------------------------------------------------------------------
6. Grafana

Create an Amazon Linux 2 instance and call it "Grafana"
Instance type: t2.micro
Security Group (Open): 3000 and 22 to 0.0.0.0/0
Key pair: Select or create a new keypair

SHH into your Grafana instance and run the following commands:

sudo yum update -y
sudo nano /etc/yum.repos.d/grafana.repo

# Paste the following content then save and exit: PS, don't copy the dotted lines

----
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
----

sudo yum install grafana -y
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server.service
sudo systemctl status grafana-server


#Access Grafana GUI suing the link below 

http://Your-IP-Address:3000

# Enter Default Credentials below

Username: admin	
Password: admin


--------------------------------------------------------------------------------------------


