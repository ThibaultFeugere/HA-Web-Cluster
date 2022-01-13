## Cluster web

Voici ce que l’on souhaite obtenir :

- Un service utilisateur hautement disponible avec des serveurs de type Postfix, Apache, Zimbra, Dovecot, etc… Dans notre exemple nous allons utiliser le service Web Apache.
- Les serveurs de ces backends devront être en mode actif/actif afin de pouvoir fournir un système apte à monter en charge
- La haute disponibilité sera gérée par des load balancer qui ne  devront pas être eux mêmes un SPOF, ils seront donc également load  balancés

Pour répondre à ces exigences nous allons monter une infrastructure web load balancés avec l'architecture suivante :

- haproxy1: loadbalancer avec keepalived et haproxy pour les services web (10.0.0.11)
- haproxy2: loadbalancer avec keepalived et haproxy pour les services web (10.0.0.12)
- web1: serveur web apache (10.0.0.21)
- web2: serveur web apache (10.0.0.22)

La VIP sur laquelle les utilisateurs viendront se connecter est la 10.0.0.10.

shared IP = 10.0.0.10

| 10.0.0.11 | 10.0.0.12 | 10.0.0.21 | 10.0.0.22 |
|---|---|---|---|
| haproxy1 | haproxy2  |  web1 | web2 |

<img src="http://www.morot.fr/wp-content/uploads/2016/09/ha-ka-haproxy-tproxy-1.png" alt="ha-ka-haproxy-tproxy" style="zoom:80%;" />



## OPTIONNEL :

Le Vagrantfile pour ce TP est le suivant :

```sh
Vagrant.configure("2") do |config|
  
  # Configuration commune à toutes les machines
  config.vm.box = "centos7-custom"

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
```

Pour lancer les machines vous devez vous placer dans le dossier du vagrantfile et faire la commande :

```sh
vagrant up #pour Virtualbox
vagrant up --provider vmware_desktop #pour Vmware workstation
```

Pour rentrer en ssh sur les machines vous devez utiliser la commande suivante :

```sh
vagrant ssh web1
vagrant ssh web2
...
```



## Déploiement HAPROXY1 et HAPROXY2

On commence par installer keepalived et haproxy sur les deux loadbalancer :

```sh
sudo yum install -y keepalived haproxy
sudo systemctl enable keepalived.service
sudo systemctl enable haproxy.service
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf # permet de dire au kernel qu'il peut transmettre les flux IP
echo "net.ipv4.ip_nonlocal_bind=1" >> /etc/sysctl.conf # permet au load balancer de bind une IP qui n'est pas locale pour le failover
```

Sur haproxy1 dans le fichier **/etc/keepalived/keepalived.conf** :

```sh
cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.old
```



Nous allons définir la configuration du Keepalive de la solution le noeud 1 sera maitre, le second sera de secours.

1. state MASTER

2. state BACKUP
   Sur le premier noeud éditer le fichier remplacer le contenu de /etc/keepalived/keepalived.conf par
   la configuration ci-dessous en changeant l'IP pour qu'elle corresponde à votre environnement.

  nota : vérifiez le nom de votre en interface, ici il s'agit de eth1 (visible via la cmmande nmcli con ou ip a)

Le service Keepalived va nous permettre de créer une VIP (Virtual IP) entre les 2 machines proxys.

```sh
vim /etc/keepalived/keepalived.conf
```

*petite astuce : avec vim pour effacer le contenu d'un fichier il faut taper dG*

on supprime le fichier et on met la configuration suivante :

```sh
vrrp_script reload_haproxy {
        script "killall -0 haproxy"
        interval 1
}

vrrp_instance VI_1 {
   virtual_router_id 100
   state MASTER
   priority 100
   # Check inter-load balancer toutes les 1 secondes
   advert_int 1
   # Synchro de l'état des connexions entre les LB sur l'interface eth1
   lvs_sync_daemon_interface eth1
   interface eth1
   # Authentification mutuelle entre les LB, identique sur les deux membres
   authentication {
        auth_type PASS
        auth_pass secret
   }
   # Interface réseau commune aux deux LB
   virtual_ipaddress {
        10.0.0.10/32 brd 10.0.0.255 scope global
   }

   track_script { 
       reload_haproxy 
   }

}
```

Redémarrer le service keepalive sur le premier serveur :

```sh
sudo systemctl restart keepalived.service
```

Sur haproxy2 dans **/etc/keepalived/keepalived.conf** :

```sh
vrrp_script reload_haproxy { 
        script "killall -0 haproxy"
        interval 1
} 

vrrp_instance VI_1 { 
   virtual_router_id 100 
   state BACKUP 
   priority 100 
   # Check inter-load balancer toutes les 1 secondes 
   advert_int 1 
   # Synchro de l'état des connexions entre les LB sur l'interface eth1 
   lvs_sync_daemon_interface eth1 
   interface eth1 
   # Authentification mutuelle entre les LB, identique sur les deux membres 
   authentication { 
        auth_type PASS 
        auth_pass secret 
   } 
   # Interface réseau commune aux deux LB 
   virtual_ipaddress { 
        10.0.0.10/32 brd 10.0.0.255 scope global 
   }

   track_script { 
       reload_haproxy 
   } 

}
```

Puis on redémarre le service sur le deuxième serveur :

```sh
sudo systemctl restart keepalived.service
```

On peut donc tester la bascule désormais en lançant un ping depuis web1 et en coupant le service keepalived sur le serveur haproxy1.

On passe maintenant à la configuration du haproxy !

Le rôle du haproxy ici sera de faire le loadbalancing au niveau des protocoles.

On sauvegarde la configuration par défaut :

```sh
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.old
```

Modifiez les lignes suivantes :

```sh
#modifiez la ligne : 
frontend  main *:500 en frontend  main *:80

#commentez la ligne 
use_backend static          if url_static

#à la fin du fichier de config
backend app
    balance     roundrobin
    server  app1 10.0.0.21:80 check
    server  app2 10.0.0.22:80 check
```

On peut maintenant installer le service Apache sur les deux serveurs web afin de tester la bascule entre les deux serveurs :

```sh
yum install httpd -y
systemctl start httpd
systemctl enable httpd
```

Sur web1 :

```sh
echo "test web1" > /var/www/html/index.html
```

Sur web2 :

```sh
echo "test web2" > /var/www/html/index.html
```

Et quand on ouvre sur le navigateur l'@IP :

http://10.0.0.10/



## Questions :

| Evenement                      | Evenement attendu | Evenement obtenu |
| ------------------------------ | ----------------- | ---------------- |
| Perte du premier noeud WEB     |                   |                  |
| Perte du premier noeud HAPROXY |                   |                  |

Est-ce que l'architecture déployée permet de faire du loadbalancing sur les deux services WEB ? 

Quel est le rôle du service Keepalived dans notre infrastructure ?

Quel est le rôle du service HAPROXY dans notre infrastructure ?
