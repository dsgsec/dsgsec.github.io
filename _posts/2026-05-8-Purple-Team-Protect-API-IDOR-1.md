## Introduction

Au moment ou je rédige ces lignes nous sommes à quelques semaines depuis la fuite de données qui à touché l’ANTS (France Titre). Pour rappel, les données des utilisateurs de l’ANTS ont été compromises suite à l’exploitation d’une vulnérabilité IDOR. 

Mais c’est loin d’être un cas isolé, de nombreuses applications ont subis des fuites de données à cause de vulnérabilités IDOR, ce qui en fait un vecteur d’attaque très répandu en ce moment. 

Ces attaques explosent depuis quelques années surtout à cause de la généralisation des APIs, des apps mobiles et des architectures microservices. En 2026, quasiment toutes les applications fonctionnent comme ça.

D’ailleurs ce n’est pas pour rien si les vulnérabilités de type ***Broken Access Control*** sont toujours aussi hautes dans les deux derniers tops 10 OWASP !

![Top 10 OWASP](/assets/img/purple-team-crapi-idor/top10_owasp.png)

### Les vulnérabilités IDOR

Les vulnérabilités de type IDOR (*Insecure Direct Object Reference*), comptent parmi les failles les plus répandues du web contemporain. Elles apparaissent lorsqu'une application expose directement des :

- identifiants d'objets
- comptes
- documents
- commandes
- dossiers

Sans vérifier correctement que l'utilisateur connecté a réellement le droit d'y accéder.

![Schéma explication IDOR](/assets/img/purple-team-crapi-idor/idor_schema.png)

Concrètement, une application vulnérable peut permettre à un utilisateur de modifier simplement un identifiant dans une URL ou une requête API pour accéder aux données d'un autre compte. Dans de nombreux cas, il ne s'agit pas d'un piratage sophistiqué, mais d'un défaut de contrôle d'accès côté serveur : l'application vérifie que l'utilisateur est authentifié, sans vérifier qu'il est autorisé à consulter la ressource demandée.

### Détection des IDOR

Les vulnérabilités de type IDOR,désormais souvent classées comme ***Broken Object Level Authorization*** (BOLA) dans les APIs modernes. Sont paradoxalement parmi les failles les plus simples à exploiter et les plus difficiles à détecter. Contrairement aux injections SQL ou aux vulnérabilités XSS, elles ne reposent pas sur une anomalie technique visible, mais sur une erreur de logique métier : le serveur ne vérifie pas correctement qu'un utilisateur a le droit d'accéder à une ressource donnée.

![Meme IDOR](/assets/img/purple-team-crapi-idor/idor_meme.png)

Dans une application moderne, chaque objet possède généralement un identifiant exposé via une API : utilisateur, facture, document, commande ou conversation. Lorsqu'un attaquant modifie cet identifiant et que le serveur renvoie malgré tout les données d'un autre utilisateur, la faille devient exploitable. 

> **Oui mais avec tous les outils que nous avons aujourd’hui, l’IA etc, nous devrions détecter ces vulnérabilités facilement non ?**

C’est là que ça se complique ! Pourtant, du point de vue d'un scanner automatisé, la requête semble parfaitement valide : l'utilisateur est authentifié, la réponse est un "200 OK", et aucun comportement anormal n'apparaît techniquement. 

Le problème, c’est que ces outils analysent des signaux techniques, pas la logique métier de l’application. Ils ne savent pas si un utilisateur est censé accéder à une ressource précise, ni si une facture, un dossier ou un profil appartient réellement à la personne connectée. Tant que la réponse HTTP est cohérente et que l’application ne renvoie pas d’erreur explicite, la requête est souvent considérée comme “saine”.

C’est ce que nous allons voir dans cet article, comment utiliser des compétences en sécurité offensive (Red Team) pour afiner nos outils de détections (Blue Team) pour sécuriser notre application avec une approche Purple Team !

### Observabilité : la pièce manquante dans la détection des IDOR

Dans les environnements modernes basés sur des API, les vulnérabilités de type IDOR ne posent pas uniquement un problème de sécurité applicative, mais surtout un problème de visibilité.

Les applications génèrent généralement des logs HTTP classiques : requête, endpoint, statut de réponse. Ces informations sont suffisantes pour du debug technique, mais insuffisantes pour comprendre un comportement utilisateur.

