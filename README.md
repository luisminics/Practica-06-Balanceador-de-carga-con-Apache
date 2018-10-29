# Practica-06-Balanceador-de-carga-con-Apache

## Instalar Virtualbox y vagrant
Para relizar esta práctica sera necesario instalar [Virtualbox](https://www.virtualbox.org/wiki/Downloads) y [vagrant](https://www.vagrantup.com/downloads.html) sobre el sistema operativo en el que vayamos a realizarla.

## Crear Vagrantfile para cuatro box de ubuntu server 18.04 dos server apache Apache, un server Mysql y un server apache balanceador de carga

Creamos un directorio donde contendremos las carpetas para nuestras maquinas virtuales en vagrant.
-  mkdir VMvagrant
-  cd VMvagrant
-  mkdir maquina-VCarga-2web-Mysql

Desde consola nos situamos dentro de la carpeta "maquina-VCarga-2web-Mysql" y lanzamos la siguientes lineas.

- vagrant init

Con esta linea de comando ya tendremos creado el archivo de configuracion "Vagrantfile" que nos permitirá configurar nuestros ubuntu server 18.04 el contendio de archivo es el siguiente:
~~~
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/xenial64"

  # Apache HTTP Server Valanceador de Carga
  config.vm.define "VCarga" do |app|
    app.vm.hostname = "VCarga"
    app.vm.network "private_network", ip: "192.168.33.10"
    app.vm.provision "shell", path: "provision/provision-for-valancer.sh"
  end
  
  # Apache HTTP Server
  config.vm.define "web" do |app|
    app.vm.hostname = "web"
    app.vm.network "private_network", ip: "192.168.33.11"
    app.vm.provision "shell", path: "provision/provision-for-apache.sh"
  end

  # Apache HTTP Server2
  config.vm.define "web2" do |app|
    app.vm.hostname = "web2"
    app.vm.network "private_network", ip: "192.168.33.12"
    app.vm.provision "shell", path: "provision/provision-for-apache.sh"
  end

  # MySQL Server
  config.vm.define "db" do |app|
    app.vm.hostname = "db"
    app.vm.network "private_network", ip: "192.168.33.13"
    app.vm.provision "shell", path: "provision/provision-for-mysql.sh"
  end

end
~~~
## Crear script para realizar "provision" en Vm "VCarga"

- Antes de crear el scrip para el valanceador de carga crearemos el archivo 000-default.conf que posteriormente remplazaremos por el que se encuentra en la ruta /etc/apache2/sites-enabled/000-default.conf
~~~
<VirtualHost *:80>
	# The ServerName directive sets the request scheme, hostname and port that
	# the server uses to identify itself. This is used when creating
	# redirection URLs. In the context of virtual hosts, the ServerName
	# specifies what hostname must appear in the request's Host: header to
	# match this virtual host. For the default virtual host (this file) this
	# value is not decisive as it is used as a last resort host regardless.
	# However, you must set it for any further virtual host explicitly.
	#ServerName www.example.com

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html

	# Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
	# error, crit, alert, emerg.
	# It is also possible to configure the loglevel for particular
	# modules, e.g.
	#LogLevel info ssl:warn

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined

	# For most configuration files from conf-available/, which are
	# enabled or disabled at a global level, it is possible to
	# include a line for only one particular virtual host. For example the
	# following line enables the CGI configuration for this host only
	# after it has been globally disabled with "a2disconf".
	#Include conf-available/serve-cgi-bin.conf

    <Proxy balancer://mycluster>
        # Server 1
        BalancerMember http://192.168.33.11

        # Server 2
        BalancerMember http://192.168.33.12
    </Proxy>

    ProxyPass / balancer://mycluster/

</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
~~~

- Una vez realizada la modificación del archivo 000-default.conf procedemos a crear el script para el box "Vcarga"
~~~
#!/bin/bash
apt-get update
apt-get install -y apache2
apt-get install -y php libapache2-mod-php php-mysql
a2enmod proxy deflate
a2enmod proxy_http deflate
a2enmod proxy_ajp deflate
a2enmod rewrite deflate
a2enmod deflate deflate
a2enmod headers deflate
a2enmod proxy_balancer deflate
a2enmod proxy_connect deflate
a2enmod proxy_html deflate
a2enmod lbmethod_byrequests deflate
rm -f /etc/apache2/sites-enabled/000-default.conf
cd /etc/apache2/sites-enabled
sudo cp /vagrant/config/000-default.conf .
sudo /etc/init.d/apache2 restart
~~~

## Crear script para realizar "provision" en Vm "web" y "web2"

- Dentro del archivo "Vagrantfile" definimos la ruta del script para realizar "provision" en nuestra maquina virtual. El contenido del script es el siguiente:
~~~
#!/bin/bash
apt-get update
apt-get install -y apache2
apt-get install -y php libapache2-mod-php php-mysql
sudo /etc/init.d/apache2 restart
cd /var/www/html
wget https://github.com/vrana/adminer/releases/download/v4.3.1/adminer-4.3.1-mysql.php
mv adminer-4.3.1-mysql.php adminer.php
apt-get install -y git
rm -f /var/www/html/index.html
cd /tmp
git clone https://github.com/josejuansanchez/iaw-practica-lamp.git
cp iaw-practica-lamp/src/. /var/www/html/ -R
cd /var/www/html
chown www-data:www-data * -R
apt-get install -y mysql-client-core-5.7
mysql -h 192.168.33.13 -u root -proot < /tmp/iaw-practica-lamp/db/database.sql
sed -i -e 's/localhost/192.168.33.13/' /var/www/html/config.php
sed -i -e 's/lamp_user/root/' /var/www/html/config.php
sed -i -e 's/lamp_user/root/' /var/www/html/config.php
~~~

## Crear script para realizar "provision" en Vm-"db"

- Dentro del archivo "Vagrantfile" definimos la ruta del script para realizar "provision" en nuestra maquina virtual. El contenido del script es el siguiente:
~~~
#!/bin/bash
apt-get update
apt-get -y install debconf-utils

DB_ROOT_PASSWD=root
debconf-set-selections <<< "mysql-server mysql-server/root_password password $DB_ROOT_PASSWD"
debconf-set-selections <<< "mysql-server mysql-server/root_password_again password $DB_ROOT_PASSWD"

apt-get install -y mysql-server
sed -i -e 's/127.0.0.1/0.0.0.0/' /etc/mysql/mysql.conf.d/mysqld.cnf
/etc/init.d/mysql restart

mysql -u root mysql -p$DB_ROOT_PASSWD <<< "GRANT ALL PRIVILEGES ON *.* TO root@'%' IDENTIFIED BY '$DB_ROOT_PASSWD'; FLUSH PRIVILEGES;"
~~~

## Inicio de la maquina virtual "db"

- vagrant up db

## Inicio de la maquina virtual "web"

- vagrant up web

## Inicio de la maquina virtual "web"

- vagrant up web2

## Inicio de la maquina virtual "web"

- vagrant up VCarga

## Repositorio github

https://github.com/luisminics/Practica-06-Balanceador-de-carga-con-Apache