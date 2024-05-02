# -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']
ENV["LC_ALL"] = "en_US.UTF-8"

MACHINES = {
  :inittest => {
        :box_name => "generic/centos8s",
        :box_version => "0",
        :ip_addr => '192.168.56.10'
  },
}

Vagrant.configure("2") do |config|
    MACHINES.each do |boxname, boxconfig|
  
        config.vm.define boxname do |box|
            config.vm.synced_folder ".", "/vagrant", disabled: false
            box.vm.box = boxconfig[:box_name]
            box.vm.host_name = boxname.to_s
            box.vm.network "private_network", ip: boxconfig[:ip_addr]
            box.vm.provider :virtualbox do |vb|
                vb.gui = true
		        vb.customize ["modifyvm", :id, "--memory", "512"]
            end
  
        box.vm.provision "shell", inline: <<-SHELL            
            sudo -i
            mkdir -p ~root/.ssh
            cp ~vagrant/.ssh/auth* ~root/.ssh
	    setenforce 0
	    systemctl stop firewalld 
            yum install epel-release -y
            yum install spawn-fcgi php php-cli mod_fcgid httpd nano -y
            cp /vagrant/watchlog /etc/sysconfig/
            cp /vagrant/watchlog.log /var/log/
            cp /vagrant/watchlog.sh /opt/
            cp /vagrant/watchlog.service /etc/systemd/system/
            cp /vagrant/watchlog.timer /etc/systemd/system/
            cp /vagrant/spawn-fcgi /etc/sysconfig
            cp /vagrant/spawn-fcgi.service /etc/systemd/system/
            cp /vagrant/httpd.service /etc/systemd/system/
            cp /vagrant/httpd-first /etc/sysconfig
            cp /vagrant/httpd-second /etc/sysconfig
            cp /vagrant/first.conf /etc/httpd/conf
            cp /vagrant/second.conf /etc/httpd/conf
          SHELL
        end
    end
  end
