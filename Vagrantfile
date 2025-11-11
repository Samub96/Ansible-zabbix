# Vagrantfile - Levanta 1 VM Ubuntu 20.04 y ejecuta ansible local
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.hostname = "zabbix-lab"

  # Red
  config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.network "forwarded_port", guest: 10051, host: 10051
  config.vm.network "forwarded_port", guest: 443, host: 8443
  
  # Red bridge para acceso a la red física
  config.vm.network "public_network", bridge: "auto", ip: "192.168.173.20"

  # Recursos
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 4096
    vb.cpus = 2
      end

  # Sincronizar la carpeta ansible dentro de la VM
  config.vm.synced_folder "./ansible", "/vagrant_ansible"

  # Provision con Ansible (ansible_local)
  config.vm.provision "ansible_local" do |ansible|
    ansible.playbook = "/vagrant_ansible/playbook.yml"
    ansible.inventory_path = "/vagrant_ansible/inventory.ini"
    ansible.install = true
    ansible.verbose = true
    ansible.limit   = "all"
  end
end
