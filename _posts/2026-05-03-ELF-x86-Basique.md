---
title: "Root-Me Cracking - ELF x86 - Basique"
date: 2026-05-01
categories: [cracking, rootme]
tags: [rootme, cracking, reverse, easy, linux]
image:
  path: /assets/img/rootme/rootme_card_template.png
---

## Contexte
> - Nom: ELF x86 - 0 Basique
> - Type: Cracking
> - Niveau: Easy

Bienvenue dans ce write-up de `cracking` de la plateforme Root-Me !
Aujourd'hui on s'attaque au challenge `ELF x86 - 0 protection`.

Ce dernier est assez intéressant car il va nous demander d'explorer un peu plus les bases de l'assembleur et de la mémoire.

## Reconnaissance
Comme d'habitude on va passer un coup de `exiftool` pour comprendre la nature de notre binaire.

**exiftool sur notre binaire**
```bash
exiftool ch2.bin
```
Et jusque là, rien de bien surprenant, c'est bien un exécutable Linux (`ELF`) 32 bits.

```bash
ExifTool Version Number         : 12.57
File Name                       : ch2.bin
Directory                       : .
File Size                       : 580 kB
File Modification Date/Time     : 2026:05:02 16:40:12+02:00
File Access Date/Time           : 2026:05:03 12:52:39+02:00
File Inode Change Date/Time     : 2026:05:02 16:40:12+02:00
File Permissions                : -rwxrwxrwx
File Type                       : ELF executable
File Type Extension             :
MIME Type                       : application/octet-stream
CPU Architecture                : 32 bit
CPU Byte Order                  : Little endian
Object File Type                : Executable file
CPU Type                        : i386
```

Lançons-le pour voir ce qu'il retourne ! Les chaînes de caractères qu'il nous affiche peuvent être précieuses lorsque nous ferons de l'analyse statique plus poussée.

### Analyse dynamique
Bon, cette fois-ci le binaire nous demande d'entrer un `username`.
![Execution de ch2.bin](/assets/img/rootme/cracking/ch2/first_run.png)
_Exécution de ch2.bin_

J'ai inséré le username `Test`, et impossible d'aller plus loin si nous n'avons pas le bon username.

Il en va de même avec des usernames génériques comme :
- root
- admin
- toor

Donc il va nous falloir trouver l'username en creusant un peu !

### Analyse statique (strings)
Nous allons utiliser la commande `strings` pour extraire toutes les chaînes de caractères de notre programme.

Peut-être notre nom d'utilisateur y figure en clair !

**Extraction des chaînes de caractères avec strings**
```bash
strings ch2.bin
```

Et là on a un petit souci... En effet la commande nous retourne pas moins de `3762` lignes.

![Output strings](/assets/img/rootme/cracking/ch2/output_strings.png)
_Output strings_

Bon, comment faire ? Et bien nous allons trier tout ça.
Dans un premier temps nous allons mettre la sortie de la commande dans un fichier, ça nous permettra de faire des recherches plus pratiques.

**Enregistrement de la sortie dans output_ch2.txt**
```bash
strings ch2.bin > output_ch2.txt
```

Maintenant nous pouvons l'ouvrir avec `nano` ou `vi` (selon vos préférences) !

**Ouverture de output_ch2.txt avec nano**
![Ouverture de output_ch2.txt avec nano](/assets/img/rootme/cracking/ch2/nano_ch2.png)
_Ouverture de output_ch2.txt avec nano_

Oui, je vous l'accorde, comme ça nous n'allons rien en tirer. Que va-t-on chercher alors ?

Vous vous souvenez des chaînes de caractères affichées lors du démarrage du programme ? Comme `username:` par exemple.

![Execution de ch2.bin](/assets/img/rootme/cracking/ch2/first_run.png)
_Exécution de ch2.bin_

Et bien nous allons les chercher dans notre fichier ! Peut-être que le username ne se trouve pas loin !