Dans le cas d'un abus IDOR, chaque requête prise isolément semble légitime :

- l'utilisateur est authentifié,
- la requête est valide,
- la réponse est cohérente (souvent un code 200).

Rien, à ce niveau, ne permet d'identifier une intention malveillante.

Le problème n'est donc pas l'absence de logs, mais l'absence de contexte.

Nous ne savons pas :

- si les ressources appartiennent réellement à l'utilisateur,
- si une séquence d'accès est cohérente avec un usage normal,
- si plusieurs requêtes forment un pattern d'énumération,
- ni comment ces actions s'inscrivent dans un parcours utilisateur complet.

C'est précisément ce manque de visibilité que l'observabilité cherche à résoudre.

![Meme Observabilité](/assets/img/purple-team-crapi-idor/observability_meme.png)

## Présentation du lab

Bon c’est bien beau tout ça, maintenant que nos bases sont posées, le contexte défini, on fait quoi ?Et bien nous allons apprendre à détecter une attaque IDOR sur une application vulnérable. Nous allons donc simuler un scénario d’abus réel et reconstruire progressivement la capacité de détection.

Maintenant que les bases théoriques sont posées, nous allons passer à la mise en pratique.

Dans ce lab, nous allons utiliser une application volontairement vulnérable, OWASP crAPI, afin de simuler un scénario réaliste d'abus d'API basé sur une vulnérabilité de type IDOR.

L'objectif n'est pas uniquement d'exploiter la vulnérabilité, mais de comprendre comment ce type d'attaque se manifeste dans un environnement réel, et surtout pourquoi il est difficile de la détecter avec des approches classiques basées uniquement sur des logs applicatifs.

Nous allons donc simuler un scénario d'attaque, analyser les signaux disponibles, puis reconstruire progressivement une capacité de détection en introduisant des mécanismes d'observabilité et de télémétrie.

### OWASP crAPI

> crAPI (Completely Ridiculous API), simule une application Web basée sur des microservices pilotée par API qui est une plate-forme pour les propriétaires de véhicules.
> 
> 
> C’est une application intentionnellement vulnérable. Cependant, crAPI est principalement rempli de vulnérabilités d'API dans le but d'enseigner, d'apprendre et de pratiquer la sécurité de l'API.
> 

![Apercu crAPI](/assets/img/purple-team-crapi-idor/crapi.png)


C’est donc le support parfait pour simuler une application à attaquer et à sécuriser !

### Point de départ

Nous allons partir de littéralement 0, aucune protection, crAPI sera vu comme une application en production fraichement déployé.

> Nous allons bien-sûr installer l’application ensemble dans cet article afin que les personnes voulant reproduire les actions chez elles puissent le faire dans les meilleurs conditions !
> 

Une fois l’application installé, nous allons réaliser les phases d’attaques classiques :

- Reconnaissance
- Identification de la faille
- Exploitation
- Extraction des données

Une fois l’attaque réalisé, nous allons nous mettre dans la peau des équipes de défense et voir ce que nous avons pu en tirer ! Avons-nous vu quelque chose ? Quoi ? Pourquoi ? Avons-nous réussi à stopper l’attaque ? Pourquoi ? 

![Schéma Lab](/assets/img/purple-team-crapi-idor/tp_schema.png)

Et toutes autres question qui empêche nos amis de la Blue Team de dormir la nuit

![alt text](/assets/img/purple-team-crapi-idor/cyberrisk_meme.png)

## Installation de crAPI

Nous allons déployer crAPI via Docker, ca sera beaucoup plus simple pour nous ! Donc assurez-vous juste d’avoir Docker d’installer sur votre machine avant d’aller plus loin.

- https://www.docker.com/get-started/

Vous pouvez retrouver les sources de crAPI directement sur Github

- https://github.com/OWASP/crAPI

Vous aurez juste à cloner le repo sur votre machine

```bash
$ git clone https://github.com/OWASP/crAPI
Cloning into 'crAPI'...
remote: Enumerating objects: 7153, done.
remote: Counting objects: 100% (132/132), done.
remote: Compressing objects: 100% (84/84), done.
remote: Total 7153 (delta 82), reused 48 (delta 48), pack-reused 7021 (from 2)
Receiving objects: 100% (7153/7153), 8.63 MiB | 21.04 MiB/s, done.
Resolving deltas: 100% (3901/3901), done.
```

