# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
config.vm.define "web" do |subconfig|
        subconfig.vm.box = "centos/7"
        subconfig.vm.hostname = "web"
        subconfig.vm.network :private_network, ip: "192.168.50.10"
        subconfig.vm.provider "virtualbox" do |vb|
                vb.memory = "1024"
                vb.cpus = "1"
        end
end

config.vm.define "log" do |subconfig|
        subconfig.vm.box = "centos/7"
        subconfig.vm.hostname = "log"
        subconfig.vm.network :private_network, ip: "192.168.50.11"
        subconfig.vm.provider "virtualbox" do |vb|
                vb.memory = "1024"
                vb.cpus = "1"
        end
    end


#config.vm.define "vm-elk" do |subconfig|
#        subconfig.vm.box = "centos/7"
#        subconfig.vm.hostname = "vm-elk"
#        subconfig.vm.network :private_network, ip: "192.168.50.12"
#        subconfig.vm.provider "virtualbox" do |vb|
#                vb.memory = "1024"
#                vb.cpus = "1"
#        end
#    end



# --- Vagrant provision examples ---
#config.vm.provision "shell",inline: "yum install -y net-tools wget httpd"
#config.vm.provision "file", source: "./provisioning_files", destination: "/tmp/provisioning_files"
#config.vm.provision "shell",inline: "sudo mv /tmp/provisioning_files/watchdog.log /var/log"

end

