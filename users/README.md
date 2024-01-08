# Users

Kubernetes permet de gérer finement qui peut accèder à quoi.  
Par défaut, une config est générée sur le control-plan qui permet d'être administrateur sur le cluster.

Mais le plus propre c'est sans doute de de créer de vrais utilisateurs et de leur affecter des droits.

## Creer un compte utilisateur nominatif avec des droits administrateur

Le principe est le suivant :

- un utilisateur se crée une clé privée, comme pour des accès ssh.
- il se créer une CSR, comme on le fait pour créer un certificat SSL.
- il envoi sa CSR à un administrateur qui va la créer sur le cluster
- La csr va être approuvée ce qui crée le nouvel utilisateur. Apartir de là l'utilisateur est reconnu mais n'a encore aucun droit.
- Finalement, on affecte à l'utilisateur les droits dont il a besoin.

### Création d'une clé privée

Cette clé privée n'a pas a être créée à chaque fois. C'est comme une clé ssh on on s'en fait une et on la réutilise sur les différents clusters. Pour la créer :

```bash
openssl genpkey -out ~/.kube/mike.key -algorithm ed25519
```

### CSR

Pareil pour la CSR : on peut utiliser la même tant qu'on est dans la même organisation :

```bash
openssl req -new -key ~/.kube/mike.key -out ~/.kube/mike.csr -subj "/CN=mike,/O=....com"
```

Le csr est à transmettre à l'administrateur du réseau.

### CSR Cluster

A partir de la csr de l'utilisateur, on va créer un CSR sur le cluster auquel l'utilisateur souhaite avoir accès.  
Pour cela, on utilise ce yaml :

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: mike
spec:
  request: LS0tLS1C...TkQgQ0VSVElGSUNBVEUgUkVRVUVTVC0tLS0tCg==
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 315360000
  usages:
    - client auth
```

la champ `request` est la représentation en `base64` de la `csr` sur une seule ligne. Pour l'obtenir :

```bash
base64 -w 0 ~/.kube/mike.csr
```

### Validation de la CSR

Une fois le csr créé sur le cluster, il doit être approuvé.  
Son rôle est actuellement `pending` :

```bash
$ k get certificatesigningrequest
NAME   AGE     SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
mike   3m16s   kubernetes.io/kube-apiserver-client   kubernetes-admin   10y                 Pending
```

Pour l'approuver :

```bash
$ k certificate approve mike
certificatesigningrequest.certificates.k8s.io/mike approved
$ k get certificatesigningrequest
NAME   AGE     SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
mike   5m12s   kubernetes.io/kube-apiserver-client   kubernetes-admin   10y                 Approved,Issued
```

### Génération du certificat

Un certificat lié à la clé privée de l'utilisateur foit ensuite être généré par l'administarteur du cluster et envoyé à l'utilisateur. Là, on est utilisateur et administrateur, on peut aller plus vite :

```bash
kubectl get csr/mike -o jsonpath="{.status.certificate}" | base64 -d > ~/.kube/mike.local.crt
```

### donner des droits à l'utilisateur

Kubernetes fonctionne par roles qu'on atribue (bind) aux utilisateurs.

### Pouvoir tout faire sur un namespace

Il peut être pratique sur un cluster de créer un `namespace` pour un projet et de laisser l'équipe avoir le champ libre dessus.

Pour ça, il fait d'abord créer un rôle dans le namespace qui a tous les droits :

```bash
k create role admin-projet1 --namespace projet1 --verb='*' --resource='*' --apiGroups='*'
k create rolebinding admin-projet1-mike --role=admin-projet1 --user=mike
```

#### Donner le role administrateur

Un role `cluster-admin` existe déjà et peut être affecté à un utilisateur pour lui donner tous les droits.  
Pour celà :

```
kubectl create clusterrolebinding mike-admin --clusterrole=cluster-admin --user=mike
```

### Ajouter le certificat

kubectl config set-credentials mike --client-key ~/.kube/mike.key --client-certificate ~/.kube/mike.local.crt --embed-certs=true
