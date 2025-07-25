---
title: "Cours - Introduction, IAC et Devops"
description: "Guide Cours - Introduction, IAC et Devops"
sidebar:
  order: 100
---


![](/100_cours_intro/images/terraform-logo.png)


## quelques aspets pratiques:

- Le lab : se connecter à https://guacamole.formation.dopl.uk/guacamole
  - Login : votre prenom en minuscule sans accents
  - Mot de passe : fournis par le formateur 

- Supports : ce site qui restera en ligne pendant les prochaines années ( et imprimable en pdf joliment grâce à la fonction d'impression de chrome/chromium)


## La formation 

Jour 1 : Language de déploiement, dans le cloud et au-delà  
Jour 2 : Architecture Terraform et bonnes pratiques 


## Devops et Terraform 


#### Le mouvement DevOps

- Dépasser l'opposition culturelle et de métier entre les développeurs et les administrateurs système.
- Intégrer tout le monde dans une seule équipe et ...
- Calquer les rythmes de travail sur l'organisation agile du développement logiciel
- Rapprocher techniquement la gestion de l'infrastructure du développement avec l'infrastructure as code.
  - Concrètement on écrit des fichiers de code pour gérer les éléments d'infra
  - l'état de l'infrastructure est plus claire et documentée par le code
  - la complexité est plus gérable car tout est déclaré et modifiable au fur et à mesure de façon centralisée
  - l'usage de git et des branches/tags pour la gestion de l'évolution d'infrastructure

--- 

#### Objectifs du DevOps

- Rapidité (celerité) de déploiement logiciel (organisation agile du développement et livraison jusqu'à plusieurs fois par jour)
  - Implique l'automatisation du déploiement et ce qu'on appelle la CI/CD c'est à dire une infrastructure de déploiement continu à partir de code.
- Passage à l'échelle (horizontal scaling) des logiciels et des équipes de développement (nécessaire pour les entreprises du cloud qui doivent servir pleins d'utilisateurs)
- Meilleure organisation des équipes
  - meilleure compréhension globale du logiciel et de son installation de production car le savoir est mieux partagé
  - organisation des équipes par thématique métier plutôt que par spécialité technique (l'équipe scale mieux)

--- 

#### Apports de Terraform pour le DevOps

- **Automatisation de l'infrastructure**   
  Terraform permet de définir l'infrastructure comme du code, ce qui facilite l'automatisation et l'orchestration de la configuration, du déploiement et de la gestion de l'infrastructure.
- **Gestion de l'état**    
  Terraform stocke l'état de l'infrastructure dans un fichier, ce qui permet de suivre l'évolution de l'infrastructure et de la gérer efficacement.
- **Gestion multi-cloud**    
  Terraform prend en charge de nombreux fournisseurs de cloud (AWS, GCP, Azure, etc.), ce qui facilite la gestion d'infrastructures multi-cloud.
- **Collaboration**    
  Terraform permet à plusieurs membres de l'équipe de travailler sur le même code d'infrastructure, ce qui facilite la collaboration et la gestion des modifications.
- **Versioning**    
  Terraform permet de versionner le code d'infrastructure, ce qui facilite la gestion des modifications et des versions de l'infrastructure.


--- 

## IAC et Terraform


#### Infrastructure as Code

**L'infrastructure as code (IaC) est une pratique consistant à décrire l'infrastructure informatique comme du code source, à l'aide d'un langage de programmation.** 

L'objectif est de définir l'infrastructure nécessaire à l'exécution du logiciel d'une manière 
* documentée
* historisée
* mutualisée 

Cela signifie que l'infrastructure (serveurs, réseaux, stockage, etc.) n'est plus gérée et provisionnée plutôt par des processus manuels.

Ce qui permet d'automatiser les changements d'infrastructure, comme par exemple lancer une infrastructure pour des tests.

---

**L'infrastructure as code est une pratique qui permet de gérer l'infrastructure informatique de manière plus automatisée, reproductible, collaborative, agile et rentable.**

Les avantages de l'infrastructure as code sont nombreux :

 **Automatisation**  
  L'infrastructure as code permet d'automatiser la configuration, le déploiement et la gestion de l'infrastructure, ce qui permet de réduire les erreurs humaines et de gagner du temps.

 **Reproductibilité**  
  Les scripts d'infrastructure as code peuvent être versionnés, ce qui permet de reproduire facilement des environnements de développement, de test ou de production.

 **Collaboration**  
  Les scripts d'infrastructure as code peuvent être partagés, modifiés et testés par plusieurs membres de l'équipe, ce qui facilite la collaboration et la gestion des changements.

 **Agilité**  
  L'infrastructure as code permet de provisionner et de déprovisionner rapidement des ressources, ce qui facilite l'adaptation aux changements et aux besoins de l'entreprise.

**Coût**   
  L'infrastructure as code permet de réduire les coûts de maintenance et de gestion de l'infrastructure, car elle permet de minimiser le temps et les ressources nécessaires pour gérer l'infrastructure.

--- 

#### Avantages de Terraform comme solution d'IAC 

**Terraform offre des avantages qui en fait une solution d'IaC populaire pour les équipes DevOps.**

- **Base d'utilisateur**   
  Terraform a acquis une maturité qui le place comme l'un des acteurs essentiels dans son domaine : le pilotage des API de fournisseurs de services cloud au sens large.  

- **Planification**   
  La fonction clef de Terraform est sa capacité à reconnaître les parties qui doivent être déployées avant les autres et également ne redéployer que ce qui est nécessaire.   

- **Modularité**   
  Terraform permet de définir des modules d'infrastructure réutilisables, ce qui facilite la création et la maintenance d'infrastructures complexes.

- **Adaptabilité**   
  Terraform permet de définir une infrastructure en apportant la "glue" nécessaire pour interconnecter de manière originale des solutions existantes en branchant ensemble différents modules.


--- 

## Comparaison entre Terraform et d'autres solutions IAC 

**Il existe 5 grandes familles d'outils d'IAC.**

--- 

### Scripts ad hoc  
  C'est la façon la plus basique de faire, en mettant dans des scripts les opérations répétables.  
  Ex: Scripts Bash  
```bash
## Update the apt-get cache
sudo apt-get update

## Install PHP and Apache
sudo apt-get install -y php apache2

## Copy the code from the repository
sudo git clone https://github.com/brikis98/php-app.git /var/www/html/app

## Start Apache
sudo service apache2 start

```

--- 

### Outils de gestion de configuration  
  On utilise des outils de gestion de configuration, ce qui signifie qu'ils sont conçus pour installer et gérer des logiciels sur des serveurs existants.
  Avantages : conventions de code, idempotence, nombreuses cibles 
  ex: Chef, Puppet et Ansible

```yaml
- name: Update the apt-get cache
  apt:
    update_cache: yes

- name: Install PHP
  apt:
    name: php

- name: Install Apache
  apt:
    name: apache2

- name: Copy the code from the repository
  git: repo=https://github.com/brikis98/php-app.git dest=/var/www/html/app

- name: Start Apache
  service: name=apache2 state=started enabled=yes
```
  
--- 

### Outils de modèles de serveur  

  Les outils de création de modèles de serveur sont devenus populaire.  
  Au lieu de lancer un tas de serveurs et de les configurer en exécutant le même code sur chacun d'eux, on crée une image autonome avec le logiciel, les fichiers et tous les autres détails pertinents.
  Un autre outil IaC déploie cette image sur les serveurs, qu'il s'agisse de VMs ou de conteneurs.
  Ex: Docker, Packer

```json
{
  "builders": [{
    "ami_name": "packer-example-",
    "instance_type": "t2.micro",
    "region": "us-east-2",
    "type": "amazon-ebs",
    "source_ami": "ami-0fb653ca2d3203ac1",
    "ssh_username": "ubuntu"
  }],
  "provisioners": [{
    "type": "shell",
    "inline": [
      "sudo apt-get update",
      "sudo apt-get install -y php apache2",
      "sudo git clone https://github.com/brikis98/php-app.git /var/www/html/app"
    ],
    "environment_vars": [
      "DEBIAN_FRONTEND=noninteractive"
    ],
    "pause_before": "60s"
  }]
}

--- 

```

### Outils d'orchestration  

  Les outils d'orchestration se basent sur des modèles de serveur pour assurer la gestion du cycle de vie des services pilotés par les équipe Devops.
  L'orchestrateur va piloter ces instances en : démarrage / arrêt, configuration, démultiplication à la demande, et autres opérations nécessaires à la bonne marche du service. 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
spec:
 selector:
    matchLabels:
      app: example-app
  replicas: 3
  strategy:
    rollingUpdate:
      maxSurge: 3
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: example-app
    spec:
      containers:
        - name: example-app
          image: httpd:2.4.39
          ports:
            - containerPort: 80
```

--- 

### Outils de provisionnement  
  Provisioning tools don't define the code that runs on each server, but create server, databases, caches, load balancers, queues, monitoring, subnet configurations, firewall settings, routing rules, Secure Sockets Layer (SSL) certificates, and almost every other aspect of your infrastructure.   
  Ex: Terraform, CloudFormation, OpenStack Heat, and Pulumi

```coffee
resource "aws_instance" "app" {
  instance_type     = "t2.micro"
  availability_zone = "us-east-2a"
  ami               = "ami-0fb653ca2d3203ac1"

  user_data = <<-EOF
              #!/bin/bash
              sudo service apache2 start
              EOF
}
```


