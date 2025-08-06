+++
date = '2025-07-21T07:43:35+02:00'
#lastmod = '2025-07-22T07:43:35+02:00'
draft = true
title = "uProbe : l'observabilité pour tous les développeurs"
series = ["Apprenons uProbe avec eBPF et Aya", "Apprenons un autre programme eBPF"]
tags = ["Rust", "eBPF", "aya", "intro"]
summary = "Nous allons voir les bases des programmes eBPF de type uProbe et uRetProbes avec Aya"
showTableOfContents = true
series_order = 1
+++

Je débute la programmation en eBPF avec Aya. L’idée de cette série d’articles est d'apprendre un nouveau type de programme eBPF et de l'expérimenter avec le framework Rust Aya.

Aujourd'hui, nous allons nous plonger dans les **uProbes** et les **uRetProbes** : des programmes eBPF qui sondent les fonctions de l'espace utilisateur sans laisser de trace.

Vous allez voir que cela peut être très intéressant pour du profilage, du débug ou de la rétro-ingénierie.

{{< alert "lightbulb" >}}Si tu ne connais pas eBPF, je te conseille de lire les deux premières parties de ma série [S’initier à eBPF avec Aya](https://medium.com/@littel.jo/sinitier-%C3%A0-ebpf-avec-aya-c9d570560261). Cela couvre les bases et t'aidera pour la suite de l'article.{{< /alert >}}

---

## Qu'est-ce qu'une u•Ret•Probe ?

En anglais, une *probe* peut se traduire par une sonde pour examiner ou explorer quelque chose. En eBPF, il y en a de plusieurs types : kProbe, kRetProbe, uProbe, uRetProbe et USDT.

### kProbe

Si tu consultes la [documentation d'eBPF](https://docs.ebpf.io/linux/program-type/), il n'y a pas de section consacrée aux programmes de type uProbe ou uRetProbe. Mais il y en a une dédiée au [kProbe](https://docs.ebpf.io/linux/program-type/BPF_PROG_TYPE_KPROBE/) :

[![Kprobe documentation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p04bcqi8orzc9pwpvvf1.png)](https://docs.ebpf.io/linux/program-type/BPF_PROG_TYPE_KPROBE/)

La kProbe a pour but d'observer des fonctions du kernel Linux. Elle est la probe parente. Toutes les autres probes sont en fait le même type de programme `BPF_PROG_TYPE_KPROBE` mais c'est juste le point d'attache qui va déterminer comment le programme est exécuté.

### kRetProbe
 
La kRetProbe est dédiée à l'observation du retour des fonctions du kernel Linux.

### uProbe

Contrairement aux **k**Probes qui sont dédiées à observer les fonctions du **k**ernel Linux, les **u**Probes sont dédiées aux fonctions de l'espace utilisateur : *User Probes*. Par exemple, on pourrait s'en servir pour compter le nombre d'appels aux fonctions `malloc` et `free` dans un programme C. uProbe permet également de récupérer le contenu des arguments de chaque fonction appelée et de débugger un programme dont tu n'as pas le code source. Ça peut donc être un bel outil de rétro-ingénierie (*reverse engineering*).

{{< alert >}}
Ça peut paraître paradoxal de vouloir tracer un code utilisateur depuis l'espace noyau. Cependant cela a le mérite d'être non intrusif car il n'y a pas besoin de modifier le programme. {{< /alert >}}

### uRetProbe

u**Ret**Probe a pour but d'étudier le **ret**our de la fonction cible de l'espace utilisateur : *User Return Probe*.
On peut donc découvrir la valeur que retourne la fonction qui permet de debugger ou d'observer le comportement final de la fonction. Mais il y a un autre intérêt : en combinant les temps de l'uProbe et de l'uRetProbe, on peut récupérer la durée que met une fonction à s'exécuter assez facilement. Il est ainsi possible de profiler chaque fonction de son programme.

{{< alert "lightbulb" >}}On pourrait par exemple l'utiliser pour des requêtes SQL où on identifierait les requêtes les plus longues.{{< /alert >}}


### USDT

USDT veut dire *User Statically-Defined Tracing*. Elle est dérivée de l'uprobe.
Comme la uProbe, elle est dédiée aux fonctions de l'espace utlisateur mais il faut rajouter dans le code des sondes usdt pour les utiliser.

{{< alert >}}
Au moment de l'écriture, le framework Aya ne gère pas les programmes de type USDT.
{{< /alert >}}

Nous allons donc nous consacrer pour la suite de l'article aux uProbes et uRetProbes.

---

## Origin story

### uTrace l'ancètre

Vouloir tracer des fonctions de l'espace utilisateur depuis le kernel linux ne date pas de l'introduction d'eBPF. Par exemple, une (première ?) tentative est apparue en 2007 avec les **utraces** :

[![Introducing utrace](screenshot/utrace.png)](https://lwn.net/Articles/224772/)

Mais elles n'ont jamais été incluses dans le code principal à cause d'opposition de certains mainteneurs.

### Habemus uProbe

En 2012, le consensus a fini par arriver et les uProbes ont été introduites lors de la version 3.5 du noyau Linux :

[![Introducing uprobe](screenshot/uprobe-history.png)](https://lwn.net/Articles/499190/)

Elles ont ensuite été améliorées avec la version 3.14 (sortie en 2014) :

[![Patching uprobe](screenshot/uprobe-history-2.png)](https://lwn.net/Articles/577142/)

### uProbe avec eBPF

Dans la documentation eBPF de kProbe, kProbe est apparue en 2015 dans la version 4.1. En voici le commit :

[![Patching kprobe eBPF](screenshot/uprobe-history-3.png)](https://github.com/torvalds/linux/commit/2541517c32be2531e0da59dfd7efc1ce844644f5)

Comme une uProbe est une kProbe avec un point d'attache différent, on pouvait commencer à développer des uProbe avec eBPF à partir du 2 avril 2015.

Cependant il fallait encore attendre que les frameworks eBPF de l'époque puissent le gérer.

Ainsi on pouvait déjà l'utiliser [en 2016 avec BCC](https://github.com/iovisor/bcc/commit/948cefe14ba1f18aa49732c5f2f65837c79572be) comme l'atteste le tutoriel de Brendan Gregg : 

[![uprobe bcc eBPF tutorial](screenshot/uprobe-history-4.png)](https://www.brendangregg.com/blog/2016-02-08/linux-ebpf-bcc-uprobes.html)

On peut voir également son *issue* sur github datant d'octobre 2015 :
[![uprobe bcc issue](screenshot/uprobe-issue.png)](https://github.com/iovisor/bcc/issues/273)


---

## Bonus Track

Pour finir la présentation, voici quelques liens bien sympathiques :

* Utilisation de uprobe **sans eBPF** par Brendan Gregg en 2015 :

[![Brendan Gregg blog](screenshot/uprobe-brendan.png)](https://www.brendangregg.com/blog/2015-06-28/linux-ftrace-uprobe.html)

* L'excellent blog de **Julia Evans** sur tous les systèmes de tracing sous Linux :

[![Julia Evans blog](screenshot/linux-tracing.png)](https://jvns.ca/blog/2017/07/05/linux-tracing-systems/)

* si les uProbes ne vous convient pas peut-être que le **bpftime** d'Eunomia peut vous intéresser :

[![bpftime: Userspace eBPF runtime for Observability, Network & General extensions Framework](screenshot/bpftime.png)](https://eunomia.dev/en/bpftime/)

Maintenant qu'on a présenté uProbe et uRetProbe, voyons comment débuter son développement avec Aya.

---

## Comment trouver les hooks ?

Quand on démarre le développement d'un nouveau programme eBPF, la première difficulté est de réussir à le démarrer. Pour cela, il a besoin d'un événement déclencheur (event-driven). Dans cet épisode, cet événement sera le passage d'une uProbe ou d'une uRetProbe dans le noyau Linux.

Aya nous facilite la tâche. Quand on lance la commande :
```Bash
cargo generate https://github.com/aya-rs/aya-template
```

Tu devras répondre à deux questions importantes qui permettront de définir le hook :

```
🤷   Target to attach the (u|uret)probe? (e.g libc):
🤷   Function name to attach the (u|uret)probe? (e.g getaddrinfo):
```

Essayons de répondre à ces questions maintenant.

### Cible pour attacher l'u•Ret•Probe

Pour la première question tu dois donner le nom d'une bibliothèque (comme la libc) ou le nom d'un binaire (dans un chemin absolu). La question aurait pu être posée autrement : quel fichier tu veux debugger ou tracer ?

Il faut voir ça comme un filtre.
* Si tu choisis `libc`, tu auras toutes les chances que le programme eBPF tourne à chaque fois qu'un programme qui utilise la bibliothèque libc est démarré
* Si tu choisis un binaire, il ne fonctionnera que si le binaire est exécuté.

### Nom de la fonction pour attacher l'u•Ret•Probe

La seconde question demande la fonction du binaire ou de la bibliothèque que tu veux débugger.

Par exemple :
* si tu écris un programme en C, tu peux mettre le nom d'une fonction du programme.
Ainsi à chaque fois que la fonction est appelée par ce programme, le programme eBPF sera lancé.
* si tu choisis quelque chose de beaucoup moins précis comme la bibliothèque `libc` et si tu choisis le syscall `execve` le programme eBPF se lancera à chaque fois qu'un programme qui utilise la libc s'exécute, ça arrivera beaucoup plus.

---

La présentation des programmes eBPF de type uProbe et uRetProbe est terminée.
Passons maintenant à la pratique : nous allons créer un petit programme et nous allons le faire réagir avec un programme eBPF de type uProbe avec Aya.
