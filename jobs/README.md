# Créer un job

Un job est une tâche lancée ponctuellement ou périodiquement qui exécute une action et quitte.

### Action ponctuelle

Démarrer un job qui va écrire `hello k8s` et quitter au bout de 2 secondes :

```bash
k create job hellok8s --image=busybox:latest -- /bin/sh -c 'echo "hello k8s"; sleep 2;'
```

Malheureusement, par défaut, Kubernetes ne va pas supprimer le pod après l'exécution.  
Pour cela il faut lui passer un argument supplémentaire : [ttlSecondsAfterFinished](https://kubernetes.io/docs/concepts/workloads/controllers/job/#ttl-mechanism-for-finished-jobs). Et cet argument ne peut être défini en ligne de commande.

On doit donc passer par un yaml :

```bash
k create job hellok8s --image=busybox:latest --dry-run=client -o yaml -- /bin/sh -c 'echo "hello k8s"; sleep 2;'
```

Ce qui permet ensuite d'ajouter l'argument et de créer le job avec `apply`

### Action périodique
