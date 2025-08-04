+++
date = '2025-08-01T07:43:35+02:00'
draft = true
title = "Tracepoint : l'observabilité kernel pour tous"
series = ["Apprenons Tracepoint avec eBPF et Aya", "Apprenons un autre programme eBPF"]
tags = ["Rust", "eBPF", "aya", "intro"]
series_order = 1
+++

Je débute la programmation en eBPF avec Aya. L’idée de cette série d’articles est d'apprendre un nouveau type de programme eBPF à chaque épisode.

Aujourd'hui nous allons nous plonger dans les **Tracepoints** : des programmes eBPF qui permettent de tracer certaines fonctions du noyau Linux.

Si tu ne connais pas eBPF et Rust, je te conseille de lire les deux premières parties de ma série [S’initier à eBPF avec Aya](https://medium.com/@littel.jo/sinitier-%C3%A0-ebpf-avec-aya-c9d570560261). Cela couvre les bases et t'aidera pour la suite de l'article. En plus il parle déjà des programmes eBPF de type Tracepoint. Aujourd'hui, nous allons prendre un peu de recul sur les articles d'origine et voir de nouveaux concepts importants dans eBPF.

---

## Qu'est-ce qu'un Tracepoint ?

Un tracepoint est un endroit dans le code Linux où on a décidé de rajouter volontairement un petit code (voir les [recommandations](https://docs.kernel.org/trace/tracepoints.html)) permettant de récupérer certains paramètres. Grâce aux tracepoints, on peut par exemple vérifier qu'un disque dur est défaillant. 

Bien que les Tracepoints peuvent être utilisés indépendemment d'eBPF, leur association avec eBPF prend tout son sens quand on les combine avec d'autres programmes et maps eBPF.

### Différence avec un kProbe

Dans le chapitre précédent sur les uProbes nous avons parlé de kProbes qui peuvent sembler proches des Tracepoints. Pour que ça soit clair, mettons en lumière leurs différences.

Un kProbe permet d'observer toutes les fonctions du code Linux alors que le tracepoint le permet que si un développeur du noyau a décidé de rajouter ce petit code pour le Tracepoint. Par contre, si par malchance, le nom de la fonction ou ses arguments change, le code du kProbe ne sera plus compatible avec la nouvelle version du noyau Linux.
Ainsi les tracepoints sont plus stables qu'un kProbe mais moins complet. C'est similaire à une [API](https://docs.kernel.org/core-api/tracepoint.html).

![Documentation kernel sur le Tracepoint](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/r99wohve4xuoc0n4sfrj.png)

Maintenant qu'on a présenté les Tracepoints, voyons comment l'utiliser avec eBPF.

[![Tracepoint documentation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qsz9yt9rcjli6toquf0b.png)](https://docs.ebpf.io/linux/program-type/BPF_PROG_TYPE_TRACEPOINT/)

---

## Comment trouver les hooks ?

Quand on démarre le développement d'un nouveau programme eBPF, la première difficulté est de réussir à le démarrer. Pour cela, il a besoin d'un événément déclencheur (*event-driven*). Dans cet épisode, Cet événement sera le passage d'un Tracepoint dans le noyau Linux.

Si tu essaies de générer un programme Tracepoints avec Aya (et `cargo generate`), tu devras répondre à deux questions importantes :

```
🤷   Which tracepoint category? (e.g sched, net etc...):
🤷   Which tracepoint name? (e.g sched_switch, net_dev_queue):
```

Pour répondre à ces questions il faut lire le fichier :
`/sys/kernel/debug/tracing/available_events`

Ce fichier contient tous les tracepoints disponibles du noyau Linux :
```
cat /sys/kernel/debug/tracing/available_events | tail
bpf_test_run:bpf_test_finish
fib6:fib6_table_lookup
mptcp:subflow_check_data_avail
mptcp:ack_update_msk
mptcp:get_mapping_status
mptcp:mptcp_sendmsg_frag
mptcp:mptcp_subflow_get_send
maple_tree:ma_write
maple_tree:ma_read
maple_tree:ma_op
```

Ainsi le format est : `{categorie}:{nom}`

Il est donc facile de trouver un hook. Encore faut-il savoir à quoi il correspond vraiment.

### Quelles sont les différentes catégories possibles ?

J'ai compté 86 catégories. Ce serait rébarbatif de tous les énumérer.

Regardons les catégories qui ont le plus de tracepoints :

```
cat /sys/kernel/debug/tracing/available_events | awk -F: '{print $1}' | sort | uniq -c | sort -n | tail
     18 vmscan
     19 block
     21 jbd2
     24 power
     26 xen
     27 sched
     30 irq_vectors
     32 writeback
    113 ext4
    690 syscalls
```

On voit qu'il y a beaucoup de Tracepoints pour les fonctions syscalls. En fait, il y a deux Tracepoints par syscall : un pour l'entrée et un pour la sortie. Il y a également beaucoup de fonctions dédiées aux systèmes de fichiers ext4.

### Comment récupérer les différentes sondes d'un tracepoint

Maintenant que notre programme eBPF démarre avec un beau "hello world" à chaque fois qu'un tracepoint est parcouru, comment récupére-t-on les variables du tracepoint ?

Pour cela, il faut lire un autre fichier :
`/sys/kernel/debug/tracing/events/{categorie}/{name}/format`

Prenons un exemple simple, la récupération du tracepoint du syscall open à l'entrée de la fonction :

```
cat /sys/kernel/debug/tracing/events/syscalls/sys_enter_open/format
name: sys_enter_open
ID: 654
format:
        field:unsigned short common_type;       offset:0;       size:2; signed:0;
        field:unsigned char common_flags;       offset:2;       size:1; signed:0;
        field:unsigned char common_preempt_count;       offset:3;       size:1; signed:0;
        field:int common_pid;   offset:4;       size:4; signed:1;

        field:int __syscall_nr; offset:8;       size:4; signed:1;
        field:const char * filename;    offset:16;      size:8; signed:0;
        field:int flags;        offset:24;      size:8; signed:0;
        field:umode_t mode;     offset:32;      size:8; signed:0;

print fmt: "filename: 0x%08lx, flags: 0x%08lx, mode: 0x%08lx", ((unsigned long)(REC->filename)), ((unsigned long)(REC->flags)), ((unsigned long)(REC->mode))
```

Tous les tracepoints ont différentes sections :
* name : le nom
* id : l'identifiant
* format : le formatage
* print fmt : l'affichage.

La partie la plus intéressante pour eBPF est la section format. Les 4 premiers champs de cette section (common_type, common_flags, common_preempt_count et common_pid) sont communs à tous les Tracepoint mais ne sont pas utiles.

Il reste donc, pour ce tracepoint, 4 sondes :
* `__syscall_nr` : numéro du syscall (type: `int`). Il est accessible à l'offset 8.
* `filename` : nom du fichier qui est ouvert (type: `const char*`). Il est accessible à l'offset 16.
* `flags` : les flags passés à open (type: `int`). Il est accessible à l'offset 24.
* `mode` : le mode d'ouverture (type: `umode_t`). Il est accessible à l'offset 32.

On voit que chaque sonde peut avoir des types différents. Ils sont écrits en C. Une des difficulté est la traduction de ces types en Rust. Dans avons déjà vu comment résoudre cela pour `int` et `const char*` (un pointeur vers une chaine de caractères) dans [d'autres articles](https://medium.com/@littel.jo/sinitier-%C3%A0-ebpf-avec-aya-partie-2-fdaac12da841).

Nous allons maintenant voir des tracepoints qui ont des sondes avec des types plus compliqués. Comme cela, à la fin de l'article vous pourrez interpréter la plupart de toutes les sondes des tracepoints.
