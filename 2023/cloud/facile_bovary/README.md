# Harpagon

## Challenge

```
Un individu dans un coin vous interpelle et vous invite à sa table. Une fois assis, il vous explique 
qu'il veut que vous infiltriez le cluster Kubernetes de Madame Bovary. Madame Bovary est une femme riche 
et influente qui a investi dans la technologie Kubernetes pour gérer les applications de son entreprise 
de production de médicaments. Vous vous doutez qu'il s'agit sans doute d'un concurrent industriel mais il
vous offre une belle récompense si vous réalisez sa demande.

Votre mission consiste à prendre le contrôle du cluster Kubernetes de Madame Bovary et à accéder à ses 
applications critiques. Vous devrez exploiter toutes les vulnérabilités possibles pour atteindre votre 
objectif.
```

```
Le fichier fourni est une machine virtuelle à charger dans Virtualbox. Cette machine virtuelle contient 
le cluster Kubernetes du challenge.

Utilisateur : ctf (404ctf pour la vm)
Mot de passe : 404ctf2023
```

## Solution

Il y a un cluster Kubernetes sur la machine:

```sh
kubectl get pods -A
```

Show the agent logs :

```sh
kubectl logs agent
> deploy 404ctf/the-container
```

Create the pod :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: the-container
  # THE LOGS WILL TELL YOU: deploy it in '404ctf' namespace
  namespace: 404ctf
spec:
  containers:
  - image: 404ctf/the-container
    name: the-container
    imagePullPolicy: IfNotPresent
    # THE LOGS WILL TELL YOU: '/opt/my_secret_dir/' must be mounted on host
    volumeMounts:
    - mountPath: /opt/my_secret_dir/
      name: host-volume
  # THE LOGS WILL TELL YOU: set SUPER_ENV var to 'SECRET'
    env:
    - name: SUPER_ENV
      value: "SECRET"
  restartPolicy: Never
  volumes:
  - name: host-volume
    hostPath:
      path: /home/ctf/flag
```

Show `the-container` logs :

```sh
kubectl logs -n 404ctf the-container
> Flag written to /opt/my_secret_dir/flag.txt
```

Print the flag content ;

```sh
kubectl exec -it -n 404ctf the-container -- cat /opt/my_secret_dir/flag.txt
> 404CTF{A_la_decouv
> Le reste du flag est dans le conteneur 404ctf/web-server
```

Create the pod :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
spec:
  containers:
  - image: 404ctf/web-server
    name: web-server
    imagePullPolicy: IfNotPresent
```

Log the pod :

```sh
kubectl logs web-server
> Starting server at port 8080
```

Forward the port :

```sh
kubectl port-forward pod/web-server 8080:8080
```

Request it :

```sh
curl -L http://localhost:8080
> Le drapeau est dans /flag
curl -L http://localhost:8080/flag
> erte_de_k8s}
```

**DONC LE FLAG EST** `404CTF{A_la_decouverte_de_k8s}`
