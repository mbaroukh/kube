# Local Kube

L'objectif de ce repo est de servir de bac à sable Kubernetes.  
Il va permettre :

- de créer facilement un cluster local de dev ou s'exercer.
- de donner des exemples de code pour des cas d'usages classiques (service web, ingress, base de données, ...)
- de permettre de faire des tests de charge.
- d'avoir des configs d'exemple pour créer des cluster sur au moins scaleway et eput-être aws/azure/google.

## Pourquoi Kubernetes

Ce n'est pas que je sois fan des outils google généralement, mais Kubernetes est actuallement la référence pour gérer des containers. Chaque cloud provider a son alternative généralement basée sur kubernetes. Mais en utilisant kubernetes et pas, par exemple, `fargate`` sur AWS, on évite d'être bloqué sur un cloud en particulier.

Il y a aussi beaucoup d'autres outils (comme [Rancher](https://www.rancher.com/) par exemple) qui simplifie la gestion d'un cluster. Mais Je trouve que c'est bien de commencer avec un vrai cluster histoire de ne pas être pris au dépourvu le jour ou on sera face à un problème.

## Installation du cluster

La doc d'installation [est ici](./install-k8s/README.md)

## Documentations à venir

- configurer l'accès au cluster depuis un poste de travail
- créer un utilisateur avec des droits restreints
- les bases de données
- créer un service elasticsearch
- Créer un ingress pour exposer plusieurs services sur des hosts différents
- installer un plugin de stockage
- Dédier un node à un service
