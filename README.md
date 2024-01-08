# Local Kube

L'objectif de ce repo est de servir de bac à sable Kubernetes.  
Il va permettre :

- de créer facilement un cluster local de dev ou s'exercer.
- de donner des exemples de code pour des cas d'usages classiques (service web, ingress, base de données, ...)
- de permettre de faire des tests de charge.
- d'avoir des configs d'exemple pour créer des cluster sur au moins scaleway et eput-être aws/azure/google.

## Pourquoi Kubernetes

Ce n'est pas que je sois fan des outils google généralement, mais Kubernetes est actuallement la référence pour gérer des containers. Chaque cloud provider a son alternative généralement basée sur kubernetes. Mais en utilisant kubernetes et pas, par exemple, `fargate`` sur AWS, on évite d'être bloqué sur un cloud en particulier.

Il y a aussi beaucoup d'autres outils soit pour un vrai cluster ([k0s](https://k0sproject.io/], [k3s](https://k3s.io/), ...) soit pour développer ([minikube](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/), [microk8s](https://microk8s.io/) mais Je trouve que c'est bien de commencer avec un vrai cluster histoire de ne pas être pris au dépourvu le jour ou on sera face à un problème.

## Installation du cluster

La doc d'installation [est ici](./install-k8s/README.md)

## Documentations

- [configurer un poste de travail](./desktop/README.md)
- [installer un plugin de stockage](./storage/README.md)
- [Gérer les utilisateurs](./users/README.md)
- Créer un job
- les bases de données
- créer un service elasticsearch
- Créer un ingress pour exposer plusieurs services sur des hosts différents
- Dédier un node à un service
- Créer un plugin
