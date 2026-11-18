---
title: "Application Security Detection & Response 01 : Définition du produit"
date: 2026-11-18
categories: [purple, devsecops, appsec, detection]
tags: [purple-team, devsecops, appsec, detection]
image:
  path: /assets/img/appsec-dfir/banniere.png
---

## Introduction

Avant de commencer à développer une application ou à analyser ses vulnérabilités, il est essentiel de comprendre ce que l'on cherche réellement à protéger.

Une application ne se résume pas à son code source, à ses API ou à son infrastructure. Elle répond avant tout à un besoin métier, manipule des données et permet à différents utilisateurs de réaliser des opérations qui peuvent avoir un impact sur l'organisation.

Dans le cadre de notre laboratoire, nous allons développer **DocuVault**, une plateforme SaaS fictive de gestion documentaire destinée aux entreprises.

Cette application constituera notre environnement de travail pour les prochaines étapes du projet : conception de l'architecture, `Threat Modeling`, développement, instrumentation, détection et exercices offensifs.

Avant d'aborder les aspects techniques, nous devons donc définir le produit :

- Quel problème DocuVault cherche-t-il à résoudre ?
- Qui sont ses utilisateurs ?
- Quelles fonctionnalités sont disponibles ?
- Quelles données sont manipulées ?
- Quelles opérations sont critiques pour l'activité ?
- Quelles règles métier doivent être respectées ?

Ces éléments nous permettront de disposer d'une vision fonctionnelle suffisamment précise pour identifier les risques de sécurité dans les prochains articles.

## Contexte Business : Présentation du produit

### Qu'est-ce que DocuVault ?

`DocuVault` est une plateforme SaaS permettant aux entreprises de centraliser, organiser et partager leurs documents depuis un espace de travail sécurisé.

Chaque entreprise dispose de son propre espace, appelé `*Organization*`, dans lequel elle peut :

- inviter des collaborateurs
- créer des dossiers
- déposer des documents
- gérer les permissions d'accès.

La plateforme repose sur un modèle `multi-tenant` : **plusieurs organisations utilisent la même application et partagent une infrastructure commune**, tout en devant conserver une séparation stricte de leurs données.

Prenons l'exemple de trois entreprises utilisant **DocuVault** :

| Organisation | Utilisateurs | Documents |
| --- | --- | --- |
| Acme Corporation | Alice, Bob | Contrats, factures |
| Globex | Charlie, David | Rapports, documents RH |
| Initech | Emma, Frank | Documents internes |

Chaque organisation possède son propre environnement logique.

Un utilisateur d'**Acme Corporation** ne doit **jamais** pouvoir consulter les documents appartenant à **Globex** ou **Initech**, sauf si une fonctionnalité de partage explicitement autorisée le permet.

Cette séparation constitue l'une des règles métier fondamentales de `DocuVault`.

### Les enjeux métier

Les documents hébergés sur la plateforme peuvent contenir des informations importantes pour les organisations :

- Contrats commerciaux.
- Documents administratifs.
- Factures et informations financières.
- Documents internes.
- Informations relatives aux collaborateurs.
- Documents confidentiels.

Une mauvaise gestion des accès pourrait entraîner une divulgation d'informations entre organisations, une modification non autorisée de documents ou la compromission de données sensibles.

La sécurité de DocuVault repose donc sur plusieurs objectifs :

| Objectif | Description |
| --- | --- |
| Confidentiality | Garantir que seuls les utilisateurs autorisés accèdent aux documents |
| Integrity | Empêcher les modifications non autorisées |
| Availability | Garantir la disponibilité des documents et des fonctionnalités |
| Tenant Isolation | Garantir la séparation des données entre organisations |
| Traceability | Conserver une trace des opérations sensibles |

Ces objectifs serviront de référence lors de notre analyse des risques.

## Rôles utilisateurs & Permissions : Les utilisateurs de la plateforme

### Administrateur de l’organisation

Chaque organisation possède au moins un `administrateur` ****chargé de gérer son espace.

Ses responsabilités comprennent :