Et ça fonctionne ! Nous avons des informations pertinentes à proximité de notre chaîne de caractères `username:`.

![Username trouvé dans output](/assets/img/rootme/cracking/ch2/strings_john.png)
_Username trouvé dans output_

Si nous regardons plus attentivement, nous pouvons voir quelques chose qui s'apparente à un `nom` (john) et peut-être un mot de passe (the ripper).

#### Exploitation
Nous pouvons les entrer pour valider le challenge.

![Validation du challenge](/assets/img/rootme/cracking/ch2/strings_win.png)
_Validation du challenge_

Encore une fois, nous allons essayer d'aller plus en profondeur et de tenter de résoudre ce challenge avec `ghidra` pour nous familiariser avec l'assembleur.

### Analyse statique (Ghidra)
Comme d'habitude :
- On lance Ghidra
- On importe notre exécutable
- On admire le magnifique code assembleur

![Première ouverture de ch2.bin sur Ghidra](/assets/img/rootme/cracking/ch2/first_run_ghidra.png)
_Première ouverture de ch2.bin sur Ghidra_

#### Découverte de la fonction main
Si vous avez lu le premier write-up, vous vous souvenez que la première chose à faire lorsque nous ouvrons un programme dans Ghidra est de trouver sa fonction `main`.

C'est dans cette fonction que nous trouverons toute la logique du programme, et il sera capital de partir de là pour pouvoir tracer le flux d'exécution.

Pour cela rien de plus simple, dans notre menu `Symbol Tree` à gauche, nous n'avons qu'à taper le nom de la fonction, donc `main`.

Cette dernière apparaîtra sous le dossier `functions`.
![Recherche de main](/assets/img/rootme/cracking/ch2/main_filter.png)
_Recherche de la fonction main_

Nous pouvons aller directement à cette dernière en `double cliquant` dessus.

![Accès à main](/assets/img/rootme/cracking/ch2/main_access.png)
_Accès à main_


Nous pouvons commencer notre analyse !

#### Analyse des instructions
Allez, on va commencer par décomposer les premières instructions de notre fonction `main`.

##### Prologue du programme
![Prologue](/assets/img/rootme/cracking/ch2/prologue.png)
_Prologue_

Le prologue, c'est le début d'une fonction en assembleur.
Il sert à mettre en place un "cadre propre" pour exécuter la fonction.

Nous pouvons l'identifier avec cette suite d'instructions :
```asm
lea ecx, [esp+4]
and esp, 0xfffffff0
push [ecx-4]
push ebp
mov ebp, esp
push ecx
sub esp, 0x24
```

C'est bien ça, nous donne le point de départ. Ce qui nous intéresse c'est la suite !

##### Écriture dans une variable locale

Vous voyez ces instructions ?
![Définition des variables locales](/assets/img/rootme/cracking/ch2/def_local_vars.png)
_Définition des variables locales_

C'est un pattern d'initialisation des variables locales : on réserve de l'espace sur la pile avec `sub esp, 0x24`, puis on initialise une variable locale en écrivant une valeur dans une case de cet espace via `mov [ebp + local_14], DAT_080a6b19`.

Ici, `DAT_080a6b19` est utilisé comme une **valeur immédiate** : l'instruction ne copie pas la donnée elle-même, elle copie **l'adresse** où se trouve cette donnée (un pointeur vers la chaîne stockée dans `.rodata`). Donc si on veut savoir quel est le username, il faut qu'on se rende à l'adresse `DAT_080a6b19` !

Pour ça nous avons juste à `double cliquer` sur `DAT_080a6b19`.

#### Récupération du username
Une fois à l'adresse `DAT_080a6b19`, nous pouvons voir notre username `john` !

![Username en variable locale](/assets/img/rootme/cracking/ch2/john_local_var.png)
_Username en variable locale_

> Mais pourquoi john est stocké sous forme verticale ?

