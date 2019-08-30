# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "generic/debian9"
  config.vm.box_check_update = true
  #config.vm.network "forwarded_port", guest: 80, host: 8080
  #config.vm.synced_folder "../data", "/vagrant_data"

  config.vm.provision "shell", inline: <<-SHELL
    set -x

    # Configura os repositórios do Zabbix no gestor de pacotes
    cd /tmp
    wget https://repo.zabbix.com/zabbix/4.0/debian/pool/main/z/zabbix-release/zabbix-release_4.0-2%2Bstretch.tar.gz
    tar zxvf zabbix-release_4.0-2+stretch.tar.gz
    mv zabbix-release/debian/zabbix.list /etc/apt/sources.list.d/
    sed -i "s/{version}/4.0/" /etc/apt/sources.list.d/zabbix.list
    sed -i "s/{distversion}/stretch/" /etc/apt/sources.list.d/zabbix.list
    sed -i "s/{distname}/debian/" /etc/apt/sources.list.d/zabbix.list
    apt-key add zabbix-release/debian/zabbix-official-repo.gpg

    apt-get update -y && apt upgrade -y
    apt-get install -y nano build-essential
    updatedb
  SHELL

  config.vm.define "zabbix", primary: true do |s|
      s.vm.hostname = "zabbix"
      s.vm.network "private_network", ip: "192.168.111.10"
      s.vm.provider "virtualbox" do |v|
        v.memory = 256
      end
      s.vm.provision "shell", inline: <<-SHELL
        # Instala os pacotes do Zabbix
        apt-get install -y zabbix-server-mysql zabbix-frontend-php zabbix-agent

        # Cria e configura a base dados do Zabbix
        mariadb -e "create database zabbix character set utf8 collate utf8_bin;"
        mariadb -e "grant all privileges on zabbix.* to zabbix@localhost identified by 'zabbix';"
        zcat /usr/share/doc/zabbix-server-mysql/create.sql.gz | mysql -uzabbix -pzabbix zabbix

        # Configura o Zabbix Server
        sed -i "s/# DBHost=localhost/DBHost=localhost/" /etc/zabbix/zabbix_server.conf
        sed -i "s/# DBPassword=/DBPassword=zabbix/" /etc/zabbix/zabbix_server.conf


        # Instala dependências e configura o Zabbix Frontend
        apt-get install -y php7.0-bcmath php7.0-mbstring php-sabre-xml php7.0-mysql
        sed -i "s|# php_value date.timezone Europe/Riga|php_value date.timezone America/Maceio|" /etc/apache2/conf-enabled/zabbix.conf
        phpenmod pdo_mysql
        apache2ctl restart

        # Habilita os serviços
        systemctl enable zabbix-server
        systemctl enable zabbix-agent
        systemctl restart zabbix-server
        systemctl restart zabbix-agent
      SHELL
  end

  config.vm.define "docker", primary: true do |s|
      s.vm.hostname = "docker"
      s.vm.network "private_network", ip: "192.168.111.11"
      s.vm.provider "virtualbox" do |v|
        v.memory = 512
      end
      s.vm.provision "shell", inline: <<-SHELL
        apt-get install -y apt-transport-https ca-certificates curl software-properties-common gnupg2
        curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
        add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
        apt-get update
        apt-get install -y docker-ce

	adduser --system --shell /bin/bash --gecos 'Ansible' --group --home /home/ansible ansible
        echo -e "ansible\nansible" | passwd ansible

        docker run \
          --name=dockbix-agent-xxl \
          --net=host \
          --privileged \
          -v /:/rootfs \
          -v /var/run:/var/run \
          --restart unless-stopped \
          -e "ZA_Server=192.168.111.10" \
          -e "ZA_ServerActive=192.168.111.10" \
          -d monitoringartist/dockbix-agent-xxl-limited:latest
      SHELL
  end

  config.vm.define "ansible", primary: true do |s|
      s.vm.hostname = "ansible"
      s.vm.network "private_network", ip: "192.168.111.12"
      s.vm.provider "virtualbox" do |v|
        v.memory = 256
      end
      s.vm.provision "shell", inline: <<-SHELL
        apt-get install -y dirmngr zabbix-agent

        sed -i "s/Server=127.0.0.1/Server=192.168.111.10/" /etc/zabbix/zabbix_agentd.conf
        sed -i "s/Hostname=Zabbix server/Hostname=ansible/" /etc/zabbix/zabbix_agentd.conf
        systemctl enable zabbix-agent
        systemctl restart zabbix-agent
        
        echo "deb http://ppa.launchpad.net/ansible/ansible/ubuntu trusty main" >> /etc/apt/sources.list.d/ansible.list
        apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 93C4A3FD7BB9C367
        apt-get update
        apt-get install -y ansible
        
        echo "[dockerservers]" >> /etc/ansible/hosts
        echo "docker" >> /etc/ansible/hosts

        mkdir /etc/ansible/host_vars
        echo "ansible_ssh_host: 192.168.111.11" >> /etc/ansible/host_vars/docker
        echo "ansible_ssh_port: 22" >> /etc/ansible/host_vars/docker
        echo "ansible_ssh_user: ansible" >> /etc/ansible/host_vars/docker
        echo "ansible_ssh_private_key_file: /root/.ssh/docker" >> /etc/ansible/host_vars/docker

        ssh-keygen -N "" -f /root/.ssh/docker
        ssh-keyscan 192.168.111.11 >> /root/.ssh/known_hosts
        sshpass -p ansible ssh-copy-id -i /root/.ssh/docker ansible@192.168.111.11
      SHELL
  end

  config.vm.define "gitea", primary: true do |s|
      s.vm.hostname = "gitea"
      s.vm.network "private_network", ip: "192.168.111.13"
      s.vm.provider "virtualbox" do |v|
        v.memory = 256
      end
      s.vm.provision "shell", inline: <<-SHELL
        apt-get install -y nginx git mariadb-server mariadb-client expect zabbix-agent

        sed -i "s/Server=127.0.0.1/Server=192.168.111.10/" /etc/zabbix/zabbix_agentd.conf
        sed -i "s/Hostname=Zabbix server/Hostname=gitea/" /etc/zabbix/zabbix_agentd.conf
        systemctl enable zabbix-agent
        systemctl restart zabbix-agent

        systemctl enable nginx
        systemctl enable mariadb
        systemctl start nginx
        systemctl start mariadb

        expect -c "
        set timeout 10
        spawn mysql_secure_installation

        expect \"Enter current password for root \\\(enter for none\\\):\"
        send \"\r\"

        expect \"Set root password?\"
        send \"y\r\"

        expect \"New password:\"
        send \"root\r\"

        expect \"Re-enter new password:\"
        send \"root\r\"

        expect \"Remove anonymous users?\"
        send \"y\r\"

        expect \"Disallow root login remotely?\"
        send \"y\r\"

        expect \"Remove test database and access to it?\"
        send \"y\r\"

        expect \"Reload privilege tables now?\"
        send \"y\r\"

        expect eof
        "
        systemctl restart mariadb.service

        mysql -uroot -proot -e "CREATE DATABASE gitea;"
        mysql -uroot -proot -e "CREATE USER 'gitea'@'localhost' IDENTIFIED BY 'gitea';"
        mysql -uroot -proot -e "GRANT ALL ON gitea.* TO 'gitea'@'localhost' IDENTIFIED BY 'gitea' WITH GRANT OPTION;"
        mysql -uroot -proot -e "FLUSH PRIVILEGES;"

        adduser --system --shell /bin/bash --gecos 'Git Version Control' --group --disabled-password --home /home/git git
        mkdir -p /var/lib/gitea/{custom,data,indexers,public,log}
        chown git:git /var/lib/gitea/{data,indexers,log}
        chmod 750 /var/lib/gitea/{data,indexers,log}
        mkdir /etc/gitea
        chown root:git /etc/gitea
        chmod 770 /etc/gitea

        wget https://dl.gitea.io/gitea/master/gitea-master-linux-amd64 -O /usr/local/bin/gitea
        chmod a+x /usr/local/bin/gitea

        echo "[Unit]
