# High Availability Web Cluster

## Équipe

- FEUGERE Thibault
- LIJOUR Udo

## Choix technologique

Comme il l'était recommandé, nous sommes parti sur Vagrant malgré une hésitation avec Docker et son orchestrateur docker-compose.

L'hyperviseur choisit est VirtualBox.

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

| 10.0.0.11 | 10.0.0.12 | 10.0.0.21 | 10.0.0.22 |
|---|---|---|---|
| haproxy1 | haproxy2  |  web1 | web2 |
| State Master | State Backup | |

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

## Lancement des machines

Il nous suffit de faire `vagrant up`.

De plus, nous avons scripté tout le TP dans le Vagrantfile et nous avons ajouté les fichiers de conf au repo pour automatiser l'installation.

## Questions 

| Evenement                      | Evenement attendu | Evenement obtenu |
| ------------------------------ | ----------------- | ---------------- |
| Perte du premier noeud WEB     |         Si le Web 1 tombe, on veut que nos utilisateurs arrivent sur le Web 2          |     L'utilisateur est redirigé vers le site web 2, avec pour HTML test web2            |
| Perte du premier noeud HAPROXY |         Deuxième noeud Haproxy prend la relève          |     On garde l'IP virtuelle 10.0.0.10             |
| Perte du deuxième noeud WEB     |         Si le Web 2 tombe, on veut que nos utilisateurs arrivent sur le Web 1          |         L'utilisateur est redirigé vers le site web 1, avec pour HTML test web2         |



Pour le premier et troisième évènement, nous pouvons utiliser curl :

```
high-availability-web-cluster git:(main) ✗ curl 10.0.0.10
test web1
```

On down Web 1

```
high-availability-web-cluster git:(main) ✗ curl 10.0.0.10
test web2
```

On peut vérifier le second évènement avec un ping.

```
high-availability-web-cluster git:(main) ✗ vagrant ssh web1    
Last login: Thu Jan 13 15:51:41 2022 from 10.0.2.2
[vagrant@web1 ~]$ ping 10.0.0.10
PING 10.0.0.10 (10.0.0.10) 56(84) bytes of data.
64 bytes from 10.0.0.10: icmp_seq=1 ttl=64 time=1.00 ms
64 bytes from 10.0.0.10: icmp_seq=2 ttl=64 time=0.429 ms
64 bytes from 10.0.0.10: icmp_seq=3 ttl=64 time=0.383 ms
64 bytes from 10.0.0.10: icmp_seq=4 ttl=64 time=0.427 ms
```

On shutdown la première machine (haproxy1) :

```
--- 10.0.0.10 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3002ms
rtt min/avg/max/mdev = 0.383/0.560/1.002/0.256 ms
[vagrant@web1 ~]$ ping 10.0.0.10
PING 10.0.0.10 (10.0.0.10) 56(84) bytes of data.
64 bytes from 10.0.0.10: icmp_seq=1 ttl=64 time=0.400 ms
64 bytes from 10.0.0.10: icmp_seq=2 ttl=64 time=0.403 ms
64 bytes from 10.0.0.10: icmp_seq=3 ttl=64 time=0.442 ms
64 bytes from 10.0.0.10: icmp_seq=4 ttl=64 time=0.901 ms
```

### Est-ce que l'architecture déployée permet de faire du loadbalancing sur les deux services WEB ? 

Oui, le premier évènement et le troisième évènement montre un loadbalancing lorsqu'un des services tombe mais on peut alors imaginer que lors d'une montée de charge avec plusieurs visiteurs pour Croqc'malin on fasse de même.

### Quel est le rôle du service Keepalived dans notre infrastructure ?

La service keepalived nous permet d'avoir une adresse IP virtuelle, c'est l'outil qui rentre dans la partie PCA de l'infrastructure pour gérer si un noeud tombe.

Il a une fonctionnalité de vérification, on peut alors lui demander de vérifier l'état d'un réseau et donc d'un service à intervalle régulier. Dans notre configuration en question c'est toutes les 1 secondes que nous vérifions l'état réseau entre les deux loadbalancers.

### Quel est le rôle du service HAPROXY dans notre infrastructure ?

Le Haproxy est le mécanisme de loadbalancer, aussi appelé répartiteur de charge en français, pour savoir vers où on dirige le trafic. Il s'occupe de transférer les requêtes entre le client et le serveur web.

Les loadbalancers possèdent différents algorithmes comme `roundrobin`, `leastconn` ou encore `source`. Dans notre cas, comme vous pouvez le voir dans `haproxy.cfg`, nous utilisons `roundrobin`