Et bien tout simplement car en `C`, une string ressemble à ça en mémoire :
```bash
0x080a6b19 → 'j'
0x080a6b1a → 'o'
0x080a6b1b → 'h'
0x080a6b1c → 'n'
0x080a6b1d → '\0'
```

La chaîne continue jusqu'au caractère nul `\0`, que l'on peut voir sur la capture sous la forme `00h`.

Et voilà, nous avons pu récupérer notre username directement en analyse statique avec Ghidra !

> Le mot de passe lui est facilement observable en dessous, donc je ne le détaille pas plus ici !

Allez, vu qu'on est motivés, nous allons essayer de résoudre ce challenge avec une analyse dynamique, donc avec un `debugger` !

### Analyse dynamique (GDB via Ghidra)
Allez, on commence par importer notre fichier `ch2.bin` dans le `debugger` de Ghidra.

![Importation dans le debugger](/assets/img/rootme/cracking/ch2/debugger_import.png)
_Importation dans le debugger_

Vous vous souvenez de cette instruction ?
![Instruction variable locale](/assets/img/rootme/cracking/ch2/dynamic_local_var.png)
_Instruction variable locale_

C'est cette instruction qui permet de stocker un pointeur vers la variable locale.
On va mettre un `breakpoint` pile sur cette ligne.

> Un **breakpoint** est une instruction placée dans un débogueur qui suspend l'exécution d'un programme à un emplacement précis (adresse mémoire, ligne de code, entrée de fonction), permettant à l'analyste d'inspecter l'état du processus à cet instant : valeurs des registres, contenu de la stack, variables locales, etc. Une fois le programme suspendu, l'exécution peut reprendre normalement ou avancer instruction par instruction.

#### Breakpoint

Grâce à ce `breakpoint` placé à cet endroit, nous allons pouvoir interroger la zone mémoire pour y lire la valeur stockée !

Pour mettre le breakpoint :
- Clic gauche sur la ligne
- Appuyer sur la touche `k`
- Valider avec `OK`

![Insertion du breakpoint](/assets/img/rootme/cracking/ch2/set_breakpoint.png)
_Insertion du breakpoint_

Nous devrions voir notre `breakpoint` dans le menu `breakpoint` en haut à droite.

![Menu breakpoint](/assets/img/rootme/cracking/ch2/breakpoint_menu.png)
_Menu breakpoint_

Maintenant il faut l'activer en cliquant sur le petit rond `gris`, celui-ci devrait ensuite être `activé`.

![Activation du breakpoint](/assets/img/rootme/cracking/ch2/enable_breakpoint.png)
_Activation du breakpoint_


#### Lancement de GDB

Puis nous pouvons lancer le debugger dans :
- Debugger
- Configure and Launch ch2.bin
- gdb

Puis on valide avec `Launch`.

![Lancement de GDB](/assets/img/rootme/cracking/ch2/lunch_debugger.png)
_Lancement de GDB_

Notre `Ghidra` devrait ressembler à ça.

![Debugger Ghidra](/assets/img/rootme/cracking/ch2/debugger_ghidra.png)
_Ghidra en mode debugger_

Et on va activer de nouveau notre breakpoint en cliquant dessus, ce dernier doit être maintenant `bleu`.

![Réactivation du breakpoint](/assets/img/rootme/cracking/ch2/renabled_bp.png)

Dans notre terminal `gdb` nous devrions avoir la confirmation de l'activation du breakpoint.


#### Debug !

Maintenant nous allons lancer notre programme avec `gdb` dans le terminal via la commande `run`.

Nous allons avoir le message suivant :
```plaintext
The program being debugged has been started already.
Start it from the beginning? (y or n)
``` 
On va répondre `y`.

![Lancement du programme avec GDB](/assets/img/rootme/cracking/ch2/run_gdb.png)

Nous pouvons voir que le programme s'est arrêté à l'endroit où nous avons placé le `breakpoint` !