Description=Gitea (Git with a cup of tea)
After=syslog.target
After=network.target
After=mariadb.service

[Service]
# Modify these two values and uncomment them if you have
# repos with lots of files and get an HTTP error 500 because
# of that
###
#LimitMEMLOCK=infinity
#LimitNOFILE=65535
RestartSec=2s
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
ExecStart=/usr/local/bin/gitea web -c /etc/gitea/app.ini
Restart=always
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea
# If you want to bind Gitea to a port below 1024 uncomment
# the two values below
###
#CapabilityBoundingSet=CAP_NET_BIND_SERVICE
#AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target" > /etc/systemd/system/gitea.service

        systemctl daemon-reload
        systemctl enable gitea
        systemctl start gitea

        rm /etc/nginx/sites-enabled/default

        echo "upstream gitea {
    server 127.0.0.1:3000;
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name gitea;
    root /var/lib/gitea/public;
    access_log off;
    error_log off;

    location / {
      try_files maintain.html \\\$uri \\\$uri/index.html @node;
    }

    location @node {
      client_max_body_size 0;
      proxy_pass http://localhost:3000;
      proxy_set_header X-Forwarded-For \\\$proxy_add_x_forwarded_for;
      proxy_set_header X-Real-IP \\\$remote_addr;
      proxy_set_header Host \\\$http_host;
      proxy_set_header X-Forwarded-Proto \\\$scheme;
      proxy_max_temp_file_size 0;
      proxy_redirect off;
      proxy_read_timeout 120;
    }
}" > /etc/nginx/sites-available/gitea

        ln -s /etc/nginx/sites-available/gitea /etc/nginx/sites-enabled/gitea
        systemctl reload nginx
      SHELL
  end

  config.vm.define "jenkins", primary: true do |s|
      s.vm.hostname = "jenkins"
      s.vm.network "private_network", ip: "192.168.111.14"
      s.vm.provider "virtualbox" do |v|
        v.memory = 384
      end
      s.vm.provision "shell", inline: <<-SHELL
        apt-get install -y default-jre nginx zabbix-agent ksh curl jq bc

        sed -i "s/Server=127.0.0.1/Server=192.168.111.10/" /etc/zabbix/zabbix_agentd.conf
        sed -i "s/Hostname=Zabbix server/Hostname=jenkins/" /etc/zabbix/zabbix_agentd.conf
        cd /tmp
        git clone https://github.com/evertton/jenkix.git
        ./jenkix/deploy_zabbix.sh -j "http://192.168.111.14" -u "evertton" -p "evertton"
        mkdir -p /etc/zabbix/scripts/agentd/jenkix/tmp
        chmod -R 777 /etc/zabbix/scripts/agentd/jenkix/tmp
        echo "zabbix ALL= NOPASSWD: /usr/bin/lsof, /bin/ps" >> /etc/sudoers
        systemctl enable zabbix-agent
        systemctl restart zabbix-agent

        wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
        echo "deb http://pkg.jenkins.io/debian-stable binary/" > /etc/apt/sources.list.d/jenkins.list
        apt-get update
        apt-get install -y jenkins

        echo 'upstream jenkins {
  keepalive 32; # keepalive connections
  server 127.0.0.1:8080; # jenkins ip and port
}

