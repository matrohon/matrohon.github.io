class: center, middle

# Cours Openstack 2016

---
#Présentation
Mathieu Rohon

OrangeLabs

Contributeur Openstack depuis 2012

Principale actvité : Openstack/Neutron

---
# Scope du cours

- Virtualisation
- Cloud Computing/IaaS
- Datacenter Virtuel
- __Opensource__
- __Openstack__

---
name: agenda
# Agenda

1. [Cloud Computing](#cloud)
1. [Virtualisation](#virtualisation)
1. [Openstack](#openstack)
    1. [Keystone](#keystone)
    1. [Glance](#glance)
    1. [Nova](#nova)
    1. Neutron
    1. Cinder
    1. Swift
    1. Heat
    1. CI/CD Openstack

---
template: agenda

###.right[Cloud Computing]

---
name: cloud
#Cloud computing
##Avant le Cloud computing

Pour assurer son bon fonctionnement, une application doit maitriser son environnement :
- resources physiques (CPU/RAM);
- stockage (disque, backup);
- OS/Librairies/Soft sur chaque serveur;
- accès réseau (IP Publique, firewall, load balancer);

Chaque application nécéssitera donc un environnement dédié. 

---
#Cloud computing
##Avant le Cloud computing

Pour les petites entreprises sans service IT, le développeur d'application doit se transformer en admin système/réseau/sotackage pour déployer son environnement.

Dans de plus grosses entreprises, le développeur d'application passera par les étapes suivantes :

- demander au service IT l'achat d'un serveur dédié;
- demander au service Réseau un port/VLan dédié, et le réglage des firewall/load balancer...
- demander au service Stockage un environnement backuper pour stocker ses données;

---
#Cloud computing
##Avant le Cloud computing

Ces étapes sont souvent longues, donc peu flexibles : difficile de redimensionner rapidement son environnement en cas de pic ou de baisse d'activité de l'application;

On finit souvent par avoir un environnement surdimensionné (CPU/RAM/Disk) :
- volontairement : pour ne pas avoir à le modifier en cas de forte charge de l'application;
- involontairement : on utilise le matériel sourcé par le département IT/Réseau/Stockage, qui ne correspond pas forcément au besoin (trop de CPU/RAM/Disk)

Les datacenter deviennent alors une collection de serveurs sous-exploités!!

---
#Cloud computing

Grâce au cloud computing, on va __flexibiliser__ l'accès aux resources, et __optimiser__ leur utilisation;

[Definition Wikipedia](https://en.wikipedia.org/wiki/Cloud_computing) :
- On demand : utlisation et facturation uniquement le temps nécéssaire.
- Shared processing resources and data : mutualisation des infrastructures;

Au lieu d'avoir une infrastructure physique par projet, chaque projet va louer son infrastructure chez un prestataire tiers (interne ou externe à l'entreprise) :
- pas de compétences nécéssaire d'administration de l'infrastructure dans le projet;
- prévision des couts "pay as you go";
- adaptation rapide en cas de pic/baisse d'activité du projet;

---
#Cloud computing
##Concept \*_as_a_Service
* Service en ligne (__Cloud__);
* Privilégie la facturation à l'usage;
* Accessible via des APIs;

--
count: false

Différents types de \*aaS :
* Storage (__STaaS__) : Swift
    - offres commerciales : Google Drive, Amazon S3, Dropbox

--
count: false

* Infra (__IaaS__) : Openstack, Cloudstack, VMWare
    - offres commerciales : Google Compute, Amazon EC2, Azure, Cloudwatt

--
count: false

* Platform (__PaaS__) : Cloud foundry
    - offres commerciales : Heroku, Scalingo, Docker

--
count: false

* Software (__SaaS__) : ?
    - offres commerciales : Google Doc, Office 365, Photoshop


--
count: false
Et bien d'autres (VPNaas, DBaas...)


---
#Cloud computing
##Rationnaliser l'IT

Si on s'appuie sur des éléments physiques, il est très difficile d'obtenir la flexibilité.
Mettre a disposition un serveur physique nécéssite des étapes manuelles.

Côté réseau et stockage, des technologie de virtualisation existent et peuvent assez facilement être pilotée via des API en mode aaS :
- Réseau virtuel : segmentation logiciel
- Stockage virtuel : NFS, cifs (file), LVM (block)

Grâce aux nouvelles technologies de virtualisation d'OS, on va pouvoir créer des serveurs virtuels, beaucoup plus simples à piloter via des API en mode aaS;

---
template: agenda

###.right[Virtualisation]

---
name: virtualisation
#Virtualisation
##La machine virtuelle (VM)

- Un système d'exploitation (OS) tournant comme un simple logiciel;
- l'OS n'a pas conscience d'être virtualisé par un autre OS;
- on parlera de __GUEST__ ou de __VM__ pour l'OS virtuel, et de __HYPREVISEUR__ ou de __HOST__ pour le système sous-jacent;
- la comminication inter-VM se fait __exclusivement par le réseau__, même si les VMs sont sur le même HOST;
- les VMs sont __isolées du host__ et n'ont donc pas accès au matériel;
- nécéssite d'émuler le matériel (CPU/RAM/IO/GPU)

<p style="text-align:center;"><img src="https://upload.wikimedia.org/wikipedia/commons/5/5c/Diagramme_ArchiEmulateur.png" width="400px"/></p>

---
#Virtualisation
##L'hyperviseur
L'hyperviseur a pour rôle de gérer les VMs et d'émuler les accès des VMs au matériel.

Historiquement, il existe 2 types d'hyperviseurs : 
- type 1 : l'hyperviseur est à la place de l'OS (VMWare ESXi, XEN)
- type 2 : l'hyperviseur tourne comme un logiciel sur un OS existant (VirtualBox, QEMU)

<p style="text-align:center;"><img src="https://upload.wikimedia.org/wikipedia/commons/e/e1/Hyperviseur.png" width="400px"/></p>

- type hybride : l'hyperviseur tourne dans le noyau (module linux) de l'OS existant (kvm)
---
#Virtualisation
##L'émulation

La solution basique pour créer une VM consiste donc à émuler son matériel :

- Solution opensource : __Qemu__
- Avantage :
    - Permet de faire tourner du matériel différent dans la VM de celui présent sur le HOST (ex CPU ARM sur x86)
- Inconvénient :
    - __très couteuse en CPU__
    - toutes les instructions CPU doivent êter réinterpretées

---
#Virtualisation
##L'émulation iso-matériel


Mais dans le CLoud Computing ce qui va nous intéresse c'est d'isoler les VMs, et non d'émuler du matériel.


Si ma VM connait le matériel du HOST, je n'ai plus les problèmes de performance?

---
#Virtualisation
##L'émulation iso-matériel

Malheureusement __non__ :
- l'OS du HOST cherche a executer des instructions privilégiées sur le CPU (protected mode, Ring 0), mais n'y est pas authorisé en tant que logiciel (Ring 3)

<p style="text-align:center;"><img src="https://upload.wikimedia.org/wikipedia/commons/thumb/2/2f/Priv_rings.svg/633px-Priv_rings.svg.png" width="300px;"/></p>
- Pas d'accès direct au matériel donc :
    - Pas d'accès performant à la RAM (chipset MMU): nécéssite l'émulation de ce chipset par le HOST
    - Pas d'accès performant (DMA) aux équipements d'entée/sortie (disque, carte réseau...)

---
#Virtualisation
##L'émulation iso-matériel aidée

La paravirtualisation (Xen):
- les Guests sont au courant qu'ils sont virtualisés;
- au lieu d'appeler les instructions couteuses à émuler, ils vont demander à l'Hyperviseur de les executer à leur place;
- VirtualBox guest additions?


<p style="text-align:center;"><img src="http://www.ibm.com/developerworks/library/l-virtio/figure1.gif" width="600px;"/></p>

---
#Virtualisation
##L'émulation iso-matériel aidée
Les améliorations matérielles :
- Intel VT-x / AMD-V / VIA VT : authoriser les VMs à exécuter les instructions privilégiées;
- Intel EPT / AMD RVI : authorise les VMs à acceder directement à la MMU;
- Intel VT-d / AMD-Vi : permet l'accès des VMs au DMA et redirige les intérruptions : on peut assigner un périphérique PCI à une VM;
- SRIOV : carte PCI-Express pouvant être divisée; chaque partie peut alors être affectée à un VMs;

Ces améliorations matérielles sont nativement intégrées dans les équipements récents (à activer dans le BIOS) et les hyperviseurs savent en tirer partie, ce qui rend la virtualisation __beaucoup moins couteuse en CPU__

---
#Virtualisation
##Créer ses VMs
On peut directement le faire avec kvm :
```sh
#qemu-img create -f qcow2 /tmp/img.qcow2 6G
Formatting '/tmp/img.qcow2', fmt=qcow2 size=6442450944 encryption=off cluster_size=65536 lazy_refcounts=off refcount_bits=16
#kvm -m 256 /tmp/img.qcow2
```

--
count: false

<p style="text-align:center;"><img src="./img/kvm.png" style="width: 400px;"/></p>

```sh
#ps -aux | grep kvm
mat      17815  1.7  0.2 706368 48772 pts/5    Sl+  15:23   0:17 qemu-system-x86_64 -enable-kvm -m 256 /tmp/img.qcow2
```

---
#Virtualisation
##Libvirt
Plusieurs technologies de virtualisation existent (KVM, Xen, VirtualBox, LXC...).

Il est donc nécéssaire de créer une API pour abstraire ces technologies pour des applications de managment de VM. c'est le but du projet [libvirt](https://libvirt.org/html/index.html)

Libvirt permet de gérer ces VM et leur ecosystème (reseau, storage) en fournissant un API, exposable sur le réseau.

Openstack ou d'autres outils de managment de VM peuvent alors utiliser cette API.

---
#Virtualisation
##Gestion des VMs
Grace a des logiciels dédiés (virt-manager/libvirt, Virtual-box, VMWare Player...) la gestion de ces VMs est simplifiée.

On peut rapidement créer une VM pour tester une nouvelle application sans compromettre l'hote. C'est très utile dans plusieurs cas :
- faire tourner des services linux sous Windows/Mac;
- installer des logiciels suspects;
- tester des applications sans compromettre son environnement;

--
count: false

<p style="text-align:center;"><img src="./img/virt-manager.png" style="width: 400px;"/></p>

---
#Virtualisation
##Gestion des VMs
Souvent, des TP d'applicatifs sont menés à partir de VM préconfigurées, incluant l'application installée sur la VM.

Ainsi on évite toute la partie configuration de l'application (plus pour l'admin) pour se concentrer sur son utilisation, toute en ayant une application par étudiant.

Pour cela l'outil plébiscité semble être Vagrant.
Chaque étudiant se verra remettre :
- un fichier descriptif de la (des) VM;
- un fichier servant de disque pour la (les) VM;

Vagrant utilisera alors VirtualBox (ou Libvirt via plugin dedié) pour booter ses VMs telles qu'elles ont été définies dans le fichier descriptif.

---
#Virtualisation
##Gestion des VMs
D'autres logiciels sont orientés vers la gestion de parc et d'hote de VMs : Proxmox, OVirt(libvirt).

Ils permettent de rationnaliser davantage la gestion du parc de VMs en les placant de manière optimisée en fonction de leur contraintes.

Quelques pratiques communes pour l'optimisation de l'utilisation des resources du DC :

- over-subscrition : le nombre de vCPU du total des VMs placée sur l'hyperviseur est supérieur au nombre de CPUs de l'hyperviseur;
- ballooning : autoriser les VMs a dépasser la mémoire allouée pendant un bref instant;
- copy-on-write sur les disques (ex QCOW2);

Si le nombre de resources vient a manquer, on peut toujours ajouter un serveur physique au cluster, et migrer à chaud (live-migrate) les VMs qui en ont besoin. De même, les nouvelles VMs seront programmées (schedulées) sur ce nouveau serveur.

Les resources physiques ne sont donc plus dédiées à un projet mais mutualisées pour plusieurs projets dans des DataCenter (DC) distants. C'est l'émergence du __Cloud Computing__

---
template: agenda

###.right[Openstack]

---
#Virtualisation
##Du cadeau de noel au public Cloud

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
#Virtualisation / IaaS
##La NASA
De son côté, la NASA utilise un clone d'EC2 pour son cloud privé, géré par Eucalyptus. Mais sans grande satisfaction, car trop fermé et pas assez modulaire.

Beaucoup de déploiement de cloud IaaS s'oriente vers VMWare qui monopolise le marché.
Un alternative Opensource est plus que nécéssaire pour concurrencer VMWare.

La Nasa lance donc un nouveau cloud manager opensource, plus flexible, appélé Nova. Rackspace, un gros acteur de l'hébergement aux USA se joint à l'éffort pour creer Openstack.
Très vite, il est décidé d'héberger le projet dans une fondation, afin d'en assurer l'indépendance, et de fédérer le plus d'entreprise.

La fondation Openstack est alors créée et l'engouement est rapide.

Une autre initiative Opensource voit également le jour : Cloudstack. Menée par Citrix, qui détiens également Xen, Cloudstack est actuellement hébergé par la fondation Apache.

---
name: openstack
#Openstack
##Introduction
But :
- service de IaaS : découper un datacenter physique en datacenters virtuels, allouables à la demande, accessible par des API REST;
- devenir la plateforme de Cloud Computing de référence;
- pour les grand déploiments (CERN, Nectar...);
- pour le cloud privé et public;
- Interopérabilité des APIs;

(schéma montrant la migration d'un cloud à l'autre)

---
#Openstack
##Fondation
- [4 open](https://wiki.openstack.org/wiki/Open) :
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
##Management
- Membres individuels : tous les developpeurs et autres...
- [Entreprises](http://www.openstack.org/foundation/companies/)
    - Platinium, Gold, Corporate
- Board of directors : 
    - gère la fondation (budget, marque, etc...)
    - Platinium Directors (1 par membre platinium)
    - Gold Directors élus par les membres Gold
    - Individual Directors élus par les membres individuels
- Technical committee
    - membres élus par les développeurs
    - gère la cohérence technique d'Openstack 
- User committee
    - membres individuels élus
    - representent les utilisateurs

source : [openstack.org](http://www.openstack.org/legal/bylaws-of-the-openstack-foundation/)

---
#Openstack
##En chiffre
Openstack en chiffres : 
- +5000 contributeurs;
- ~300 company ont proposés des commits;
- ~1600 commit par mois;
source : [stackalitics.com](www.stackalitics.com)
source : [activity.openstack.org](http://activity.openstack.org/dash/browser/)

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
- un projet (dev/roadmap/git..) dédié (launchpad/git/gerrit);
- des resources as a service(compute, storage, networks...);
- une [API REST] (https://en.wikipedia.org/wiki/Representational_state_transfer) (souvent calquée et compatible EC2);
- une base de données dédiée;
- une architecture modulaire;
- une implémentation opensource de référence :
    - dans l'ADN d'openstack;
    - nécécaire pour les tests fonctionnels automatisés;
---
#Openstack
##Architecture d'un composant
<p style="text-align:center;"><img src="./img/OpenstackProjectDesign.png" style="width: 500px;"/></p>

---
#Openstack
##Architecture nova/libvirt/kvm/mariadb :
<p style="text-align:center;"><img src="./img/OpenstackProjectDesign_libvirt.png" style="width: 500px;"/></p>

---
#Openstack
##Architecture nova/VMWare/postgre :
<p style="text-align:center;"><img src="./img/OpenstackProjectDesign_vmware.png" style="width: 500px;"/></p>

---
#Openstack
##Mutliples composants
Chaque composant utilise le framework python Web Server Gateway Interface ([WSGI](http://wsgi.readthedocs.org/en/latest/)), configuré via la lib paste.deploy. Cette configuration est accessible via le fichier /etc/composant/*-paste.ini.

Beaucoup de composants vont avoir le même genre de tache effectuer (gestion du fichier de config, gestion des log etc....). Pour factoriser ce travail, un ensemble de librairie sont proposées dans le projet transverse Openstack oslo.

De même afin d'éviter des problèe d'incompatibilité de dépendances entre les projets Openstack, un pojet transverse appelé "requirements" à été créé. Lorqu'un projet veut utiliser/modifier une dépendance envers une librairy, il le fera via ce projet.

Chaque composant peut gérér l'accès à ces API en fonction de l'utilisateur et de son rôle. Il le fait via son fichier /etc/composant/policy.json

Enfin, chaque composant fournit un client CLI qui facilite l'utilisation de l'API REST via la ligne de commande.

---
#Openstack
##Mutliples composants
Les composants de base : 
- Keystone : gestion d'identité, et catalogue de service;
- Glance : gestion des images utilisées pour lancer une VM;
- Nova : gestion des VM;

Les composants additionnels : 
- Cinder : gestion des volume (disque dur, mode block) attaché à chaque VM:
- Neutron : gestion du réseaux et des ports de VMs;
- Horizon : interface graphique;
- Heat : Orchestartion de VM;
- Swift : gestion de stockage (mode fichier)

et [bien d'autres](http://governance.openstack.org/reference/projects/index.html)...

---
#Openstack
##Mutliples composants

Historiquement les projet openstack se divisant entre : 
- integrated
- incubated
- stackforge

Il devenait trop difficile de déterminer quels projets devaient être "integrated", alors Openstack est passé en mode BigTent :

- Tous projets relatifs à Openstack et respectant les 4 open peuvent devenir un projet openstack officiel;
- La notion de DefCore persiste : éléments indispensable d'un cloud Openstack (nova, glance, keystone...);

---
#Openstack
##Architecture Simplifiée

<p style="text-align:center;"><img src="http://docs.openstack.org/juno/install-guide/install/apt/content/figures/1/a/common/figures/openstack_havana_conceptual_arch.png" style="width: 500px;"/></p>
---
#Openstack
##Architecture Operationnelle

<p style="text-align:center;"><img src="http://git.openstack.org/cgit/openstack/operations-guide/plain/doc/openstack-ops/figures/osog_0001.png" style="width: 600px;"/></p>

---
#Openstack
##Demo
Dans cette première démo, nous allons créer une VM, et étudier les étapes et les composants indispensables au boot d'une VM dans Openstack. 

---
class: center
#Openstack
##Demo - Login

<img src="./img/horizon_login.png" style="width: 300px;"/>

---
class: center
#Openstack
##Demo - Tableau de bord

<img src="./img/horizon_dashboard.png" style="width: 700px;"/>

---
class: center
#Openstack
##Demo - Creation d'une VM

<img src="./img/horizon_dashboard_instances.png" style="width: 700px;"/>

---
class: center
#Openstack
##Demo - Creation d'une VM

<img src="./img/horizon_launch_instances1.png" style="width: 700px;"/>

---
class: center
#Openstack
##Demo - Creation d'une VM

<img src="./img/horizon_launch_instances2.png" style="width: 700px;"/>

---
class: center
#Openstack
##Demo - Creation d'une VM

<img src="./img/horizon_launch_instances2.png" style="width: 700px;"/>

---
class: center
#Openstack
##Demo - Creation d'une VM

<img src="./img/horizon_running_instance.png" style="width: 700px;"/>

---
#Openstack
##Demo

Dans cette simple démo, nous avons utilisés 3 composants indispensables pour booter une VM : 
- keystone : pour s'identifier;
- glance : pour choisir l'image de base;
- nova : pour créer la VM;

Nous avons également utilisé un composant facultatif : 
- horizon : le tableau de bord

---
#Openstack
##Demo

Toute cette démo aurait pu se passer d'horizon si on avait utilisé notre login/password conjointement aux clients en ligne de commande de :

- glance pour connaitre les images disponibles;
```sh
$ glance --os-username demo --os-password labo --os-tenant-name demo --os-auth-url http://192.168.122.241:5000/ image-list
+--------------------------------------+---------------------------------+
| ID                                   | Name                            |
+--------------------------------------+---------------------------------+
| b6305617-0451-46d8-98fd-2fde8f1a9c98 | cirros-0.3.4-x86_64-uec         |
+--------------------------------------+---------------------------------+
```

---
#Openstack
##Demo
- nova pour lister les flavor;

```sh
$ nova --os-username demo --os-password labo --os-tenant-name demo --os-auth-url http://192.168.122.241:5000/ flavor-list
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| 1  | m1.tiny   | 512       | 1    | 0         |      | 1     | 1.0         | True      |
| 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | True      |
| 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | True      |
| 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | True      |
| 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | True      |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
``` 

---
#Openstack
##Demo
- nova pour booter la VM;
```sh
$ nova --os-username demo --os-password labo --os-tenant-name demo --os-auth-url http://192.168.122.241:5000/ boot --flavor 1 --image cirros-0.3.4-x86_64-uec vm1
+--------------------------------------+----------------------------------------------------------------+
| Property                             | Value                                                          |
+--------------------------------------+----------------------------------------------------------------+
| OS-EXT-STS:vm_state                  | building                                                       |
....
| id                                   | ed8af0fe-9817-4533-a599-c8dafe23c66b                           |
| image                                | cirros-0.3.4-x86_64-uec (b6305617-0451-46d8-98fd-2fde8f1a9c98) |
....
| name                                 | vm1                                                            |
| status                               | BUILD                                                          |
...
+--------------------------------------+----------------------------------------------------------------+
```

---
#Openstack
##Demo
- nova pour afficher le status de la VM;
```sh
$ nova --os-username demo --os-password labo --os-tenant-name demo --os-auth-url http://192.168.122.241:5000/ list
+--------------------------------------+------+--------+------------+-------------+------------------+
| ID                                   | Name | Status | Task State | Power State | Networks         |
+--------------------------------------+------+--------+------------+-------------+------------------+
| ed8af0fe-9817-4533-a599-c8dafe23c66b | vm1  | ACTIVE | -          | Running     | private=10.0.0.3 |
+--------------------------------------+------+--------+------------+-------------+------------------+
```

---
#Openstack
##Demo

Par commodité, les clients des composants peuvent utliser des variables d'environnement : 

```sh
$ env | grep OS
OS_PASSWORD=labo
OS_AUTH_URL=http://192.168.122.241:5000/v2.0
OS_USERNAME=demo
OS_TENANT_NAME=demo

$ nova list
+--------------------------------------+------+--------+------------+-------------+------------------+
| ID                                   | Name | Status | Task State | Power State | Networks         |
+--------------------------------------+------+--------+------------+-------------+------------------+
| ed8af0fe-9817-4533-a599-c8dafe23c66b | vm1  | ACTIVE | -          | Running     | private=10.0.0.3 |
+--------------------------------------+------+--------+------------+-------------+------------------+
```

---
template: agenda

###.right[Openstack - Keystone]

---
name: keystone
#Openstack
##Keystone

Avant de pouvoir utiliser un cloud Opensatck, il faut __s'identifier__!!

Keystone est l'une des brique de base d'openstack, et est indispensable à son fonctionnement;

Keystone gère : 
- la liste des utilisateurs;
- la liste des tenants/projets;
- la liste des roles;
- la correspondance entre les rôles<->utilisateurs<->tenants
- le catalogue des services Openstack;
- l'authentification des utilisateurs par attribution de token;

La population des entrées dans la base Keystone est faite généralement par l'administrateur du cloud Openstack

---
#Openstack
##Keystone

Keystone est accessible par API : 
- API admin : port 35357;
- API user : port 5000;

Keystone existe en deux versions d'API : v2 et v3. La v3 ajoute la gestion des domaines;

Keystone est composé de plusieurs sous parties :

<p style="text-align:center;"><img src="http://allthingsopendotcom.files.wordpress.com/2014/07/keystone.png" style="width: 500px;"/></p>

---
#Openstack
##Keystone - Identity

La notion de __tenant__ (aussi appelé __projet__), __user__, __role__ :
- un tenant ou projet, est une organisation à laquelle sont allouées des resources physiques du cloud (CPU/RAM/Disk...);
- chaque utilisateur appartient à un tenant;
- pour s'identifier sur un cloud openstack, un user utilisera le couple user;
- un utilisateur peut appartenir à plusieurs tenant;
- lorsqu'un utilisateur créé une resource (ex. une VM), il la créé pour le compte d'un tenant;
- chaque utilisateur à un rôle dans chacun de ses tenants (admin/member...);

---
#Openstack
##Keystone - Identity

```sh
$ openstack project list
+----------------------------------+--------------------+
| ID                               | Name               |
+----------------------------------+--------------------+
| 1484c557018b49a4acd6e4e331e3bc55 | service            |
| 3ae93431f89941bea16f92346471be11 | alt_demo           |
| 80d01fa581d041d9bfa1770a5ae09ad9 | admin              |
| cbf2d14974b64205824d121f5b38c525 | invisible_to_admin |
| de060cd2e96e4b1abdeb34b4cf1a121e | demo               |
+----------------------------------+--------------------+

$ openstack user list
+----------------------------------+----------+
| ID                               | Name     |
+----------------------------------+----------+
| c14f4133ab044f4ca002e795904383d3 | admin    |
| 48aa640562d14ffab7bdee26a90bd484 | demo     |
| 434109ca12b54477945f490fbc2ee4e3 | alt_demo |
| 3c4ebde0229e4f89af27a8d78ebf57c2 | nova     |
| aab6960638964c27a5500e644fafda01 | glance   |
| 08e6e73d9d3942df99ae8d290027652f | cinder   |
+----------------------------------+----------+
```
---
#Openstack
##Keystone - Policy
```sh
$ openstack role list
+----------------------------------+---------------+
| ID                               | Name          |
+----------------------------------+---------------+
| 76816e40d9d44200b85b98a9326fcb61 | admin         |
| 9961fe4db10846319f91054071926269 | Member        |
| d11de740d31b4e27aa1b1db81cac268c | service       |
+----------------------------------+---------------+
```
---
#Openstack
##Keystone - Policy

Le user "admin" à le role "admin" dans le projet "demo" et dans le projet "alt-demo"

```sh
$ openstack role list --user admin --project demo
+----------------------------------+-------+---------+-------+
| ID                               | Name  | Project | User  |
+----------------------------------+-------+---------+-------+
| 76816e40d9d44200b85b98a9326fcb61 | admin | demo    | admin |
+----------------------------------+-------+---------+-------+
$ openstack role list --user admin --project alt_demo
+----------------------------------+-------+----------+-------+
| ID                               | Name  | Project  | User  |
+----------------------------------+-------+----------+-------+
| 76816e40d9d44200b85b98a9326fcb61 | admin | alt_demo | admin |
+----------------------------------+-------+----------+-------+
```
---
#Openstack
##Keystone - Policy
Tandis que le user demo a le role member dans le projet demo
```sh
$ openstack role list --user demo --project demo
+----------------------------------+-------------+---------+------+
| ID                               | Name        | Project | User |
+----------------------------------+-------------+---------+------+
| 9961fe4db10846319f91054071926269 | Member      | demo    | demo |
+----------------------------------+-------------+---------+------+
```
---
#Openstack
##Keystone - Catalog
La notion __d'endpoint__ :
- chaque service Openstack est enregistré dans keystone;
- il enregistre son type;
- il enregistre ses URL;
    - PublicURL;
    - AdminURL;
    - InternalURL;

---
#Openstack
##Keystone - Catalog
```sh
$ openstack endpoint list
+----------------------------------+-----------+--------------+----------------+
| ID                               | Region    | Service Name | Service Type   |
+----------------------------------+-----------+--------------+----------------+
| bdb373a7c92546a08fcae97d5e64280d | RegionOne | glance       | image          |
| b057b3c9c007439cbe588c77e4b802d6 | RegionOne | cinderv2     | volumev2       |
| bc3a4293d2df41149936726049a99c2d | RegionOne | nova         | compute        |
| e00562c57f0b46f2a504adebe3276d0f | RegionOne | keystone     | identity       |
| 8db911c7eed2460c9250186176a6fc5b | RegionOne | nova_legacy  | compute_legacy |
| 13d006f2a5b74e24bbee04b38c2f9bc5 | RegionOne | cinder       | volume         |
+----------------------------------+-----------+--------------+----------------+
```
---
#Openstack
##Keystone - Catalog
```sh
$ openstack endpoint show keystone
+--------------+-----------------------------------+
| Field        | Value                             |
+--------------+-----------------------------------+
| adminurl     | http://192.168.122.237:35357/v2.0 |
| enabled      | True                              |
| id           | e00562c57f0b46f2a504adebe3276d0f  |
| internalurl  | http://192.168.122.237:5000/v2.0  |
| publicurl    | http://192.168.122.237:5000/v2.0  |
| region       | RegionOne                         |
| service_id   | 208c0d3e39f943ba8f5be8425daafb3c  |
| service_name | keystone                          |
| service_type | identity                          |
+--------------+-----------------------------------+
```
---
#Openstack
##Keystone - Token
La notion de token : 
- avant d'utiliser un endpoint, l'utilisateur s'authentifie via keystone et son username/password/tenant;
- keystone lui renvoie un token;
- l'utilisateur emploiera ce token pour s'authentifier sur les endpoints;
- chaque endpoint vérifiera la validité de ce token auprès de keystone avant de traiter la requete;
- keystone donnera également le rôle de l'utilisateur au endpoint;
- ainsi l'endpoint pourra vérifier si l'utilisateur : 
    - à le droit de demander la requete (role);
    - à le quota nécéssaire pour créer la resource;

---
#Openstack
##Keystone - Token
```sh
$ openstack token issue --os-username admin --os-project-name admin --os-auth-url http://localhost:5000/v2.0 
Password: 
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2016-04-01T15:41:53Z             |
| id         | aa521d73f8654911a181b964b01892f4 |
| project_id | 80d01fa581d041d9bfa1770a5ae09ad9 |
| user_id    | c14f4133ab044f4ca002e795904383d3 |
+------------+----------------------------------+
```
Je peux utiliser le token aa521d73f8654911a181b964b01892f4 pour faire des requetes sur les endpoint.

---
#Openstack
##Keystone - Résumé

<p style="text-align:center;"><img src="http://2.bp.blogspot.com/-9E7v6qpQr4Y/UyHiinS7XCI/AAAAAAAAAAM/XWbt2qvw1is/s1600/SCH_5002_V00_NUAC-Keystone.png " style="width: 700px;"/></p>

---
#Openstack
##Keystone - Backend

Les différentes partie de keystone peuvent s'appuyer sur différents backend :

<p style="text-align:center;"><img src="http://allthingsopendotcom.files.wordpress.com/2014/07/keystone.png" style="width: 700px;"/></p>

---
#Openstack
##Keystone - Fédération

Keystone peut être configuré pour utiliser un module d'identité externe (idP) : 
- keystone se décharge alors de ses parties :
    - création de user;
    - authentification de user;
    - perte de mot de passe;
- possibilité de partager un cloud entre plusieurs idPs;
- le user peut alors accèder à plusieurs services internet en Single Sign On;
- Algorithme :
    - le client contacte keystone;
    - keystone renvoie une liste d'idPs;
    - le client s'authentifie avec un idPs;
    - le client renvoie la réponse de l'idP à keystone;
    - keystone valide cette réponse et renvoie un token au user;
    - le user peut accèder aux services openstack avec ce token;

https://wiki.openstack.org/wiki/Keystone/Federation/Blueprint

---
#Openstack
##Keystone - API utilisateur

Les API de Keystone sont principalement utilent pour l'administrateur du cloud Openstack.

Les utilisateurs les utilisent utilisent de manière transparante pour 
- s'authentifier;
- obtenir un token;
- obtenir le catalogue de service;

Ainsi l'URL de keystone sera souvent utilisée comme __point d'entrée__ vers un cloud openstack : pour que l'utilisateur puisse utiliser un cloud Openstack, l'admin du cloud lui fournirra l'URL Keystone, un tenant et un user/password;

Par exemple, ce sont les seuls éléments nécéssait dans nova pour lister ses VMs :
```sh
$ nova --os-username demo --os-password labo --os-tenant-name demo --os-auth-url http://192.168.122.241:5000/ list
```

---
#Openstack
##Keystone - API utilisateur

Dans horizon, on pourra interagir avec keystone pour connaitre ses tenants :

<img src="./img/horizon_keystone.png" style="width: 700px;"/>
---
template: agenda

###.right[Openstack - Glance]
---
name: glance
#Openstack
##Glance

Avant de pouvoir déployer une VM, il faut choisir un __OS de base__, soit une __image de base__

Glance est une autre brique de base d'Openstack;

Ce service gère : 
- les images de base des VMs;
    -> une image est un disque virtuel bootable (avec un OS installé; ex : debian 8.0, centos, iso...)
- les propriétés sur les images;
- les quotas d'espace utilisable par tenant pour ces images;

Les demandes de démarrage de VMs dans Openstack passent par la mention de l'image de base dans Glance;

---
#Openstack
##Glance

Plusieurs proprités peuvent être affectées à une images :
- Type d'images;
- Architecture;
- Distribution;
- Version de la distribution;
- Espace dique minimum;
- RAM minimum;
- Publique ou privée;

```sh
$ glance image-list
+--------------------------------------+---------------------------------+
| ID                                   | Name                            |
+--------------------------------------+---------------------------------+
| ef6f18a3-886e-4eef-a65e-ddf5c6698d16 | cirros-0.3.4-x86_64-uec         |
| 191d6af5-4764-448a-a9c0-b0842e6fbf0b | cirros-0.3.4-x86_64-uec-kernel  |
| 86c0d488-757a-468d-afa8-553215e43144 | cirros-0.3.4-x86_64-uec-ramdisk |
+--------------------------------------+---------------------------------+
```
---
#Openstack
##Glance

```sh
$ glance image-show ef6f18a3-886e-4eef-a65e-ddf5c6698d16
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | eb9139e4942121f22bbc2afc0400b2a4     |
| container_format | ami                                  |
| created_at       | 2016-04-05T13:24:30Z                 |
| disk_format      | ami                                  |
| id               | ef6f18a3-886e-4eef-a65e-ddf5c6698d16 |
| kernel_id        | 191d6af5-4764-448a-a9c0-b0842e6fbf0b |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros-0.3.4-x86_64-uec              |
| owner            | df0c49848a864914b76788ff320eccbc     |
| protected        | False                                |
| ramdisk_id       | 86c0d488-757a-468d-afa8-553215e43144 |
| size             | 25165824                             |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2016-04-05T13:24:30Z                 |
| virtual_size     | None                                 |
| visibility       | public                               |
+------------------+--------------------------------------+
```

---
#Openstack
##Glance

Plusieurs format de disque sont utilisables : 
- raw
- vmdk, vhd (VMWare, Xen, Microsoft, VirtualBox....)
- vdi (VirtualBox, qemu)
- iso
- __qcow2__ (qemu, avec copy on write)
- aki, ari, ami (Amazon EC2)

On peut aussi utiliser des formats de Containers (disk+metadata) :
- bare (pas de conteneurs)
- ovf/ova (VMWare, Xen...)
- ami/ari/aki (Amazon EC2...)
- docker

---
#Openstack
##Glance

Glance écoute sur le port 9292, comme nous le dit le catalogue de service Keystone :

```sh
$ openstack endpoint show glance
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| adminurl     | http://192.168.122.237:9292      |
| enabled      | True                             |
| id           | bec00829127843e6888b958e15945035 |
| internalurl  | http://192.168.122.237:9292      |
| publicurl    | http://192.168.122.237:9292      |
| region       | RegionOne                        |
| service_id   | 7129a41efc974c998464bc196f96e839 |
| service_name | glance                           |
| service_type | image                            |
+--------------+----------------------------------+
```

---
#Openstack
##Glance

<p style="text-align:center;"><img src="http://4.bp.blogspot.com/-8BGR7XSvuSw/VEr6NqccUKI/AAAAAAAAAGc/PP4yLwUYzpI/s1600/glance.png" style="width: 300px;"/></p>

- glance API : 
    - expose l'API;
    - valide le token avec keystone;
    - traite les requètes relatives aux images;
- glance Registry : gère les metadata des images;
- glance database : contient les infos des images;
- glance backend : espaces de stockage des images;

---
#Openstack
##Glance - API utilisateur

Chaque utilisateur pourra gérer ses propres images via un ensemble d'API et de commandes CLI : 
```sh
    image-create        Create a new image.
    image-deactivate    Deactivate specified image.
    image-delete        Delete specified image.
    image-download      Download a specific image.
    image-list          List images you can access.
    image-reactivate    Reactivate specified image.
    image-show          Describe a specific image.
    image-tag-delete    Delete the tag associated with the given image.
    image-tag-update    Update an image with the given tag.
    image-update        Update an existing image.
    image-upload        Upload data for a specific image.
```

Souvent les cloud provider proposent des images publiques à leur tenants.

---
template: agenda

###.right[Openstack - Nova]
---
name: nova
#Openstack
##Nova

J'ai un identifiant sur le cloud, j'ai des images de base pour mes VMs, je peux désormais __déployer ma VM__!

Nova est le composant central d'Openstack.  Il est utilisé pour :
- gérér les VMs (flavor, availability zone);
- provisionner les VMs à leur démarrage (cloud-init);
- donner accès aux VMs depuis l'extérieur du cloud (security-group/floating-IPs);

---
#Openstack
##Nova - concept


---
#Openstack
##Horizon

---
name: cinder
#Openstack
##Cinder

---
name: neutron
#Openstack
##Neutron

---
#Openstack
##Heat



---
#Openstack
##Swift





