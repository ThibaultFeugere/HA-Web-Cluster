# High Availability Web Cluster

## Équipe

- FEUGERE Thibault
- LIJOUR Udo

## Choix technologique

Comme il l'était recommandé, nous sommes parti sur Vagrant malgré une hésitation avec Docker et son orchestrateur docker-compose.

## Configuration de VirtualBox

Bien que Vagrant automatise de nombreuses tâches, la plage d'adresse IP nécessaire doit être changé au préalable pour éviter l'erreur : 

```
The IP address configured for the host-only network is not within the
allowed ranges. Please update the address used to be within the allowed
ranges and run the command again.

  Address: 10.0.0.11
  Ranges: 192.168.56.0/21

Valid ranges can be modified in the /etc/vbox/networks.conf file. For
more information including valid format see:

  https://www.virtualbox.org/manual/ch06.html#network_hostonly
``` 

On y apprend : `On Linux, Mac OS X and Solaris Oracle VM VirtualBox will only allow IP addresses in 192.168.56.0/21 range to be assigned to host-only adapters. For IPv6 only link-local addresses are allowed. If other ranges are desired, they can be enabled by creating /etc/vbox/networks.conf and specifying allowed ranges there.`.

Cela nous concerne puisque nous sommes sur MacOs, nous avons donc créé le fichier `/etc/vbox/networks.conf` et ajouté cette ligne :

`* 10.0.0.0/24`

Nous l'avons volontairement versionné sur le repository.

## Configuration Vagrant

```ruby
Vagrant.configure("2") do |config|
    config.vm.box = "centos/7"
  config.vm.box_check_update = false 
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

  config.vm.define "haproxy2" do |haproxy2|
    haproxy2.vm.network "private_network", ip: "10.0.0.12"
    haproxy2.vm.provision "shell",
      inline: "sudo hostnamectl set-hostname haproxy2"
  end

  config.vm.define "web1" do |web1|
    web1.vm.network "private_network", ip: "10.0.0.21";
    web1.vm.provision "shell",
      inline: "sudo hostnamectl set-hostname web1"
  end

  config.vm.define "web2" do |web2|
    web2.vm.network "private_network", ip: "10.0.0.22"
    web2.vm.provision "shell",
      inline: "sudo hostnamectl set-hostname web2"
  end
end
```

