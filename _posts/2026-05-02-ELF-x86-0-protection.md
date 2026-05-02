---
title: "Root-Me Cracking - ELF x86 - 0 protection"
date: 2026-05-01
categories: [cracking, rootme]
tags: [rootme, cracking, reverse, easy, linux]
image:
  path: /assets/img/rootme/rootme_card_template.png
---

## Contexte
> - Nom: ELF x86 - 0 protection
> - Type: Cracking
> - Niveau: Easy

Bienvenue dans notre premier write-up de `cracking` de la plateforme Root-Me !
Nous allons commencer tout doucement avec le challenge `ELF x86 - 0 protection`.

Comme son nom l'indique il s'agit d'un binaire `ELF` à cracker. De plus, la plateforme nous donne l'indice suivant sur la page du challenge :
> Premier challenge de cracking, programme rédigé en C avec vi et compilé avec GCC32.

Nous savons donc qu'il s'agit d'un programme compilé en `32 bit`, ce qui peut rendre un challenge de cracking plus simple, surtout pour des raisons de lisibilité et de complexité réduite du binaire.

## Reconnaissance
Même si le challenge est relativement simple en soi, sa faible complexité nous permettra de prendre en main certains outils et d'adopter quelques automatismes.

### exiftool
Nous allons commencer avec la commande `exiftool`, qui va nous permettre de récupérer des informations pertinentes sur notre fichier.

**lancement de exiftool**
```bash
exiftool ch1.bin
```

Effectivement nous avons quelques informations pertinentes.

**output exiftool**
```bash
ExifTool Version Number         : 12.57
File Name                       : ch1.bin
Directory                       : .
File Size                       : 7.3 kB
File Modification Date/Time     : 2026:05:02 12:32:30+02:00
File Access Date/Time           : 2026:05:02 14:22:38+02:00
File Inode Change Date/Time     : 2026:05:02 12:32:30+02:00
File Permissions                : -rwxrwxrwx
File Type                       : ELF executable
File Type Extension             :
MIME Type                       : application/octet-stream
CPU Architecture                : 32 bit
CPU Byte Order                  : Little endian
Object File Type                : Executable file
CPU Type                        : i386
```

Nous avons bien la confirmation que nous avons affaire à un exécutable `Linux` (ELF) en 32 bit.

Lançons-le pour voir ce qu'il retourne !

### Analyse dynamique
Nous allons exécuter ce binaire pour voir :
- ce qu'il fait
- ce qu'il attend comme entrée utilisateur
- les différentes chaînes de caractères présentes dans le programme

**Exécution de ch1.bin**
```bash
./ch1.bin
```

Apparemment, il s'agit d'un programme qui nous demande un mot de passe pour pouvoir récupérer le flag.
![Execution de ch1.bin](/assets/img/rootme/cracking/ch1/exec_file.png)
_Exécution de ch1.bin_

Il nous faut donc trouver ce mot de passe. Mais comment ?
Et si nous pouvions dans un premier temps extraire toutes les chaînes de caractères d'un fichier exécutable ? Peut-être que ce dernier se trouve `en clair` à l'intérieur ?

C'est ce que nous allons vérifier avec la commande `strings` !

### Analyse statique (strings)
La commande `strings` sert à extraire et afficher les chaînes de caractères lisibles (ASCII/UTF-8) contenues dans un fichier binaire, utile en analyse forensique et en reverse engineering pour repérer rapidement des indices (URLs, mots de passe, messages, etc.).

**Lancement de strings sur ch1.bin**
```bash
strings ch1.bin
```

Nous pouvons voir dans la sortie une suite de caractères qui s'apparente à un mot de passe : il s'agit de la chaîne `123456789`.

![Mot de passe dans la commande strings](/assets/img/rootme/cracking/ch1/strings_output.png)
_Mot de passe dans la commande strings_


#### Exploitation

Si nous entrons ce mot de passe dans `ch1.bin` nous pouvons voir que le challenge est validé !

**Entrée du mot de passe**
```bash
echo "123456789" | ./ch1.bin
```

![Challenge validé](/assets/img/rootme/cracking/ch1/strings_win.png)
_Challenge validé_

On s'arrête là ? Bien sûr que non ! Il y a d'autres moyens de réussir le challenge, nous allons les explorer afin de toucher du doigt les autres méthodes qui vont nous aider pour les prochains challenges !

### Analyse statique (Ghidra)
Même si ce challenge pouvait être résolu directement avec `strings`, il est aussi possible d'utiliser un outil de reverse engineering comme **Ghidra**.

Ghidra permet de :
-   désassembler le binaire
-   obtenir une version pseudo-C du programme
-   comprendre la logique des conditions (comparaisons, `jmp`, etc.)

Dans ce cas précis, on va essayer de voir les instructions appelées avec Ghidra pour s'habituer à lire de l'assembleur et à reconnaître des patterns !

