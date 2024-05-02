# -*- mode: ruby -*-
# vim: set ft=ruby :

ENV["LC_ALL"] = "en_US.UTF-8"
ENV['VAGRANT_SERVER_URL'] = 'https://vagrant.elab.pro'


MACHINES = {
  :router1 => {
        :box_name => "ubuntu/focal64",
        :vm_name => "router1",
        :net => [   
                    #ip, adpter, netmask, virtualbox__intnet
                    ["192.168.12.1", 2, "255.255.255.252",  "router1router2"], 
                    ["192.168.31.2", 3, "255.255.255.252",  "router1router3"], 
                    ["192.168.11.1", 4, "255.255.255.252",  "router1"], 
                    ["192.168.56.10", 8, "255.255.255.0"],                    
                ],
        :node_port_fwd => {
                          }
                
  },

  :router2 => {
        :box_name => "ubuntu/focal64",
        :vm_name => "router2",
        :net => [   
                    #ip, adpter, netmask, virtualbox__intnet
                    ["192.168.23.1", 2, "255.255.255.252",  "router2router3"], 
                    ["192.168.12.2", 3, "255.255.255.252",  "router1router2"], 
                    ["192.168.11.1", 4, "255.255.255.252",  "router2"], 
                    ["192.168.56.11", 8, "255.255.255.0"],                      
                ],
        :node_port_fwd => { 
                          }      

  },

  :router3 => {
        :box_name => "ubuntu/focal64",
        :vm_name => "router3",
        :net => [
                   ["192.168.31.1", 2, "255.255.255.252",  "router1router3"], 
                   ["192.168.23.2", 3, "255.255.255.252",  "router2router3"], 
                   ["192.168.33.1", 4, "255.255.255.252",  "router3"], 
                   ["192.168.56.12",  8, "255.255.255.0"],
                ],
        :node_port_fwd => {
                          }
  },

}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]
      
      box.vm.provider "virtualbox" do |v|
        v.memory = 2048
        v.cpus = 1
        
       end
   
       boxconfig[:net].each do |ipconf|
        box.vm.network("private_network", ip: ipconf[0], adapter: ipconf[1], netmask: ipconf[2], virtualbox__intnet: ipconf[3])
      end


      boxconfig[:node_port_fwd].each do |port, port_config|
        box.vm.network "forwarded_port", guest: port_config[:guest], host: port_config[:host]
      end

      if boxconfig.key?(:public)
        box.vm.network "public_network", boxconfig[:public]
      end

      box.vm.provision "shell", inline: <<-SHELL
        mkdir -p ~root/.ssh
        cp ~vagrant/.ssh/auth* ~root/.ssh
      SHELL

  #   Запуск ansible-playbook
     if boxconfig[:vm_name] == "router3"
        box.vm.provision "ansible" do |ansible|
         ansible.playbook = "ansible/provision.yaml"
         ansible.inventory_path = "ansible/hosts"
         ansible.host_key_checking = "false"
         ansible.limit = "all"
        end
      end
 
    end
  end
end