Puis à vous rendre dans `crAPI/deploy/docker`

```bash
cd crAPI/deploy/docker
```

Avant de lancer l’application nous allons modifier le fichier .`env` situé dans ce dossier et remplacer la valeur suivante :

```bash
LISTEN_IP="127.0.0.1"
```

Par :

```bash
LISTEN_IP="0.0.0.0"
```

Cela nous permettra d’accéder à l’application en dehors de `127.0.0.1`.

Puis lancer notre application via `docker compose` :

```bash
$ docker-compose up -d
[+] up 118/118
 ✔ Image mongo:4.4                       Pulled                                                                                                                                                                                         39.7s
 ✔ Image postgres:14                     Pulled                                                                                                                                                                                         37.2s
 ✔ Image crapi/crapi-identity:latest     Pulled                                                                                                                                                                                         45.5s
 ✔ Image crapi/mailhog:latest            Pulled                                                                                                                                                                                         11.6s
 ✔ Image crapi/crapi-web:latest          Pulled
 <SNIP>
```

Une fois l’installation terminé vous devriez pouvoir accéder à `crAPI` via votre navigateur sur le port `8888`!

![Login crAPI](/assets/img/purple-team-crapi-idor/crapi_login.png)


## Découverte de crAPI

En explorant la barre de navigation nous pouvons déjà constater que nous pouvons nous `connecter` mais aussi `créer un compte`, ce que nous allons faire de ce pas afin d’accéder au reste de l’application.

Nous pouvons accéder au formulaire de création de compte sur : http://127.0.0.1:8888/signup

![Création de compte](/assets/img/purple-team-crapi-idor/crapi_register.png)

Nous pouvons désormais nous connecter pour accéder à crAPI !

![Index crAPI](/assets/img/purple-team-crapi-idor/crapi_index.png)

Et nous pouvons même explorer la boutique pour acheter des pièce de voiture sur `/shop`

![crAPI Shop](/assets/img/purple-team-crapi-idor/crapi_shop.png)

## Découverte d’une vulnérabilité IDOR

### Achat de pièces

Si nous regardons de plus près le shop, il est possible d’`acheter` des pièces.

Par exemple achetons une roue (`Wheel`), en cliquant sur le bouton `Buy` du produit.

Et nous avons bien la validation comme quoi nous avons acheter notre produit !

![Order success](/assets/img/purple-team-crapi-idor/order_success.png)

### Order details

Une fois l’achat validé nous pouvons accéder au détail de notre commande

