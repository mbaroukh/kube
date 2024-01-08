# Configurer son poste de travail

## Accèder au cluster à distance

### Installation kubectl

La première chose dont vous allez avoir besoin, c'est de `kubectl`.

Pour l'installer, personnellent, j'utilise [asdf](https://asdf-vm.com/).  
J'utilise asdf pour beaucoup de choses : node, java, yarn, terraform, bun, aws-cli, ...
Ca me permet d'installer ces outils dans ma home pour mon utilisateur et non gloabalement sur le tout le système.  
Et ça me permet, par projet, de choisir quelle version utiliser.  
asdf a un système de plugins bien penseé qui fait qu'on trouve des [plugins pour à peu près tout](https://github.com/asdf-vm/asdf-plugins/tree/master/plugins).

Après avoir suivi l'installation que se fait en 1 minute montre en main, pour installer kubectl il suffit d'ajouter le plugin, installer la derni-re version et de la sélectionner par défaut :

```bash
asdf plugin add kubectl
asdf install kubectl latest
asdf global kubectl latest
```

C'est tout. A partir de là, vous avez la commande `kubectl` disponible.

### Copie de la config

Pour accèder au cluster à distance, vous avez besoin d'avoir la configuration Kubernetes.  
Cette configuration des dans `~/.kube/config` par défaut.

> **ATTENTION**: Si vous avez déjà une configuration d'un autre cluster sur votre poste,
> elle sera écrasée. Faites des backups avant.

Pour copier la config :

```bash
mkdir ~/.kube
rsync -av k8s@192.168.68.104:.kube/config ~/.kube/config
```

C'est la façon la plus simple et qui est amplement suffisant dans le cas d'un cluster de dev.

Cependant, si vous êtes sur un cluster réellement utilisé, le plus propre c'éest de créer de vrais utilisateurs et de leur affecter des droits.

## Alias utiles

J'utilise 2 alias :

- k : pour éviter de taper "kubectl"
- kns : pour pouvoir changer de namespace
- kc: pour pouvoir changer de configuration / cluster

### k

C'est juste un alias :

```bash
alias k=kubectl
```

Ce qui permet de taper juste `k get pods` au lieu de `kubectl get pods`

### kns

Pour changer de namespace. L'alias est :

```bash
alias kns="kubectl config set-context --current --namespace"
```

### kc / kcg

Ce denier alias est en fait une fonction.  
On peut normalement gérer tous les cluster dans une même conf, mais ça oblige à les merger.  
Je préfères définir ces fonctions :

```bash
function kc() { export KUBECONFIG=~/.kube/config.$1; }
function kcg() { ln -sf ~/.kube/config.$1 ~/.kube/config; }
```

Por les utiliser, je mets un fichier de conf par cluster dans `~/.kube` : `config.local`, `config.staging`, ...

Ensuite, en fonction de si je vais travailler longtemps surun cluster je le défini par défault gloablement avec `kcg local` ou juste pour ma session en cours avec `kg local`.

### Recap

Au final, je mets ça dans mon `~/.bashrc` :

```bash
alias k=kubectl
alias kns="kubectl config set-context --current --namespace"
function kc() { export KUBECONFIG=~/.kube/config.$1; }
function kcg() { ln -sf ~/.kube/config.$1 ~/.kube/config; }
```
