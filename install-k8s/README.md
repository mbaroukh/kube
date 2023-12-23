# Créer un cluster local

Il est très facile de se créer un cluster de dev avec minikube ou k3s par exemple.  
Mais je trouve ça pas mal de s'exercer sur un cluster plus proche de ce que serait un vrai cluster de prod.

Donc là, l'objectif c'est de créer un cluster avec un `control-plan` et au moins 1 noeud.

> **_NOTE:_**
> Sur les cluster kubernetes il y a parfois 2 interfaces : 1 interface pour exposer les services et une pour la communication entre les noeuds. Là, je vais considérer que les VMs sont sur un réseau privé et qu'il n'est pas utile d'avoir des 2 interfaces.

## Prérequis

Le minimum nécessaire, c'est :

- Un gestionnaire de machines virtuelles. Ici je vais utiliser [VirtualBox](https://www.virtualbox.org/) mais libre à vous d'utiliser autre chose si vous êtes plus à l'aise avec (QEmu, VMWare, ...).
- `kubectl`. Permettra de contrôler le cluster. Personnellement je l'ai installé via [asdf](https://asdf-vm.com/).
- `ansible`. Sur mon poste j'ai juste installé la package `ansible` via apt.

### VM Source

Pour éviter d'installer X fois des machines, on va installer une VM avec tous les packages requis juste avant de lancer kubeadm.  
Puis on dupliquera ces machines d'abord pour faire le control-plan et ensuite autant de fois que nécessaire pour créer le nombre de nodes souhaités.

#### Installation de la VM

Je ne vais pas détailler le process.
L'objectif à la fin c'est d'avoir une VM opérationnelle.
Quelques infos quand même :

- Je vous laisse choisir les ressources mais Kubernetes ne s'installera pas avec moins de 2 cpu. Et moins de 4Go suffiraient pour l'installation mais deviendraient peut-être compliqués par la suite.
- Côté disque, j'ai mis 40Go non préalloué. L'image finale occupe environ 6Go par noeud.
- La carte doit être en mode `bridge` sinon les VMs ne pourront pas communiquer.
- Je vous conseil d'utiliser La debian stable du moment (12 à ce jour) pour être au plus compatibel avec la suite.
- Pour faire simple, j'ai laissé l'install tout configurer dans une seule partition mais avec LVM quand même au cas ou.
- J'ai mis le clavier en français bien sur mais j'ai laissé le système en en-US.
- Au moment de la sélection des packages, pas de desktop. Juste `SSH Server` et `Standard system utilities`
- J'ai appelé cette vm `k8s-src` donc si vous donnez le même nom vosu pourrez copier/coller les commandes suivantes.
- j'ai créé un utilisateur `k8s`. Si vous l'appelez différemment, il faudra adapter le [playbook](./k8s.common.yml).

Pour continuer vérifiez les points suivants :

- vous arrivez à vous connecter au compte k8s sur la machine sans taper votre mot de passe.
- une fois connecté, vous arrivez a exécuter la commande `sudo su -` sans avoir à entrer de mot de passe.

Si c'est le cas, parfait.  
Sinon, je ne vais pas tout détailler non plus mais en gros vous devez, sur la vm :

1. Mettre votre clé publique dans `.ssh/authorized` du compte
   ```bash
   $ mkdir .ssh
   $ echo "ssh-ed25519 AAAAC3Nza...wnuSQuWw8c mike@..." >.ssh/authorized_keys
   $ chmod -R go-rwX .ssh/
   ```
2. Installer `sudo`
   ````bash
   $ su -
   $ Password:
   $ root@k8s-src:~# apt install sudo
   ```bash
   ````
3. Enfin, autoriser l'utilisateur k8s a devenir root sans mot de passe.
   On aurait pu utiliser le groupe sudo mais ça oblige à changer la conf par défaut.
   Je préfère cette méthode :
   ```bash
   $ echo "k8s   ALL=(ALL:ALL) NOPASSWD:ALL" >/etc/sudoers.d/k8s
   ```
   Après ça on fait l'install et si on veut revenir à la conf par défaut, il suffit de supprimer le fichier `/etc/sudoers.d/k8s`.
   Mais vu que c'est un cluster de dev, ce ne sera pas nécessaire.

### Playbook commun

Pour l'installation des noeuds, les systèmes doivent être configurés un peu avant. En particulier :

- installation du repo kubernetes
- installation d'un gestionnaire de containers. Là c'est `containerd`.
- configuration de `containerd` pour que les [control groups](https://fr.wikipedia.org/wiki/Cgroups) soient gérés par systemd. C'est ainsi que kubernetes pourra contraindre les pods en cpu/memoire/réseau/...
- ne pas avoir de swap. Sinon l'initialisation avec kubeadm va fonctionner, mais le cluster ne démarrera pas.

C'est ce que fait le playbook [k8s.common.yml](./k8s.common.yml).  
Il s'occupe aussi :

- de désactiver ipv6, pour s'éviter des problème. On est sur un réseau interne donc ce sera plus simple en ipv4.
- installer les packages `kubeadm` et `kubectl`
- télécharger les images dont aura besoin k8s pour configurer le cluster. Pour éviter de les re-télécharger pour chaque noeud.
- installer `jq` qui sera bien pratique pour travailler avec kubernetes.
- installer `rsync ` qui servira à transférer des fichiers entre notre desktop et les VMs.

Pour l'exécuter, vous avez juste besoin de l'ip de la vm que vous venez de configurer :

```bash
$ ansible-playbook -i 192.168.68.101, k8s.common.yml

PLAY [installation des parties communes] ***************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************************************************************************************************************
ok: [k8s-src]
...
k8s-src             : ok=19   changed=16   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

> **Notez** la virgule après l'ip dans la commande précédente. Ca permet de faire comprendre à ansible que c'est le contenu d'un
> inventaire et non le chemin vers un fichier qui contient l'inventaire.

That's it. Si tout se passe bien, on peut passer à l'installation du controlplan.

### Control plan

On va installer la machine `control-plan` en se basant sur l'image `k8s-src`.

Avec Virtualbox vous pouvez utiliser l'interface, ou vous pouvez utiliser la commande clonevm :

```bash
VBoxManage clonevm  "k8s-src" --basefolder=/home/mike/vms/k8s/ --name k8s-controlplan --register
```

(l'option `basefolder` est facultative, mais j'aime bien mettre toutes les images au même endroit).

si vous utilisez l'interface, faites bien attention à ce qu'il crée une nouvelle adresse mac pour l'interface réseau.  
C'est le cas par défaut en ligne de commande.  
(et tant mieux parce que pas envie de chercher comment le paramètrer)

Après ça, vous avez une machine que vous pouvez démarrer, éventuellement en mode headless :

```bash
$ VBoxManage startvm k8s-controlplan --type headless
Waiting for VM "k8s-controlplan" to power on...
VM "k8s-controlplan" has been successfully started.
```

Le seul intérêt de démarrer en mode non headless aurait été de récupérer son ip, mais vous savez que votre dhcp va faire +1 sur l'ip de `k8s-src`. Donc dans mon cas ce sera `192.168.68.99`.

La seule config à faire sur le serveur est changer son nom.  
Je n'ai pas fait de playbook pour ça.
Il suffit de se connecter à la machine et d'exécuter :

```bash
echo "k8s-controlplan"|sudo tee /etc/hostname
sudo sed -i "s/k8s-src/k8s-controlplan/g" /etc/hosts
```

Il y aura une erreur de sudo mais pas grave.

Toutes les VMs auront les même clés de serveur ssh. Ce n'est pas idéal mais ça ira. Si vous voulez les regénérer :

```
rm /etc/ssh/ssh_host_*
dpkg-reconfigure openssh-server
```

Ensuite le mieux est de rebooter (pas nécessaire, mais histoire de).

On va maintenant utiliser `kubeadm` pour configurer le cluster.
kubeadm sera utilisé aussi bien pour configurer le control-plan que pour configurer les nodes.

Pour configurer le control-plan :

```bash
sudo kubeadm init --pod-network-cidr 192.168.71.1/24
```

Le paramètre `pod-network-cidr` est fonction de votre réseau.  
Il sert à indiquer à kubernetes sur quelle plage d'adresses il peut créer des services.  
Ce sera utilisé par le le CNI (Container Network Interface) qu'on installera par la suite.

Voilà comment j'ai fait pour déterminer sa valeur chez moi :  
Ma machine sur le wifi a actuellement l'ip `192.168.68.76/22`. On pourrait calculer le détail du réseau,, mais il y a des outils en ligne donc autant les utiliser. En mettant l'ip sur [IP Calculator](https://jodies.de/ipcalc) on voit que ce reseau a 1022 ips allant de `192.168.68.1` a `192.168.71.254`. La plage `192.168.68.1/24` étant actuellement utilisée par mon dhcp, je vais dire que mon cluster sera sur `192.168.71.1/24` soit 254 adresses entre `192.168.71.1` et `192.168.71.254`.
J'aurais un problème lorsque j'aurais ajouté environ 400 nouvelles machines sur mon réseau, mais on verra bien à ce moment 🙂.

Finalement, si tout se passe bien avec `kubeadm` vous devriez avoir une sortie qui ressemble à ça :

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.68.104:6443 --token xbr0o0.51l6sf2dajz1ni04 \
        --discovery-token-ca-cert-hash sha256:8aca0cc2bd45d3e460f97888a1bdc1e4c5000c64b21498be6b167ddd7769914f
```

Exécutez tout de suite les commandes qui permettront à votre utilisateur d'avoir un kubectl correctement configuré :

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Après ça, vous devriez pouvoir lancer des commandes kubernetes :

```bash
$ kubectl get node
NAME              STATUS     ROLES           AGE     VERSION
k8s-controlplan   NotReady   control-plane   3m21s   v1.28.5
```

Le status `NotReady`, c'est par ce qu'il reste à installer un CNI pour gérer le réseau.

### Installation du CNI

Le CNI s'occupe de gérer la partie réseau du cluster.  
C'est lui qui va attribuer des ips aux services, configurer les routes, ...

Le plus connu c'est calico :
https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises

L'installation est très simple puisque le CNI fonctionne lui même comme un `daemon-set` sur le cluster.

Il suffit donc de télécharger le fichier de définition puis l'installer avec kubectl :

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

Au bout de quelques secondes, le cluster devrait passer en `Ready` :

```bash
$ kubectl get node
NAME              STATUS   ROLES           AGE   VERSION
k8s-controlplan   Ready    control-plane   96s   v1.28.5
```

### Node 1

L'installation du premier node est une formalité à présent.

On repête les opération de clone de vm et de configuration effectutuée pour le controlplan.

```bash
$ VBoxManage clonevm  "k8s-src" --basefolder=/home/mike/vms/k8s/ --name k8s-node1 --register
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
Machine has been successfully cloned as "k8s-node1"

$ VBoxManage startvm k8s-node1 --type headless
Waiting for VM "k8s-node1" to power on...
VM "k8s-node1" has been successfully started.

$ echo "k8s-node1"|sudo tee /etc/hostname
$ sudo sed -i "s/k8s-src/k8s-node1/g" /etc/hosts

$ sudo su -
sudo: unable to resolve host k8s-src: No address associated with hostname
$ rm /etc/ssh/ssh_host_*
$ dpkg-reconfigure openssh-server
Creating SSH2 RSA key; this may take some time ...
3072 SHA256:JdTJcnGJcSbtDFopu+pRAptnEazuCAsewzxZs1ofzfI root@k8s-src (RSA)
...
$ sudo reboot
```

Une fois le node redémarré, on peut utiliser la commande que nous a fournie la sortie de kubeadm sur le control-plan pour configurer le node :

```bash
$ sudo kubeadm join 192.168.68.104:6443 --token xbr0o0.51l6sf2dajz1ni04 \
        --discovery-token-ca-cert-hash sha256:8aca0cc2bd45d3e460f97888a1bdc1e4c5000c64b21498be6b167ddd7769914f
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
...
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

C'est fini. Sur le control-plan on voir bien le nouveau node :

```bash
$ kubectl get node
NAME              STATUS   ROLES           AGE   VERSION
k8s-controlplan   Ready    control-plane   24m   v1.28.5
k8s-node1         Ready    <none>          18s   v1.28.5
```

On peut répeter l'opération autant de fois que nécessaire pour ajouter des nodes.
