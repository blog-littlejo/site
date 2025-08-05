+++
date = '2025-07-21T07:43:35+02:00'
#lastmod = '2025-07-22T07:43:35+02:00'
draft = true
title = "uProbe : l'observabilité pour tous les développeurs"
series = ["Apprenons uProbe avec eBPF et Aya", "Apprenons un autre programme eBPF"]
tags = ["Rust", "eBPF", "aya", "intro"]
summary = "Nous allons voir les bases des programmes eBPF de type uProbe et uRetProbes avec Aya"
series_order = 1
+++

Je débute la programmation en eBPF avec Aya. L’idée de cette série d’articles est d'apprendre un nouveau type de programme eBPF et de l'expérimenter avec le framework Rust Aya.

Aujourd'hui nous allons nous plonger dans les **uProbes** et les **uRetProbes** : des programmes eBPF qui sondent les fonctions de l'espace utilisateur.

Si tu ne connais pas eBPF, je te conseille de lire les deux premières parties de ma série [S’initier à eBPF avec Aya](https://medium.com/@littel.jo/sinitier-%C3%A0-ebpf-avec-aya-c9d570560261). Cela couvre les bases et t'aidera pour la suite de l'article.

---

# Qu'est-ce qu'un u•Ret•Probe ?

En anglais, une *probe* est une sonde pour examiner ou explorer quelque chose.

## Dans la famille des probes, je demande

### kProbe

Si tu consultes la [documentation d'eBPF](https://docs.ebpf.io/linux/program-type/), il n'y a pas de section consacrée aux programmes de type uProbe ou uRetProbe. Mais il y a une dédiée au [kProbe](https://docs.ebpf.io/linux/program-type/BPF_PROG_TYPE_KPROBE/) :

![Kprobe documentation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p04bcqi8orzc9pwpvvf1.png)

### uProbe

Contrairement aux **k**Probes qui sont dédiées à observer les fonctions du **k**ernel Linux, les **u**Probes sont dédiés aux fonctions de l'espace utilisateur : *User Probes*.

## uProbe et uRetProbe : les inséparables ?

u**Ret**Probe a pour but d'étudier le **ret**our de la fonction cible : *User Return Probes*. Ainsi en ayant les temps du uProbe et du uRetProbe, on peut avoir la durée que met une fonction à s'exécuter assez facilement. Il est ainsi possible de tracer chaque fonction de son programme. On pourrait par exemple l'utiliser pour des requêtes SQL où on identifierait les requêtes les plus longues.

## Autres intérêts des u•Ret•Probes

uProbe permet également de récupérer le contenu des arguments de chaque fonction appelée et débugger un programme dont tu n'as pas le code source. Ça peut donc être un bel outil de reverse engineering.

Pour finir la présentation, ça peut paraître paradoxal de vouloir tracer un code utilisateur depuis l'espace noyau. Cependant cela a le mérite d'être non intrusif car il n'y a pas besoin de modifier le programme.


---

# Origin story

## uTrace l'ancètre

Vouloir tracer des fonctions de l'espace utilisateur depuis le kernel linux ne date pas de l'introduction d'eBPF. Par exemple, une (première ?) tentative est apparue en 2007 avec les utraces :

[![Introducing utrace](screenshot/utrace.png)](https://lwn.net/Articles/224772/)

Mais ils n'ont jamais été inclus dans le code principal à cause d'opposition de certains mainteneurs.

## Habemus uProbe

En 2012, le consensus a fini par arriver et les uProbes ont été introduites lors la version 3.5 du noyau Linux :

[![Introducing uprobe](screenshot/uprobe-history.png)](https://lwn.net/Articles/499190/)

Ils ont ensuite été améliorés avec la version 3.14 (sortie en 2014) :

[![Patching uprobe](screenshot/uprobe-history-2.png)](https://lwn.net/Articles/577142/)

## uProbe avec eBPF : une naissance silencieuse

Je n'ai pas trouvé la date exacte de l'apparition des uprobes avec eBPF. Dans la documentation de kprobe, kprobe est apparu dans la version 4.1 (sortie en 2015). En voici le commit :

[![Patching kprobe eBPF](screenshot/uprobe-history-3.png)](https://github.com/torvalds/linux/commit/2541517c32be2531e0da59dfd7efc1ce844644f5)

Mais il n'y a pas de trace pour la possibilité d'utiliser uprobe avec eBPF (si quelqu'un l'a trouvé, je n'hésiterai pas à modifier l'article).
Cependant, on peut déjà l'utiliser depuis 2016 comme le montre le tutoriel de Brendan Gregg : 

[![uprobe bcc eBPF tutorial](screenshot/uprobe-history-4.png)](https://www.brendangregg.com/blog/2016-02-08/linux-ebpf-bcc-uprobes.html)

Il suffit juste tourner sur une distribution Linux moderne pour expérimenter cela.

Maintenant qu'on a présenté uProbe et uRetProbe, voyons comment les manipuler un peu plus concrètement avec Aya.

---

# Comment trouver les hooks ?

Quand on démarre le développement d'un nouveau programme eBPF, la première difficulté est de réussir à le démarrer. Pour cela, il a besoin d'un événement déclencheur (event-driven). Dans cet épisode, cet événement sera le passage d'un uProbe et d'un uRetProbe dans le noyau Linux.

Aya nous facilite la tâche quand on lance la commande :
```Bash
cargo generate https://github.com/aya-rs/aya-template
```

Tu devras répondre à deux questions importantes qui permettront de définir le hook :

```
🤷   Target to attach the (u|uret)probe? (e.g libc):
🤷   Function name to attach the (u|uret)probe? (e.g getaddrinfo):
```

La première bonne nouvelle c'est que les questions sont les mêmes pour uProbe et uRetProbe.

Essayons de répondre à ces questions maintenant.

## Cible pour attacher le uProbe/uRetProbe

Pour la première question tu dois donner le nom d'une bibliothèque (comme la libc) ou le nom d'un binaire (dans un chemin absolu). La question aurait pu être posée autrement : quel fichier tu veux debugger ou tracer ?

Il faut voir ça comme un filtre.
* Si tu choisis `libc`, tu auras toutes les chances que le programme eBPF tourne à chaque fois qu'un programme qui utilise la bibliothèque libc est démarré
* Si tu choisis un binaire, il ne fonctionnera que si le binaire est exécuté.

## Nom de la fonction pour attacher le uProbe/uRetProbe

La seconde question demande la fonction du binaire ou de la bibliothèque que tu veux débugger. Par exemple si tu écris un programme, tu peux mettre le nom d'une fonction du programme. Ainsi à chaque fois que la fonction est appelée, le programme eBPF sera lancé. Si tu choisis quelque chose de beaucoup moins précis comme la bibliothèque libc et si tu choisi le syscall execve le programme eBPF se lancera à chaque fois qu'un programme qui utilise la libc s'exécute. Ça arrivera beaucoup plus.

---

L'introduction aux programmes eBPF de type uProbe et uRetProbe est terminée. Dans les prochains épisodes, nous allons expérimenter tout cela avec Aya.