![Bouton d'accès détail de la commande](/assets/img/purple-team-crapi-idor/crapi_order_detail_button.png)

Cela nous redirige vers l’url :

- http://127.0.0.1:8888/orders?order_id=x

Ou `X` est l’id de notre commande. Par exemple dans notre cas il s’agit de la commande numéro `7` !

![Détail de la commande](/assets/img/purple-team-crapi-idor/order_detail.png)

Vous voyez ou je veux en venir ? Effectivement il est très facile de deviner les id des commandes, ceux-ci on l’air de partir de 0 et de s’incrémenter séquentiellement : 

- 0
- 1
- 2
- 3
- 4
- etc

Il est donc facile de faire un script qui va créer une liste partant de 0 et allant à 10 000 par exemple et qui va relever les ID existants pour en extraire les données !

### Découverte de l’API

Par ailleurs, si nous ouvrons l’onglet `Network` (`Réseau`) de notre navigateur avec la fonction `Inspecter l’élément` et que nous rechargeons notre page, nous pouvons constater les différentes ressources appelés par notre page


Chose intéressante, notre page contacte une `API`, qui permet de renvoyer les données à afficher sur notre page

![Détail appel réseau](/assets/img/purple-team-crapi-idor/crapi_api.png)

Cela nous permets de savoir que la page interroge l’`API crAPI` via 

- `http://127.0.0.1:8888/workshop/api/shop/orders/<id>`

Afin de récupérer les données de la commande ciblé par le paramètre `id`, et de l’afficher sur notre page !

Donc que se passe t’il si nous changeons le paramètre `id` par un autre ?

### Exploitation

Pour faire ça proprement nous allons utiliser `Burpsuite`, notre intercepteur de requête HTTP préféré !

> Nous allons changer l’url pour naviguer sur crAPI dans notre url, car actuellement nous étions sur `127.0.0.1`, et `burp` n’intercepte aucune requête provenant de `127.0.0.1` ou `localhost`. Nous allons donc utilisé l’adresse IP de notre poste. 

![Changement adresse IP](/assets/img/purple-team-crapi-idor/change_ip.png)

#### Interception dans Burpsuite

Burp lancé et configuré, nous allons recharger notre page pour intercepter la requête pour `/orders?order_id=7`

Une fois la page rechargée, nous voyons bien notre requête intercepté dans `burpsuite` !

![Interception dans burp](/assets/img/purple-team-crapi-idor/burp_intercept.png)

Nous allons cliquer sur `Forward` pour la laisser passer.

Et là nous pouvons voir que la prochaine requête est celle qui va récupérer les données de notre commande via l’API ! 

![Interception request API dans burp](/assets/img/purple-team-crapi-idor/burp_intercept_api.png)

Nous allons l’envoyer dans notre Repeater pour la manipuler à souhait et jouer avec !

> Pour cela on peut faire `CTRL + R` ou `Clique Droit > Send to repeater`
> 

Nous pouvons désormais la manipuler à volonté !

![Passage de la requête dans repeater](/assets/img/purple-team-crapi-idor/burp_repeater.png)

Appuyons sur `send` pour observer la réponse de notre requête !

Effectivement nous avons bien les données de notre commande :

- L’utilisateur qui passe commande
    - Ses coordonnées
- Le produit
    - Les données du produits
- La quantité
- Status de la commande
- etc

![Lancement de la requête dans repeater](/assets/img/purple-team-crapi-idor/burp_repeater_run.png)

> **Si je change l’ID de la commande (7) par un autre, comme 1, l’application devrait vérifier côté serveur que cette commande appartient bien à l’utilisateur connecté ([damien@crapi.com](mailto:damien@crapi.com)) et refuser l’accès si ce n’est pas le cas ?**
> 

En principe, si c’est sécurisé : **OUI**

Essayons ! Dans notre URL modifions 7 par 1 par exemple et observons le résultat

![Modification de l'id](/assets/img/purple-team-crapi-idor/burp_change_idor.png)

Aïe ! On a accès au détail de la commande d’un autre utilisateur, donc accès à des données auxquelles on ne devrait pas avoir accès…

Voilà comment fonctionne une IDOR.

## Exploitation massive d’une vulnérabilité IDOR

Nous avons vu comment exploiter une `IDOR` pour passer des détails d’une commande d’un utilisateur à un autre. 

Nous allons maintenant aller plus loin en automatisant ce processus. L’objectif est de parcourir une plage d’identifiants afin d’identifier ceux qui sont valides, et d’observer si l’application retourne effectivement des données associées à ces ressources.

Pour cela nous allons utiliser la fonction `intruder` de `Burpsuite` pour envoyer des requêtes massives en changeant l’`id` dans l’url entre chaque requêtes.

> Pour cela on peut faire `CTRL + I` ou `Clique Droit > Send to intruder`
> 

### Configuration intruder

Nous allons surligner l’`id` dans notre URL et appuyer sur `Add §` pour spécifier quelle valeur doit être modifiée entre chaque requête.

![Burp intruder](/assets/img/purple-team-crapi-idor/burp_intruder.png)

Nous allons maintenant configurer nos `payloads`, c’est ici que nous allons dire par quoi remplacer le chiffre `7` dans notre url.

Dans le volet de droite nous allons définir :

- Payload Type : Numbers
- From : 1
- To :  100

![Burp intruder payload](/assets/img/purple-team-crapi-idor/burp_intruder_payload.png)

Grâce à cette configuration nous allons tester les ID de manière automatisée de l’ID 1 à 100 !

### Lancement de l’attaque

Pour lancer notre attaque nous avons juste à cliquer sur le bouton `Start Attack`.

Lors de l’attaque le code HTTP de réponse est extrêmement pertinent car si le code de retour est :

- 200 : Alors nous avons accès au détail de la commande de l’ID associé
- 500 : Aucun détail de disponible

Par exemple ici, nous savons que nous pouvons récupérer les détails des commandes des ID 1 à 9

![Burp code HTTP](/assets/img/purple-team-crapi-idor/burp_code_http.png)

![Détail d'une autre commande dans intruder](/assets/img/purple-team-crapi-idor/detail_intruder.png)

Et voilà nous avons exploité notre IDOR et extrait massivement les données des commandes des utilisateurs de crAPI !

Bon, et côté défense, qu’avons nous ?

## Blue-Team : intervention !

Comme vous le savez, actuellement notre application est déployé avec le strict minium, et il en va de même pour notre infrastructure : 

- Pas de SIEM
- Pas de monitoring
- Pas de puit de log
- Pas de dashboard

Donc si aujourd’hui nous subissons l’attaque que nous avons réalisée au-dessus, nous ne sommes pas en capacité de la détecter (ce qui est déjà très problématique), de réagir, et réaliser des investigations.

### Investigation

Alors partons du principe que nous avons été victime de cette attaque et que nos données ont fuitées (aïe) !

Nous devons alors investiguer sur la compromission pour savoir :

- Quand à eu lieu l’incident
- Comment ?
- Qui ?
- Où ?

Pour cela allons interroger les logs !

 

### Accesslogs nginx

On va commencer simple. Si nous allons voir les `access log` de notre service web `Nginx` ?

Pour ça nous pouvons faire un `docker logs` sur notre container web pour afficher les logs `nginx` !

```bash
docker logs crapi-web
```

Nous pouvons voir les requêtes exécutées lors de notre attaque !

![Logs nginx](/assets/img/purple-team-crapi-idor/http_logs.png)

Nous pouvons voir que :

- Les ID de `1 à 9` sont en code `HTTP 200 OK`
- Les ID `après 9` sont en erreurs : `HTTP 500 SERVER ERROR`

Grâce à ces logs nous savons :

- Quels détails de commandes ont été consultés, donc potentiellement exfiltrés ?
- À quel moment ?
- Et depuis quelle adresse IP ?

En revanche, il a fallu attendre que nous soyons informés de la fuite pour lancer l’investigation. De plus, nous ne savons pas depuis quel compte de l’application l’attaque a été réalisée (oui c’est terrible en effet).

![Bob meme](/assets/img/purple-team-crapi-idor/bob_meme.png)

Par ailleurs, nous pouvons également noter que l’adresse IP source n’est pas nécessairement fiable, car l’attaquant peut utiliser des proxys ou des techniques de rotation d’IP, par exemple en changeant d’adresse toutes les quelques requêtes.

On va remédier à tout ça, en intégrant de observabilité dans notre stack !

## Modification du docker-compose.yml

Avant de continuer nous allons modifier notre `docker-compose.yml`.

Pourquoi ? Car actuellement, ce dernier va chercher les images `crAPI` dans Docker Hub, donc si nous voulons faire des modifications dans le code source de notre application, ces dernières ne seront pas prises en compte car les images ne seront pas construite à partir de nos sources.

Pour cela, modifier `crAPI/deploy/docker/docker-compose.yml` par ce dernier

```yaml
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#         http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

services:

  crapi-identity:
    container_name: crapi-identity
    image: crapi/crapi-identity:${VERSION:-latest}
    volumes:
      - ./keys:/app/keys
    environment:
      - LOG_LEVEL=${LOG_LEVEL:-INFO}
      - DB_NAME=crapi
      - DB_USER=admin
      - DB_PASSWORD=crapisecretpassword
      - DB_HOST=postgresdb
      - DB_PORT=5432
      - SERVER_PORT=${IDENTITY_SERVER_PORT:-8080}
      - ENABLE_SHELL_INJECTION=${ENABLE_SHELL_INJECTION:-false}
      - JWT_SECRET=crapi
      - MAILHOG_HOST=mailhog
      - MAILHOG_PORT=1025
      - MAILHOG_DOMAIN=example.com
      - SMTP_HOST=smtp.example.com
      - SMTP_PORT=587
      - SMTP_EMAIL=user@example.com
      - SMTP_PASS=xxxxxxxxxxxxxx
      - SMTP_FROM=no-reply@example.com
      - SMTP_AUTH=true
      - SMTP_STARTTLS=true
      - JWT_EXPIRATION=604800000
      - ENABLE_LOG4J=${ENABLE_LOG4J:-false}
      - API_GATEWAY_URL=https://api.mypremiumdealership.com
      - TLS_ENABLED=${TLS_ENABLED:-false}
      - TLS_KEYSTORE_TYPE=PKCS12
      - TLS_KEYSTORE=classpath:certs/server.p12
      - TLS_KEYSTORE_PASSWORD=passw0rd
      - TLS_KEY_PASSWORD=passw0rd
      - TLS_KEY_ALIAS=identity
    depends_on:
      postgresdb:
        condition: service_healthy
      mongodb:
        condition: service_healthy
      mailhog:
        condition: service_healthy
    healthcheck:
      test: /app/health.sh
      interval: 15s
      timeout: 15s
      retries: 15
    deploy:
      resources:
        limits:
          cpus: '0.8'
          memory: 384M

  crapi-community:
    container_name: crapi-community
    build:
      context: ../../services/community
      dockerfile: Dockerfile
    image: crapi/crapi-community:local
    environment:
      - LOG_LEVEL=${LOG_LEVEL:-INFO}
      - IDENTITY_SERVICE=crapi-identity:${IDENTITY_SERVER_PORT:-8080}
      - DB_NAME=crapi
      - DB_USER=admin
      - DB_PASSWORD=crapisecretpassword
      - DB_HOST=postgresdb
      - DB_PORT=5432
      - SERVER_PORT=${COMMUNITY_SERVER_PORT:-8087}
      - MONGO_DB_HOST=mongodb
      - MONGO_DB_PORT=27017
      - MONGO_DB_USER=admin
      - MONGO_DB_PASSWORD=crapisecretpassword
      - MONGO_DB_NAME=crapi
      - TLS_ENABLED=${TLS_ENABLED:-false}
      - TLS_CERTIFICATE=certs/server.crt
      - TLS_KEY=certs/server.key
    depends_on:
      postgresdb:
        condition: service_healthy
      mongodb:
        condition: service_healthy
      crapi-identity:
        condition: service_healthy
    healthcheck:
      test: /app/health.sh
      interval: 15s
      timeout: 15s
      retries: 15
    deploy:
      resources:
        limits:
          cpus: '0.3'
          memory: 192M

  crapi-workshop:
    container_name: crapi-workshop
    build:
      context: ../../services/workshop
      dockerfile: Dockerfile
    image: crapi/crapi-workshop:local
    environment:
      - LOG_LEVEL=${LOG_LEVEL:-INFO}
      - IDENTITY_SERVICE=crapi-identity:${IDENTITY_SERVER_PORT:-8080}
      - DB_NAME=crapi
      - DB_USER=admin
      - DB_PASSWORD=crapisecretpassword
      - DB_HOST=postgresdb
      - DB_PORT=5432
      - SERVER_PORT=${WORKSHOP_SERVER_PORT:-8000}
      - MONGO_DB_HOST=mongodb
      - MONGO_DB_PORT=27017
      - MONGO_DB_USER=admin
      - MONGO_DB_PASSWORD=crapisecretpassword
      - MONGO_DB_NAME=crapi
      - SECRET_KEY=crapi
      - API_GATEWAY_URL=https://api.mypremiumdealership.com
      - TLS_ENABLED=${TLS_ENABLED:-false}
      - TLS_CERTIFICATE=certs/server.crt
      - TLS_KEY=certs/server.key
      - FILES_LIMIT=1000
      - GUNICORN_WORKERS=${GUNICORN_WORKERS:-4}
      - GUNICORN_TIMEOUT=${GUNICORN_TIMEOUT:-120}
      - GUNICORN_MAX_REQUESTS=${GUNICORN_MAX_REQUESTS:-1000}
      - GUNICORN_MAX_REQUESTS_JITTER=${GUNICORN_MAX_REQUESTS_JITTER:-50}
      - DB_CONN_MAX_AGE=${DB_CONN_MAX_AGE:-600}
    depends_on:
      postgresdb:
        condition: service_healthy
      mongodb:
        condition: service_healthy
      crapi-identity:
        condition: service_healthy
      crapi-community:
        condition: service_healthy
    healthcheck:
      test: /app/health.sh
      interval: 15s
      timeout: 15s
      retries: 15
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M

  crapi-chatbot:
    container_name: crapi-chatbot
    build:
      context: ../../services/chatbot
      dockerfile: Dockerfile
    image: crapi/crapi-chatbot:local
    ports:
      - "${LISTEN_IP:-127.0.0.1}:5500:5500"
    environment:
      - TLS_ENABLED=${TLS_ENABLED:-false}
      - SERVER_PORT=${CHATBOT_SERVER_PORT:-5002}
      - WEB_SERVICE=crapi-web
      - IDENTITY_SERVICE=crapi-identity:${IDENTITY_SERVER_PORT:-8080}
      - DB_NAME=crapi
      - DB_USER=admin
      - DB_PASSWORD=crapisecretpassword
      - DB_HOST=postgresdb
      - DB_PORT=5432
      - MONGO_DB_HOST=mongodb
      - MONGO_DB_PORT=27017
      - MONGO_DB_USER=admin
      - MONGO_DB_PASSWORD=crapisecretpassword
      - MONGO_DB_NAME=crapi
      - API_USER=admin@example.com
      - API_PASSWORD=Admin!123
      - OPENAPI_SPEC=/app/resources/crapi-openapi-spec.json
      - CHATBOT_LIFE=${CHATBOT_LIFE:-1}
      - CHATBOT_LLM_PROVIDER=${CHATBOT_LLM_PROVIDER:-openai}
      - CHATBOT_LLM_MODEL=${CHATBOT_LLM_MODEL:-}
      - CHATBOT_EMBEDDINGS_MODEL=${CHATBOT_EMBEDDINGS_MODEL:-}
      - CHATBOT_EMBEDDINGS_DIMENSIONS=${CHATBOT_EMBEDDINGS_DIMENSIONS:-1536}
      - CHATBOT_OPENAI_API_KEY=${CHATBOT_OPENAI_API_KEY:-}
      - CHATBOT_OPENAI_BASE_URL=${CHATBOT_OPENAI_BASE_URL:-}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY:-}
      - AZURE_OPENAI_API_KEY=${AZURE_OPENAI_API_KEY:-}
      - AZURE_AD_TOKEN=${AZURE_AD_TOKEN:-}
      - AZURE_OPENAI_ENDPOINT=${AZURE_OPENAI_ENDPOINT:-}
      - AZURE_OPENAI_API_VERSION=${AZURE_OPENAI_API_VERSION:-2024-02-15-preview}
      - AZURE_OPENAI_CHAT_DEPLOYMENT=${AZURE_OPENAI_CHAT_DEPLOYMENT:-}
      - AZURE_OPENAI_EMBEDDINGS_DEPLOYMENT=${AZURE_OPENAI_EMBEDDINGS_DEPLOYMENT:-}
      - GROQ_API_KEY=${GROQ_API_KEY:-}
      - MISTRAL_API_KEY=${MISTRAL_API_KEY:-}
      - COHERE_API_KEY=${COHERE_API_KEY:-}
      - AWS_BEARER_TOKEN_BEDROCK=${AWS_BEARER_TOKEN_BEDROCK:-}
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID:-}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY:-}
      - AWS_SESSION_TOKEN=${AWS_SESSION_TOKEN:-}
      - AWS_REGION=${AWS_REGION:-}
      - AWS_ASSUME_ROLE_ARN=${AWS_ASSUME_ROLE_ARN:-}
      - AWS_EXTERNAL_ID=${AWS_EXTERNAL_ID:-}
      - AWS_ROLE_SESSION_NAME=${AWS_ROLE_SESSION_NAME:-crapi-chatbot-session}
      - GOOGLE_APPLICATION_CREDENTIALS=${GOOGLE_APPLICATION_CREDENTIALS:-}
      - VERTEX_PROJECT=${VERTEX_PROJECT:-}
      - VERTEX_LOCATION=${VERTEX_LOCATION:-}
      - MAX_CONTENT_LENGTH=50000
      - CHROMA_HOST=chromadb
      - CHROMA_PORT=8000
    depends_on:
      mongodb:
        condition: service_healthy
      crapi-identity:
        condition: service_healthy
      chromadb:
        condition: service_healthy

  crapi-web:
    container_name: crapi-web
    build:
      context: ../../services/web
      dockerfile: Dockerfile
    image: crapi/crapi-web:local
    ports:
      - "${LISTEN_IP:-127.0.0.1}:8888:80"
      - "${LISTEN_IP:-127.0.0.1}:30080:80"
      - "${LISTEN_IP:-127.0.0.1}:8443:443"
      - "${LISTEN_IP:-127.0.0.1}:30443:443"
    environment:
      - COMMUNITY_SERVICE=crapi-community:${COMMUNITY_SERVER_PORT:-8087}
      - IDENTITY_SERVICE=crapi-identity:${IDENTITY_SERVER_PORT:-8080}
      - WORKSHOP_SERVICE=crapi-workshop:${WORKSHOP_SERVER_PORT:-8000}
      - CHATBOT_SERVICE=crapi-chatbot:${CHATBOT_SERVER_PORT:-5002}
      - MAILHOG_WEB_SERVICE=mailhog:8025
      - TLS_ENABLED=${TLS_ENABLED:-false}
    depends_on:
      crapi-community:
        condition: service_healthy
      crapi-identity:
        condition: service_healthy
      crapi-workshop:
        condition: service_healthy
    healthcheck:
      test: curl 0.0.0.0:80/health
      interval: 15s
      timeout: 15s
      retries: 15
    deploy:
      resources:
        limits:
          cpus: '0.3'
          memory: 128M

  postgresdb:
    container_name: postgresdb
    image: 'postgres:14'
    command: ["postgres", "-c", "max_connections=500"]
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: crapisecretpassword
      POSTGRES_DB: crapi
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready" ]
      interval: 15s
      timeout: 15s
      retries: 15
    volumes:
      - postgresql-data:/var/lib/postgresql/data/
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M

  mongodb:
    container_name: mongodb
    image: 'mongo:4.4'
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: crapisecretpassword
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongo mongodb:27017/test --quiet
      interval: 15s
      timeout: 15s
      retries: 15
      start_period: 20s
    volumes:
      - mongodb-data:/data/db
    deploy:
      resources:
        limits:
          cpus: '0.3'
          memory: 128M

  chromadb:
    container_name: chromadb
    image: 'chromadb/chroma:latest'
    environment:
      IS_PERSISTENT: 'TRUE'
    healthcheck:
      test: [ "CMD", "/bin/bash", "-c", "cat < /dev/null > /dev/tcp/localhost/8000" ]
      interval: 15s
      timeout: 15s
      retries: 15
      start_period: 20s
    volumes:
      - chromadb-data:/data

  mailhog:
    user: root
    container_name: mailhog
    build:
      context: ../../services/mailhog
      dockerfile: Dockerfile
    image: crapi/mailhog:local
    environment:
      MH_MONGO_URI: admin:crapisecretpassword@mongodb:27017
      MH_STORAGE: mongodb
    ports:
      - "${LISTEN_IP:-127.0.0.1}:8025:8025"
    healthcheck:
      test: [ "CMD", "nc", "-z", "localhost", "8025" ]
      interval: 15s
      timeout: 15s
      retries: 15
    deploy:
      resources:
        limits:
          cpus: '0.3'
          memory: 128M

  api.mypremiumdealership.com:
    container_name: api.mypremiumdealership.com
    build:
      context: ../../services/gateway-service
      dockerfile: Dockerfile
    image: crapi/gateway-service:local
    healthcheck:
      test: bash -c 'echo -n "GET / HTTP/1.1\n\n" > /dev/tcp/127.0.0.1/443'
      interval: 15s
      timeout: 15s
      retries: 15
      start_period: 15s
    deploy:
      resources:
        limits:
          cpus: '0.1'
          memory: 50M

volumes:
  mongodb-data:
  postgresql-data:
  chromadb-data:
```

Vous noterez que les modifications interviennent essentiellement sur les lignes :

```yaml
build:
      context: ../../services/identity
```

Ces via ces lignes que nous indiquons à partir de quel dossier il faut build les images pour nos services !

Nous pouvons relancer un build avec

```bash
$ docker compose -f deploy/docker/docker-compose.yml build
```

Et relancer la construction de nos containers

```bash
$ cd crAPI/deploy/docker
$ docker compose up -d
```

Dans le prochain article nous allons voir comment intégrer de l’observabilité à notre application !
