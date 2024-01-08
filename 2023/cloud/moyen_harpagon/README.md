# Harpagon

## Challenge

```
La Flèche

Brisant le quatrième mur

Hé quoi ! Ce coquin d'Harpagon ne se lassera donc jamais d’importuner les jeunes gens ! Voilà donc 
maintenant qu'il va jusqu'à louer des serveurs pour mettre ses écus à l’abri, alors même qu'il refuse à 
ses propres enfants la moindre dot. Son fils, mon maître, est désespéré de ne pouvoir prétendre à sa 
bien-aimée Mariane, que son père tente de lui ravir. Sa fille Élise n'est pas mieux traitée, la voilà 
promise à un ancêtre. Ha non vraiment, je ne puis me résoudre à les abandonner ! Sachez que j'ai donc 
mené mon enquête, et je pense avoir trouvé de quoi déstabiliser le coquin. Je ne crois pas qu'il ait 
jamais réussi à faire marcher sa machine comme il le souhaitait, mais elle dois néanmoins contenir toutes
sortes de choses qui pourraient faire avancer notre affaire. Accepteriez-vous de m'apporter votre 
concours pour dévoiler les secrets du vieil avare afin que nos chers amis puissent plus aisément le 
raisonner ? 
```

```
Connectez vous au VPS d'Harpagon, investiguez ce qu'il y a fait et retrouvez son secret.
Attention, les services peuvent mettre un peu de temps à démarrer.
Harpagon n'est pas très doué et n'a jamais réussi à utiliser sa cassette.

ssh 404ctf@challenges.404ctf.fr -p 31284
Mot de passe : T8h2UKEstg 
```

## Solution

Afficher l'historique avec `history` -> Il y a eu :
1. une installation helm
2. un upgrade helm

Rollback l'installation :

```sh
helm rollback cassette 
```

Le token admin se trouve dans la configmap (ou aussi le secret) du déploiement cassette.

Décoder en base64 le token.
Reverse la string (c'est écrit à l'envers).
Le flag est trouvé.