- Inviter et supprimer des collaborateurs.
- Attribuer des rôles aux utilisateurs.
- Gérer les documents de l'organisation.
- Administrer les paramètres de l'espace.
- Gérer les permissions et les partages.

Un administrateur de l’organisation ne doit toutefois pas disposer de privilèges sur les autres organisations.

### Manager

Le `Manager` intervient dans la gestion quotidienne des documents et des collaborateurs.

Il peut notamment :

- Créer et organiser des documents.
- Modifier les documents autorisés.
- Partager des documents.
- Consulter l'activité documentaire de son organisation.

Il ne dispose pas des mêmes privilèges d'administration qu'un administrateur de l’organisation.

### Membre

Le `membre` représente l'utilisateur standard de la plateforme.

Il peut :

- Consulter les documents auxquels il a accès.
- Déposer de nouveaux documents selon les permissions accordées.
- Modifier ou supprimer ses propres documents lorsque cela est autorisé.
- Partager des documents si son rôle et les permissions associées le permettent.

### Synthèse des rôles

| Fonctionnalité | Platform Admin | Org Admin | Manager | Membre |
| --- | --- | --- | --- | --- |
| Administrer la plateforme | Oui | Non | Non | Non |
| Gérer son organisation | Oui* | Oui | Non | Non |
| Inviter des utilisateurs | Oui* | Oui | Non | Non |
| Attribuer les rôles | Oui* | Oui | Non | Non |
| Consulter les documents autorisés | Oui* | Oui | Oui | Oui |
| Créer des documents | Oui* | Oui | Oui | Oui** |
| Modifier des documents | Oui* | Oui | Oui** | Oui** |
| Supprimer des documents | Oui* | Oui | Oui** | Oui** |
| Gérer les partages | Oui* | Oui | Oui** | Oui** |

> *Les privilèges globaux du Platform Administrator sont réservés aux opérations d'administration et ne signifient pas qu'il agit systématiquement comme membre de chaque organisation.*
> 

**Les opérations sur les documents restent soumises aux permissions associées à la ressource.**

Cette dernière distinction est importante : le rôle d'un utilisateur ne doit pas nécessairement suffire à déterminer s'il peut accéder à chaque document.

Nous devrons également prendre en compte les permissions propres aux ressources et les relations de partage.

## Les fonctionnalités de DocuVault

Nous allons maintenant définir les principales fonctionnalités de l'application.

Cette étape permettra de déterminer les opérations métier que nous devrons implémenter.

### Authentification & gestion des sessions

La plateforme doit permettre aux utilisateurs de s'authentifier et d'accéder à leur espace de travail.

Fonctionnalités prévues :

- Création de compte par invitation.
- Connexion avec identifiant et mot de passe.
- Déconnexion.
- Gestion des sessions.
- Réinitialisation du mot de passe.
- Consultation des informations du compte.

Les mécanismes d'authentification et de gestion des sessions devront notamment empêcher les accès non autorisés aux comptes.

### Gestion de l’organisation

Les organisations constituent le cœur du modèle multi-tenant.

Chaque organisation possède ses propres utilisateurs, documents et paramètres.

Fonctionnalités prévues :

- Création d'une organisation.
- Consultation des informations de l'organisation.
- Modification des paramètres.
- Gestion des membres.
- Gestion des invitations.

Une règle fondamentale devra être respectée : les utilisateurs ne peuvent agir que dans le périmètre des organisations auxquelles ils appartiennent et pour lesquelles ils disposent des permissions nécessaires.

### Gestion des utilisateurs

Les administrateurs d'organisation doivent pouvoir gérer les collaborateurs de leur espace.

Fonctionnalités prévues :

- Inviter un utilisateur.
- Accepter une invitation.
- Consulter les membres.
- Modifier un rôle.
- Désactiver ou supprimer un utilisateur.

Les opérations de gestion des rôles devront être particulièrement encadrées afin d'éviter les élévations de privilèges.

### Gestion des documents

La gestion documentaire représente la fonctionnalité principale de DocuVault.

Les utilisateurs pourront créer, consulter, modifier et supprimer des documents.

Fonctionnalités prévues :

- Création de dossiers.
- Création de documents.
- Consultation des documents.
- Modification des métadonnées.
- Déplacement entre dossiers.
- Suppression.
- Recherche de documents.