server {
  listen          80;       # Listen on port 80 for IPv4 requests
  server_name     jenkins;
  
  #this is the jenkins web root directory (mentioned in the /etc/default/jenkins file)
  root            /var/run/jenkins/war/;

  access_log      /var/log/nginx/access.log;
  error_log       /var/log/nginx/error.log;
  
  ignore_invalid_headers off; #pass through headers from Jenkins which are considered invalid by Nginx server.

  location ~ "^/static/[0-9a-fA-F]{8}\/(.*)$" {
    #rewrite all static files into requests to the root
    #E.g /static/12345678/css/something.css will become /css/something.css
    rewrite "^/static/[0-9a-fA-F]{8}\/(.*)" /$1 last;
  }

  location /userContent {
    #have nginx handle all the static requests to the userContent folder files
    #note : This is the $JENKINS_HOME dir
    root /var/lib/jenkins/;
    
    if (!-f $request_filename){
      #this file does not exist, might be a directory or a /**view** url
      rewrite (.*) /$1 last;
      break;
    }
    sendfile on;
  }

  location / {
      sendfile off;

      proxy_pass         http://127.0.0.1:8080;
      proxy_redirect     default;
      proxy_http_version 1.1;
      proxy_set_header   Host              $host;
      proxy_set_header   X-Real-IP         $remote_addr;
      proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
      proxy_set_header   X-Forwarded-Proto $scheme;
      proxy_max_temp_file_size 0;

      #this is the maximum upload size
      client_max_body_size       10m;
      client_body_buffer_size    128k;

      proxy_connect_timeout      90;
      proxy_send_timeout         90;
      proxy_read_timeout         90;
      proxy_buffering            off;
      proxy_request_buffering    off; # Required for HTTP CLI commands in Jenkins > 2.54
      proxy_set_header Connection ""; # Clear for keepalive
  }
}' > /etc/nginx/sites-available/jenkins

        rm /etc/nginx/sites-enabled/default
        ln -s /etc/nginx/sites-available/jenkins /etc/nginx/sites-enabled/jenkins
        systemctl reload nginx

        sleep 10
        echo "Jenkins Initial Password: $(cat /var/lib/jenkins/secrets/initialAdminPassword)"
      SHELL
  end
end