![Programme en pause au breakpoint](/assets/img/rootme/cracking/ch2/paused.png)
_Programme en pause au breakpoint_

#### Lecture de la variable
On va revenir sur notre instruction qui définit la variable locale :
```asm
mov [ebp + local_14], DAT_080a6b19
```

Cette instruction écrit dans la variable locale `[ebp + local_14]` l'adresse où se trouve la donnée `DAT_080a6b19`. Autrement dit, elle ne copie pas la donnée elle-même, elle copie **l'adresse où elle se trouve** en mémoire (c'est un pointeur).

On a donc juste à interroger `DAT_080a6b19` qui correspond à l'adresse `0x80a6b19`.

Pour ce faire on exécute la commande suivante dans `gdb` :
```asm
x/s 0x80a6b19
```


> La commande x/s dans GDB
> - x = examine (examiner la mémoire)
> - La syntaxe c'est :
>   - x / [nombre] [format] [adresse]
> - x/s = string, lit jusqu'au `\0`

Ce qui nous donne la valeur `john` !

![John found !](/assets/img/rootme/cracking/ch2/john_gdb.png)
_Lecture de john à l'adresse de la variable_

#### Lecture de la valeur en mémoire
Là nous avons cherché directement `0x80a6b19`, c'est-à-dire l'adresse de la chaîne stockée dans `.rodata`. Si nous voulons lire la valeur du **pointeur** stocké dans la variable locale `[ebp-0xc]` (c'est-à-dire l'adresse elle-même, pas la chaîne), nous allons relancer le programme jusqu'au breakpoint puis interroger la valeur de `EBP`.

Parce que `[ebp-0xc]` est une adresse relative, on ne peut pas la lire directement sans connaître la valeur de EBP.

Pour trouver l'adresse absolue sur la stack il faut :
```bash
adresse absolue = valeur de EBP - 0xc
```

Donc on relance le programme avec `run`.

![Relancement du programme](/assets/img/rootme/cracking/ch2/new_run.png)
_Relancement du programme_

Ensuite nous allons passer à l'instruction suivante avec la commande :
```bash
(gdb) si
```
pour être sûr que la variable soit chargée.

Nous pouvons voir que nous sommes passés à l'instruction suivante :

![Instruction suivante](/assets/img/rootme/cracking/ch2/next_instruction.png)
_Instruction suivante_

Et on va revenir sur la première instruction brute :
```asm
MOV dword ptr [EBP + -0xc], 0x80a6b19
```

Et calculer où se trouve `[EBP - 0xc]` en mémoire. Pour ça il faut connaître la valeur de `EBP` :

```
(gdb) info registers ebp
```

![Récupération EBP](/assets/img/rootme/cracking/ch2/get_ebp.png)
_Récupération EBP_

`EBP` pointe vers la **base de la stack frame** courante. Toutes les variables locales sont stockées **en dessous** de EBP (adresses négatives).

Donc quand on voit `[EBP - 0xc]` dans le code asm, ça veut dire :

```
EBP          = 0xff9d9508   ← base de la frame
EBP - 0xc    = 0xff9d94fc   ← adresse de notre variable locale
```

On peut faire la soustraction à l'aide de Python :
```python
python3 -c "print(hex(0xff9d9508 - 0xc))"
```
Ce qui nous donne :
```
0xff9d94fc
```

On veut lire la valeur stockée dans notre variable locale `[ebp-0xc]` à l'adresse `0xff9d94fc`. Une adresse fait 4 octets sur un système 32 bits.

Nous allons utiliser `x/wx` pour récupérer cette valeur :

> x/wx = examiner 1 word (4 octets) en hexadécimal à une adresse donnée.

Et on récupère bien la valeur `0x80a6b19` qui est le pointeur vers `john`.

![Récupération [ebp-0xc]](/assets/img/rootme/cracking/ch2/final.png)
_Récupération [ebp-0xc]_