Chaque document sera associé à une organisation propriétaire et à un utilisateur créateur.

L'application devra vérifier les autorisations avant chaque opération sensible.

### Partage des documents

DocuVault permettra également de partager des documents entre collaborateurs.

Un utilisateur autorisé pourra accorder un accès à un autre membre de son organisation.

Les permissions de partage pourront notamment être :

| Permission | Description |
| --- | --- |
| Viewer | Consultation du document |
| Editor | Consultation et modification |
| Owner | Contrôle complet du document |

Le partage devra être limité aux utilisateurs autorisés et respecter les règles d'isolation entre organisations.

Pour la première version, nous ne prévoirons pas de partage public anonyme. Cette fonctionnalité pourrait être étudiée dans une évolution ultérieure.

### Dépôt et téléchargement des fichiers

La plateforme devra permettre le téléversement et le téléchargement de fichiers.

Fonctionnalités prévues :

- Téléversement d'un fichier.
- Consultation des métadonnées.
- Téléchargement.
- Suppression.
- Vérification du type et de la taille des fichiers.

Cette fonctionnalité introduit plusieurs problématiques de sécurité, notamment la validation des fichiers, leur stockage et le contrôle des accès.

### Historique des activités

Enfin, DocuVault conservera un historique des opérations réalisées sur la plateforme.

Cet historique permettra notamment de retrouver :

- Les connexions et déconnexions.
- Les créations et suppressions de documents.
- Les modifications de permissions.
- Les invitations d'utilisateurs.
- Les opérations de partage.
- Les accès refusés à des ressources.

Cette fonctionnalité aura également un intérêt particulier pour notre futur environnement de supervision.

Nous chercherons à distinguer l'historique métier visible par les utilisateurs des événements de sécurité techniques destinés à l'investigation.

## Les parcours utilisateurs critiques

Une fonctionnalité ne doit pas uniquement être comprise individuellement. Il faut également comprendre comment les utilisateurs interagissent avec plusieurs composants pour réaliser une opération complète.

Nous allons donc identifier quelques parcours représentatifs.

### Parcours 1 : Authentification et accès à l'organisation

**Objectif :** permettre à un utilisateur d'accéder à son espace de travail.

1. L'utilisateur ouvre DocuVault.
2. Il renseigne ses identifiants.
3. Le backend vérifie les informations d'authentification.
4. Une session est créée.
5. L'utilisateur accède à son organisation.
6. L'application charge les informations et les permissions associées.

> **Enjeu de sécurité :** garantir que l'utilisateur ne puisse accéder qu'à son propre compte et aux organisations auxquelles il appartient.
> 

### Parcours 2 : Consultation d'un document

**Objectif :** permettre à un utilisateur de consulter un document.

1. L'utilisateur ouvre son espace documentaire.
2. Il sélectionne un document.
3. Le frontend envoie une requête à l'API.
4. Le backend identifie l'utilisateur et la ressource demandée.
5. Les permissions sont vérifiées.
6. Le document est retourné si l'accès est autorisé.

> **Enjeu de sécurité :** empêcher un utilisateur d'accéder à une ressource appartenant à un autre utilisateur ou à une autre organisation sans autorisation.
> 

### Parcours 3 : Partage d'un document

**Objectif :** permettre à un utilisateur de partager un document avec un collaborateur.

1. L'utilisateur sélectionne un document.
2. Il choisit un collaborateur.
3. Il définit le niveau de permission.
4. L'application vérifie qu'il dispose du droit de partager cette ressource.
5. Une relation de partage est créée.
6. Le destinataire peut accéder au document selon les permissions accordées.

> **Enjeu de sécurité :** empêcher le partage non autorisé et éviter qu'un utilisateur puisse attribuer des permissions supérieures à celles qu'il possède.
> 

### Parcours 4 : Modification des permissions d'un utilisateur

**Objectif :** permettre à un administrateur de modifier le rôle d'un collaborateur.

1. L'administrateur ouvre la gestion des membres.
2. Il sélectionne un utilisateur.
3. Il choisit un nouveau rôle.
4. L'application vérifie ses droits d'administration.
5. Le rôle est modifié.
6. L'opération est enregistrée dans l'historique.

