Vagrant.configure("2") do |config|
  
  # Backup Server (backup01)
  config.vm.define "backupserver" do |backupServer|
   backupServer.vm.box = "bento/ubuntu-22.04"

   backupServer.vm.box_check_update = false # Disabling the autolated upgrades 

    #backupServer.vm.network "forwarded_port", guest: 3306, host: 3307 # Mapping Ports 3306 to 3307 

   backupServer.vm.network "private_network", type: "static", ip: "172.16.238.12", netmask: "255.255.255.0" # fixing an Private StaticNetwork

    # Shared folder to the guest VM. 
    #backupServer.vm.synced_folder "./mysqldata", "/var/lib/mysql"

    # My Provider for Vagrant: BirtualBox
   backupServer.vm.provider "virtualbox" do |vb| 
      vb.gui = false # Don't display the VirtualBox GUI when booting the machine
      vb.name = "backup01" 
      vb.memory = "1024"
      vb.cpus = 1
    end
  end


  # Web Server (devapp01) : TOMCAT
  config.vm.define "webdevapp01" do |webserver|
    webserver.vm.box = "bento/ubuntu-22.04"
    webserver.vm.box_check_update = false
    webserver.vm.network "forwarded_port", guest: 8080, host: 8083
    webserver.vm.network "private_network", type: "static", ip: "172.16.238.10", netmask: "255.255.255.0"
    webserver.vm.synced_folder "./app", "/home/vagrant/app", create: true
    #webserver.vm.synced_folder "./srv-tomcatwebapps", "/opt/tomcat/webapps"
    #webserver.vm.synced_folder "./srv-tomcatconf", "/opt/tomcat/webapps"

    # Provisioning script to install Java 17, Tomcat 10.1.24, and Git
    webserver.vm.provision "shell", inline: <<-SHELL
      sudo apt update
      sudo apt install -y openjdk-17-jdk git

      # Install Tomcat 10.1.24
      sudo useradd -m -U -d /opt/tomcat -s /bin/false tomcat
      wget https://downloads.apache.org/tomcat/tomcat-10/v10.1.24/bin/apache-tomcat-10.1.24.tar.gz -P /tmp
      sudo tar xf /tmp/apache-tomcat-10.1.24.tar.gz -C /opt/tomcat
      sudo ln -s /opt/tomcat/apache-tomcat-10.1.24 /opt/tomcat/latest
      sudo chown -R tomcat: /opt/tomcat
      sudo sh -c 'chmod +x /opt/tomcat/latest/bin/*.sh'

      # Create a systemd service file for Tomcat
      sudo bash -c 'cat <<EOF > /etc/systemd/system/tomcat.service
      [Unit]
      Description=Apache Tomcat Web Application Container
      After=network.target

      [Service]
      Type=forking

      Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"
      Environment="CATALINA_PID=/opt/tomcat/latest/temp/tomcat.pid"
      Environment="CATALINA_HOME=/opt/tomcat/latest"
      Environment="CATALINA_BASE=/opt/tomcat/latest"
      Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"
      Environment="JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom"

      ExecStart=/opt/tomcat/latest/bin/startup.sh
      ExecStop=/opt/tomcat/latest/bin/shutdown.sh

      User=tomcat
      Group=tomcat
      UMask=0007
      RestartSec=10
      Restart=always

      [Install]
      WantedBy=multi-user.target
      EOF'

      sudo systemctl daemon-reload
      sudo systemctl enable tomcat
      sudo systemctl start tomcat

      # Setup cron job to backup code to backup01 every hour
      (crontab -l 2>/dev/null; echo "0 * * * * rsync -avz /home/vagrant/app vagrant@172.16.238.12:/home/vagrant/backup/app") | crontab -
    SHELL

    # My Provider for Vagrant: 
    webserver.vm.provider "virtualbox" do |vb| 
      vb.gui = false # Don't display the VirtualBox GUI when booting the machine
      vb.name = "devapp01" 
      vb.memory = "1024"
    end
  end


  # Database Server (dbapp01): PostgreSQL 
  config.vm.define "dbserver" do |serverPost|
    serverPost.vm.box = "bento/ubuntu-22.04"
    serverPost.vm.box_check_update = false # Disabling the autolated upgrades 
    serverPost.vm.network "forwarded_port", guest: 5432, host: 5433 # Mapping Ports 5432 of the VM to 5433 of the Host
    serverPost.vm.network "private_network", type: "static", ip: "172.16.238.11", netmask: "255.255.255.0" # fixing an Private StaticNetwork

    # Provisioning script to install PostgreSQL
    serverPost.vm.provision "shell", inline: <<-SHELL
      sudo apt update
      sudo apt install -y postgresql postgresql-contrib

      # Stop the PostgreSQL service
      sudo systemctl stop postgresql

      # Change the owner of the synced folder to postgres
      sudo chown -R postgres:postgres /var/lib/postgresql

      # Initialize the PostgreSQL data directory
      sudo -u postgres /usr/lib/postgresql/14/bin/initdb -D /var/lib/postgresql/14/main

      # Configuration to allow access only from devapp01
      echo "host all all 172.16.238.10/24 md5" | sudo tee -a /etc/postgresql/*/main/pg_hba.conf
      echo "listen_addresses = '*'" | sudo tee -a /etc/postgresql/*/main/postgresql.conf
      
      # Setup cron job to backup database to backup01 every 30 minutes
      (crontab -l 2>/dev/null; echo "*/30 * * * * pg_dumpall -U postgres | gzip > /tmp/db_backup.gz && rsync -avz /tmp/db_backup.gz vagrant@172.16.238.12:/home/vagrant/backup/db_backup.gz") | crontab -
    
      # Start the PostgreSQL service
      sudo systemctl start postgresql
    SHELL

    # Shared folder to the guest VM. 
    serverPost.vm.synced_folder "./pgdata", "/var/lib/postgresql", owner: "postgres", group: "postgres"

    # My Provider for Vagrant: BirtualBox
    serverPost.vm.provider "virtualbox" do |vb| 
      vb.gui = false # Don't display the VirtualBox GUI when booting the machine
      vb.name = "dbapp01" 
      vb.memory = "1024"
      vb.cpus = 1
    end
  end


  
end