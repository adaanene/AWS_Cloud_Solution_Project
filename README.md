**WARNING:** this project uses AWS resources that are not covered by AWS free tier so it is strongly recommended to set up an AWS budget and notifications for when spending reaches a predefined limit and to delete all resources at the end of the project

**NOTE:** The steps carried out for this project implementation are found in documentation.md
## Bastion AMI

```
sudo su -
sudo dnf update -y
sudo dnf install python3 -y

yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

yum install wget vim telnet htop git mysql net-tools chrony -y

systemctl start chronyd

systemctl enable chronyd

``````


## Nginx AMI

```
sudo su -
sudo dnf update -y
sudo dnf install python3 -y

yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

yum install wget vim telnet htop git mysql net-tools chrony -y

systemctl start chronyd

systemctl enable chronyd
```

## Webservers AMI

```
sudo su -
sudo dnf update -y
sudo dnf install python3 -y

yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

yum install wget vim telnet htop git mysql net-tools chrony php -y

systemctl start chronyd

systemctl enable chronyd
```


## Configure selinux policies - webservers and nginx server

```
setsebool -P httpd_can_network_connect=1
setsebool -P httpd_can_network_connect_db=1
setsebool -P httpd_execmem=1
setsebool -P httpd_use_nfs 1
```


## Install aws efs-utils for mounts - nginx and webservers

```
git clone https://github.com/aws/efs-utils
cd efs-utils
yum install -y make
yum install rpm-build -y
make rpm
yum install -y ./build/amazon-efs-utils*rpm
```

## Set up self-signed certificate for nginx server

``````
sudo mkdir /etc/ssl/private

sudo chmod 700 /etc/ssl/private

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt

sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
``````


## Set up self-signed certificate for the webservers 

```
yum install -y mod_ssl

openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/pki/tls/private/apache-selfsigned.key -out /etc/pki/tls/certs/apache-selfsigned.crt

vi /etc/httpd/conf.d/ssl.conf 
```


## Bastion userdata

``````
#!/bin/bash
yum update -y
yum install ansible git -y
``````

## Nginx userdata

``````
#!/bin/bash
sudo yum install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
git clone https://github.com/adaanene/ACS-project-config.git
mv /ACS-project-config/reverse.conf /etc/nginx/
mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf-distro
cd /etc/nginx/
touch nginx.conf
sed -n 'w nginx.conf' reverse.conf
sudo systemctl restart nginx
rm -rf reverse.conf
rm -rf /ACS-project-config

``````


## Wordpress userdata

**NOTE:** use access point for wordpress within NFS filesystem, the RDS endpoint of your database, the name, username and password for your wordpress database within MySQL.

Change:
`enpoint: project-15-database.cmpdna6xtofl.eu-west-2.rds.amazonaws.com
access point: accesspoint=fsap-0c850690a4e5e8bd2 fs-00e875e0aa3feb6e3
username: admin
password: admin12345
database name: wordpressdb`

``````
#!/bin/bash
sudo yum update -y
mkdir /var/www/
sudo mount -t efs -o tls,accesspoint=fsap-0c850690a4e5e8bd2 fs-00e875e0aa3feb6e3:/ /var/www/
sudo yum install -y httpd 
sudo systemctl start httpd
sudo systemctl enable httpd
sudo yum module reset php -y
sudo yum module enable php:remi-7.4 -y
sudo yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
wget http://wordpress.org/latest.tar.gz
tar xzvf latest.tar.gz
rm -rf latest.tar.gz
cp wordpress/wp-config-sample.php wordpress/wp-config.php
mkdir /var/www/html/
cp -R /wordpress/* /var/www/html/
cd /var/www/html/
touch healthstatus
sed -i "s/localhost/project-15-database.cmpdna6xtofl.eu-west-2.rds.amazonaws.com/g" wp-config.php 
sed -i "s/username_here/admin/g" wp-config.php 
sed -i "s/password_here/admin12345/g" wp-config.php 
sed -i "s/database_name_here/wordpressdb/g" wp-config.php 
sudo chcon -t httpd_sys_rw_content_t /var/www/html/ -R
sudo systemctl restart httpd

``````



## Tooling userdata

**NOTE:** use access point of your tooling filesystem, the RDS endpoint of your database, the name, username and password for your tooling database within MySQL 

Change:
`enpoint: project-15-database.cmpdna6xtofl.eu-west-2.rds.amazonaws.com
access point: accesspoint=fsap-059fb9b0214742c05 fs-00e875e0aa3feb6e3
username: admin
password: admin12345
database name: toolingdb`

``````
#!/bin/bash
mkdir /var/www/
sudo mount -t efs -o tls,accesspoint=fsap-059fb9b0214742c05 fs-00e875e0aa3feb6e3:/ /var/www/
sudo yum install -y httpd 
sudo systemctl start httpd
sudo systemctl enable httpd
sudo yum module reset php -y
sudo yum module enable php:remi-7.4 -y
sudo yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
git clone https://github.com/adaanene/tooling-1.git
mkdir /var/www/html
cp -R /tooling-1/html/*  /var/www/html/
cd /tooling-1
mysql -h project-15-database.cmpdna6xtofl.eu-west-2.rds.amazonaws.com -u admin -p toolingdb < tooling-db.sql
cd /var/www/html/
touch healthstatus
sed -i "s/$db = mysqli_connect('mysql.tooling.svc.cluster.local', 'admin', 'admin', 'tooling');/$db = mysqli_connect('project-15-database.cmpdna6xtofl.eu-west-2.rds.amazonaws.com', 'admin', 'admin12345', 'toolingdb');/g" functions.php
sudo chcon -t httpd_sys_rw_content_t /var/www/html/ -R
sudo systemctl restart httpd

``````

### REFERENCES

[Self-signed certificate for Nginx and Apache](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-nginx-on-centos-7)