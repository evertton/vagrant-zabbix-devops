# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "generic/debian9"
  config.vm.box_check_update = true
  #config.vm.network "forwarded_port", guest: 80, host: 8080
  #config.vm.synced_folder "../data", "/vagrant_data"

  config.vm.provider "virtualbox" do |v|
   v.memory = 512
  end

  config.vm.provision "shell", inline: <<-SHELL
    set -x
    apt-get update -y && apt upgrade -y
    apt-get install -y nano build-essential
    updatedb
  SHELL

  config.vm.define "test", primary: true do |s|
      s.vm.hostname = "test"
      s.vm.network "private_network", ip: "192.168.111.9"
      s.vm.provision "shell", inline: <<-SHELL
        
      SHELL
  end
  
  config.vm.define "zabbix", primary: true do |s|
      s.vm.hostname = "zabbix"
      s.vm.network "private_network", ip: "192.168.111.10"
      s.vm.provision "shell", inline: <<-SHELL
        # Configura os repositórios do Zabbix no gestor de pacotes
        cd /tmp
        wget https://repo.zabbix.com/zabbix/4.0/debian/pool/main/z/zabbix-release/zabbix-release_4.0-2%2Bstretch.tar.gz
        tar zxvf zabbix-release_4.0-2+stretch.tar.gz
        mv zabbix-release/debian/zabbix.list /etc/apt/sources.list.d/
        sed -i "s/{version}/4.0/" /etc/apt/sources.list.d/zabbix.list
        sed -i "s/{distversion}/stretch/" /etc/apt/sources.list.d/zabbix.list
        sed -i "s/{distname}/debian/" /etc/apt/sources.list.d/zabbix.list
        apt-key add zabbix-release/debian/zabbix-official-repo.gpg

        # Instala os pacotes do Zabbix
        apt-get update
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
  
end