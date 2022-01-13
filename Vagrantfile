Vagrant.configure("2") do |config|
  
  # Configuration commune à toutes les machines
  config.vm.box = "centos/7"

  # Ajoutez cette ligne afin d'accélérer le démarrage de la VM (si une erreur 'vbguest' est levée, voir la note un peu plus bas)
  #config.vbguest.auto_update = false
  # Désactive les updates auto qui peuvent ralentir le lancement de la machine
  config.vm.box_check_update = false 
  # La ligne suivante permet de désactiver le montage d'un dossier partagé (ne marche pas tout le temps directement suivant vos OS, versions d'OS, etc.)
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.provider "virtualbox" do |vb|
    vb.gui = true
  end

  # Config une première VM "haproxy1"
  config.vm.define "haproxy1" do |haproxy1|
    haproxy1.vm.network "private_network", ip: "10.0.0.11"
    haproxy1.vm.provision "shell",
      inline: "sudo hostnamectl set-hostname haproxy1"
  end

  # Config une  VM "haproxy2"
  config.vm.define "haproxy2" do |haproxy2|
    haproxy2.vm.network "private_network", ip: "10.0.0.12"
    haproxy2.vm.provision "shell",
      inline: "sudo hostnamectl set-hostname haproxy2"
  end

  # Config une  VM "web1"
  config.vm.define "web1" do |web1|
    web1.vm.network "private_network", ip: "10.0.0.21";
    web1.vm.provision "shell",
      inline: "sudo hostnamectl set-hostname web1"
  end

  # Config une  VM "web2"
  config.vm.define "web2" do |web2|
    web2.vm.network "private_network", ip: "10.0.0.22"
    web2.vm.provision "shell",
      inline: "sudo hostnamectl set-hostname web2"
  end
end
