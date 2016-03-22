class: center, middle

# Cours Openstack 2016

---
name: agenda
# Agenda

1. [Virtualisation](#virtualisation)
1. [Openstack](#openstack)
1. Keystone
1. Glance
1. Nova
1. Neutron
1. Cinder
1. Swift
1. Heat
1. CI/CD Openstack


---
name: virtualisation
#Virtualisation
##Rationnaliser l'IT
Pour assurer son bon fonctionnement, chaque application doit maitriser son environnement d'exécution (OS/Librairies).

Sans virtualisation chaque application nécéssite son serveur dédié.

- demander au service IT l'achat d'un serveur dédié;
- demander au service Réseau un port/VLan dédié;
- environnement contraint, peu flexible;
- on finit souvent par avoir un serveur surdimensionné (CPU/RAM/Disk) :

    - volontairement : pour ne pas avoir à le modifier en cas de forte charge de l'application;

    - involontairement : on utilise le matériel sourcé par le département IT, qui ne correspond pas forcément à mon besoin (trop de CPU/RAM/Disk)


Les datacenter deviennent alors une collection de serveur sous-exploités!!

---
#Virtualisation
##Rationnaliser l'IT
Au lieu d'avoir un serveur physique par application, on va virtualiser les resources d'un serveur pour mutualiser les coûts, mais en gardant la segmentation logicielle.

On obtient ainsi des Machines Virtuelles (VM):
- n'ayant accès qu'a une partie des resources (2 cpu sur 16, 2 Go de RAM /16Go, un disk de 20Go sur les 500Go dispo...)
- mais ayant une son propre environnement d'exécution (OS/Librairies)

---
#Virtualisation
##Technologies de virtualisation

Il faut donc diviser les resources physiques et ordonnancer leur acces entre chaque VM.
- l'odonnancement est gérée par l'hyperviseur; (KVM, Xen, ESXi...)
- l'émulation est gérée par l'émulateur :); (Qemu...)

<img src="https://upload.wikimedia.org/wikipedia/commons/5/5c/Diagramme_ArchiEmulateur.png" style="width: 500px;"/>
[//]: # (![](https://upload.wikimedia.org/wikipedia/commons/5/5c/Diagramme_ArchiEmulateur.png))

---
#Virtualisation
##Technologies de virtualisation
L'émulation peut être très couteuse en CPU lorsqu'il s'agit d'émuler un équipement différent de l'équipement physique sous-jacent (par exemple CPU ARM sur x86).

Mais, dans le monde cloud, on cherche surtout à diviser les équipements phyisques, sans compromettre l'étanchéité entre les VM. Les équipementier ont améliorer leur matériels jour pour optimiser le partage d'équipements physiques dans ce sens (Intel VT, AMD-V). Ainsi le surcoût de la virtualisation est acceptable.

---
#Virtualisation
##Gérer ses VM
On peut directement le faire avec kvm :
```basch
#qemu-img create -f qcow2 /tmp/img.qcow2 6G
Formatting '/tmp/img.qcow2', fmt=qcow2 size=6442450944 encryption=off cluster_size=65536 lazy_refcounts=off refcount_bits=16
#kvm -m 256 /tmp/img.qcow2
```

--
<img src="./img/kvm.png" style="width: 400px;"/>

```basch
#ps -aux | grep kvm
mat      17815  1.7  0.2 706368 48772 pts/5    Sl+  15:23   0:17 qemu-system-x86_64 -enable-kvm -m 256 /tmp/img.qcow2
```

---
#Virtualisation
##Libvirt
Plusieurs technologies de virtualisation existent (KVM, Xen, VirtualBox, LXC...).

Il est donc nécéssaire de créer une API pour abstaire ces technologies pour des applications de managment de VM. c'est le but du projet [libvirt](#https://libvirt.org/html/index.html)

Libvirt permet de gérer ces VM et leur ecosystème (reseau, storage) en fournissant un API, exposable sur le réseau.

Openstack ou d'autres outils de managment de VM peuvent alors utiliser cette API.

---
#Virtualisation
##Gestion des VM
Grace a des logiciels dédiés (virt-manager/libvirt, Virtual-box, VMWare Player...) la gestion de ces VM est simplifiée.

On peut rapidement créer une VM pour tester une nouvelle application sans compromettre l'hote. C'est très utile dans plusieurs cas :
- faire tourner des services linux sous Windows/Mac;
- installer des logiciels suspects;
- tester des applications dans compromettre son environnement;

--
<img src="./img/virt-manager.png" style="width: 400px;"/>

---
#Virtualisation
##Gestion des VM
Souvent, des TP d'applicatifs sont menés à partir de VM préconfigurées, incluant l'application installée sur la VM.

Ainsi on évite toute la partie configuration de l'application (plus pour l'admin) pour se concentrer sur son utilisation, toute en ayant une application par étudiant.

Pour cela l'outil plébiscité semble être Vagrant.
Chaque étudiant se verra remettre :
- un fichier descriptif de la (de) VM;
- un fichier servant de disque pour la (les) VM;

Vagrant utilisera alors VirtualBox (ou Libvirt via plugin dedié) pour booter ses VM telles qu'elles ont été définies dans le fichier descriptif.

---
#Virtualisation
##Gestion des VM
D'autres logiciels sont orientés vers la gestion de parc et d'hote de VM : Proxmox, OVirt(libvirt).

Ils permettent de rationnaliser davantage la gestion du parc de VM en les placant de manière optimisée en fonction de leur contraintes.

Quelques pratiques communes pour l'optimisation de l'utilisation des resources du DC :

- over-subscrition : le nombre de vCPU du total des VM placé sur l'hyperviseur est supérieur au nombre de CPU de l'hyperviseur;
- ballooning : autoriser les VM a dépasser la mémoire allouée pendant un bref instant;
- deduplication : copy-on-write sur les disques (ex QCOW2);

Si le nombre de resources vient a manquer, on peut toujours ajouter un serveur physique au cluster, et migrer à chaud (live-migrate) les VM qui en ont besoin. De même, les nouvelles VM seront programmées (schedulées) sur ce nouveau serveur.

Les resources physiques ne sont donc plus dédiées à un projet mais mutualisées pour plusieurs projets dans des DataCenter (DC) distants. C'est l'émergence du __Cloud Computing__

---
#Virtualisation
##du cadeau de noel au public Cloud

Ces techniques de virtualistion sont utilisées depuis longtemps chez les gros hébergeurs (GAFA), pour leur propre besoins interne.

la légende :
```
Amazon, dont le parc était sudimenssionné pour pouvoir répondre au pic
d'audience de noel, se retrouvait avec des datacenter sous exploités
tout le reste de l'année.
Ils ont donc créé une offre publique de location de ses resources :
c'est la naissance d'amazon EC2
```
D'autres Cloud provider proposent des services accessibles à la demande, comme du storage et des software. Elles rencontrent un grand succes et on parle alors de \*aaS.

---
#Virtualisation / \*aaS
##Concept \*_as_a_Service
* Service en ligne (__Cloud__);
* Privilégie la facturation à l'usage;
* Accessible via des APIs;


--
count: false

Différents types de \*aaS :
* Storage (__STaaS__) : Google Drive, Amazon S3, Dropbox

--
count: false

* Infra (__IaaS__) : Google Compute, Amazon EC2, Azure, Cloudwatt

--
count: false

* Platform (__PaaS__) : Heroku, Scalingo, Docker

--
count: false

* Software (__SaaS__) : Google Doc, Office 365, Photoshop


--
count: false
Différents modèles de facturation :
* Quantité de stockage, de cpu, de mémoire, de bande passante, d'utilisateurs
* Par minute, semaine, mois, an.

---
#Virtualisation / IaaS
##La NASA
Des son côté, la NASA utilise un clone d'EC2 pour son cloud privé, géré par Eucalyptus. Mais sans grande satisfaction, car trop fermé et pas assez modulaire.

Beaucoup de déploiement de cloud IaaS s'oriente vers VMWare qui monopolise le marché.
Un alternative Opensource est plus que nécéssaire pour concurrencer VMWare.

La Nasa lance donc un nouveau cloud manager opensource, plus flexible, appélé Nova. Rackspace, un gros acteur de l'hébergement aux USA se joint à l'éffort pour creer Openstack.
Très vite, il est décidé d'héberger le projet dans une fondation, afin d'en assurer l'indépendance, et de fédérer le plus d'entreprise.

La fondation Openstack est alors créee et l'engouement est rapide.

Une autre initiative Opensource voit également le jour : Cloudstack. Menée par Citrix, qui détiens également Xen, Cloudstack est actuellement hébergé par la fondation Apache.

---
name: openstack
#Openstack
##Introduction
But :
- service de IaaS : découper un datacenter physique en datacenter virtuel, allouable à la demande, accesisble par des API Rest. (schéma DC virtuel sur DC phy)
- devenir la plateforme de Cloud Computing de référence
- pour les grand déploiments (CERN, Nectar...);
- pour le cloud privé et public;
- Interopérabilité des APIs;

(schéma montrant la migration d'un cloud à l'autre)

---
#Openstack
##Fondation
- [4 open](#https://wiki.openstack.org/wiki/Open) :
    - Open Source : no Open Core; no entreprise edition.
    - Open Design : Design Summit tous les 6 mois;
    - Open Development : public code reviews, public roadmaps;
    - Open Community : meritocracy, élections des Leader par les developpeurs;
- faire vivre la communaté; (communication)
- s'assurer de l'intéropérabilité des API;
- s'assurer de la qualité du code;
- proteger la marque Openstack;

---
#Openstack
##Managment
- Technical committee
- User committee
...

---
#Openstack
##En chiffre
Openstack en chiffres : 
- +5000 contributeurs;
- ~300 company ont proposés des commits;
- ~1600 commit par mois;
source : [stackalitics.com](#www.stackalitics.com)
source : [activity.openstack.org](#http://activity.openstack.org/dash/browser/)

---
#Openstack
##Le code
- développé en python (2.7 + compatibilité python 3);
- avec des regles de codage fortes (pep8)
    - pour une meilleur maintenabilité;
    - pour une facilité d'adoption pour les newcomers;
- avec un grande couverture de tests unitaires et fonctionnels;
- sous licence Apache 2.0
    - modifiable et redistribuable;
---
#Openstack
##Le code
Openstack est composé de plusieurs services ayant en commun :
- l'apport d'une API REST (souvent calquée et compatible EC2);
- une architecture modulaire;
- une implémentation opensource de référence :
    - dans l'ADN d'openstack;
    - nécécaire pour les tests fonctionnels automatisés;

(schéma modularité...)

---
#Openstack
##Historique

(schéma...)

---
#Openstack
##Communauté
- Summit : 2 fois par an (+Midcycle)
- Mailing List : openstack; openstack-dev; openstack-operator....
- IRC : un chan par projet;
- réunion hebdo par projet et par sous-projet;
- ask.openstack.org;
---
#Openstack
##Mutliples composants

Chaque composant va fournir :
- des resources as a service(compute, storage, networks...);
- une API;
- une base de données dédiée;
- un projet (dev/roadmap..) dédié;

Les composants de base : 
- Keystone : gestion d'identité, et catalogue de service;
- Glance : gestion des images utilisées pour lancer une VM;
- Nova : gestion des VM;

Les composant additionnels : 
- Cinder : gestion des volume (disque dur, mode block) attaché à chaque VM:
- Neutron : gestion du réseaux et des ports de VMs;
- Horizon : interface graphique;
- Heat : Orchestartion de VM;
- Swift : gestion de stockage (mode fichier)

et bien d'autres...
---
#Openstack
##Architecture Simplifiée

http://git.openstack.org/cgit/openstack/operations-guide/plain/doc/openstack-ops/figures/osog_0001.png
---
#Openstack
##Architecture Operationnelle

<img src="http://git.openstack.org/cgit/openstack/operations-guide/plain/doc/openstack-ops/figures/osog_0001.png" style="width: 500px;"/>




##Keystone

---

##Glance

---

##Nova

---

##Neutron

---

##Cinder

---

##Swift

---

##Horizon

---

##Heat