Si nous importons notre fichier `ch1.bin` dans `Ghidra` nous allons nous retrouver avec ça :
![Ghidra ch1.bin](/assets/img/rootme/cracking/ch1/ghidra_ch1.png)
_Ghidra ch1.bin_

Alors oui, ça peut être terrifiant, mais à force on s'habitue !

#### Découverte de la fonction main
Avant d'aller plus loin, nous allons récupérer la fonction `main` du programme. C'est en effet dans cette fonction que se situe la logique principale. L'analyser est essentiel pour comprendre le comportement du binaire et suivre son flux d'exécution.

##### Fonction main dans la liste des fonctions

Nous pouvons trouver la fonction nommée directement `main` dans la liste des fonctions sur Ghidra.

![Fonction main dans la liste des fonctions](/assets/img/rootme/cracking/ch1/main_functions_ghidra_list.png)
_Fonction main dans la liste des fonctions_

Et si on double-clique dessus, nous sommes directement sur cette dernière !
![Détail de la fonction main](/assets/img/rootme/cracking/ch1/main_function_detail.png)
_Détail de la fonction main_


#### Analyse des instructions
Nous allons maintenant analyser la fonction `main` du binaire à l'aide de Ghidra afin de comprendre la logique de vérification du mot de passe.

![Instructions de la fonction main](/assets/img/rootme/cracking/ch1/main_function_instructions.png)
_Instructions de la fonction main_


##### Prologue
Le prologue de fonction - lignes 1 à 8 :
```asm
LEA   ECX, [ESP + 0x4]
AND   ESP, 0xfffffff0    ; alignement mémoire sur 16 octets
PUSH  dword ptr [ECX + local_res0]
PUSH  EBP               ; sauvegarde du frame pointer
MOV   EBP, ESP          ; nouveau frame
PUSH  ECX
SUB   ESP, 0x24         ; réservation de 0x24 (36) octets sur la pile
```

C'est le **prologue standard** d'une fonction en x86. À retenir :

-   `PUSH EBP / MOV EBP, ESP` → c'est **toujours** le début d'une fonction
-   `SUB ESP, 0x24` → on réserve de l'espace pour les **variables locales**

##### Le mot de passe en clair

```asm
080486ae    MOV    dword ptr [EBP + local_10], s_123456789_08048841 = "123456789"
```

Ghidra a identifié une chaîne de caractères et nous la montre directement : `"123456789"`.

Cette valeur est stockée dans une variable locale (`local_10`). C'est **le mot de passe attendu**, stocké en clair dans le binaire ! Comme on l'a déjà vu avec `strings`.

##### La comparaison
```asm
08048700    CALL    <EXTERNAL>::strcmp
08048705    TEST    EAX, EAX
08048707    JNZ     LAB_0804871e
```

C'est le cœur du challenge ! Voici ce qui se passe :

```
strcmp(input_utilisateur, "123456789")
```

`strcmp` retourne `0` si les deux chaînes sont **identiques**, autre chose sinon.

`TEST EAX, EAX` positionne le flag **ZF (Zero Flag)** selon la valeur de EAX :

-   Si `EAX == 0` → ZF=1 → le `JNZ` **ne saute pas** → mot de passe correct
-   Si `EAX != 0` → ZF=0 → le `JNZ` **saute** vers `LAB_0804871e` → mauvais mot de passe


> Mais du coup, si l'instruction `JNZ` n'était pas là, en cas de mauvais mot de passe on ne sauterait pas vers la fonction qui nous indique `mauvais mot de passe` ?

Et vous avez raison ! On va donc supprimer notre instruction, ou la rendre `null` pour que le flux de notre programme passe au travers !

![Instruction de saut JNZ](/assets/img/rootme/cracking/ch1/jnz_instruction.png)
_Instruction de saut JNZ_


#### Modification des instructions

Pour modifier notre instruction nous allons faire :
- Clic droit sur l'instruction
- Patch Instruction

Nous allons remplacer `JNZ` par `NOP` et supprimer l'adresse de destination puis valider avec `Entrée`.

![Modification de l'instruction JNZ par NOP](/assets/img/rootme/cracking/ch1/patch_instruction.png)
_Modification de l'instruction JNZ par NOP_

Maintenant il ne reste plus qu'à exporter notre binaire avec sa modification, donc une version `patchée`, en allant dans :
- File
- Export Program...

> Attention à bien choisir le format de fichier `original file`, ça sera plus simple.

![Exportation du fichier patché](/assets/img/rootme/cracking/ch1/patched_program.png)
_Exportation du fichier patchée_

Nous voyons bien notre fichier patchée !

```bash
[May 02, 2026 - 16:08:44 (CEST)] exegol-rootme /workspace # ls
ch1.bin  ch1_patched.bin
```

Maintenant si nous lançons notre programme et que nous rentrons n'importe quel mot de passe, nous bypassons la vérification et arrivons au bout du programme !

![Résultat fichier patché](/assets/img/rootme/cracking/ch1/patched_program_result.png)
_Résultat fichier patchée_