> **Enjeu de sécurité :** empêcher un utilisateur standard de modifier ses propres permissions ou celles d'un autre collaborateur sans autorisation.
> 

---

## Les données manipulées

Pour comprendre les risques auxquels `DocuVault` sera exposé, nous devons également identifier les données qu'il manipule.

Toutes les données n'ont pas le même niveau de sensibilité et ne présentent pas les mêmes conséquences en cas de compromission.

| Catégorie | Exemples | Sensibilité |
| --- | --- | --- |
| User Data | Nom, email, profil | Moyenne |
| Authentication Data | Hash de mot de passe, sessions, tokens | Élevée |
| Organization Data | Nom, paramètres, membres | Moyenne à élevée |
| Documents | Contrats, factures, documents internes | Élevée |
| Permissions | Rôles, droits, relations de partage | Élevée |
| Activity Data | Historique des opérations | Moyenne à élevée |

Cette classification reste volontairement initiale. Elle sera approfondie lors du `Threat Modeling`.

Nous devrons notamment porter une attention particulière aux données d'authentification, aux documents et aux informations permettant de déterminer les autorisations d'un utilisateur.

Une compromission de ces dernières pourrait avoir des conséquences directes sur la confidentialité et l'intégrité des données.

---

## Périmètre de la première version

Afin de conserver un laboratoire maîtrisable, nous allons définir un périmètre fonctionnel pour la première version de `DocuVault`.

L'objectif est de développer une application suffisamment réaliste pour étudier les problématiques de sécurité, sans chercher à reproduire toutes les fonctionnalités d'un véritable produit SaaS commercial.

### Fonctionnalités incluses

La première version comprendra :

- Authentification et gestion des sessions.
- Gestion des organisations.
- Gestion des utilisateurs et des rôles.
- Gestion des documents.
- Gestion des permissions.
- Partage entre utilisateurs d'une même organisation.
- Téléversement et téléchargement de fichiers.
- Historique des opérations sensibles.

### Fonctionnalités exclues initialement

Certaines fonctionnalités seront volontairement écartées :

- Paiement et facturation.
- Intégrations avec des services externes.
- Authentification fédérée SSO.
- Application mobile.
- Partage public de documents.
- Intelligence artificielle intégrée à l'application.

Ces fonctionnalités pourront éventuellement être ajoutées par la suite si elles apportent une réelle valeur aux scénarios de sécurité étudiés.

### Stack technique retenue

| Composant | Technologie |
| --- | --- |
| Frontend | React |
| Backend | Python / FastAPI |
| Database | PostgreSQL |
| Reverse Proxy | Nginx |
| WAF | ModSecurity v3 + OWASP CRS |
| SIEM | Wazuh |
| Containerization | Docker Compose |

La conception détaillée de cette architecture fera l'objet du prochain article.

## Conclusion

Nous avons maintenant défini les fondations fonctionnelles de DocuVault.

Nous disposons d'une vision du produit, de ses utilisateurs, de ses fonctionnalités, des données manipulées et des parcours métier les plus importants.

Ces éléments constituent une première base de travail pour la suite du laboratoire.

Il est important de souligner que la sécurité ne commence pas au moment où l'on écrit les premières lignes de code ou lorsque l'on lance un outil de pentest.

Elle commence par la compréhension du produit, de ses usages et de ce qui doit être protégé.

C'est cette compréhension qui nous permettra de construire un `Threat Model` pertinent et de définir des mécanismes de sécurité adaptés aux risques réels de l'application.

## What's Next?

Nous allons maintenant passer de la vision fonctionnelle à la conception technique.

Nous détaillerons l'architecture de `DocuVault`, les différents composants, leurs responsabilités, les flux de communication et les frontières de confiance.

Nous commencerons également à identifier les interactions entre le `frontend`, le `backend`, la `base de donnée`s et les composants d'`infrastructure`.

Cette architecture constituera le point de départ de notre `Data Flow Diagram (DFD)`, indispensable à la prochaine étape : le `Threat Modeling`.