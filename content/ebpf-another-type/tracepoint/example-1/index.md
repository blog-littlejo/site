+++
date = '2025-08-02T07:56:34+02:00'
draft = true
title = "Tracepoint : timer_start"
series = ["Apprenons Tracepoint avec eBPF et Aya"]
tags = ["Rust", "eBPF", "aya"]
series_order = 2
+++

## Exemple 1: `void *`

### Qu'est-ce que `void *` ?

Si tu as déjà regardé pas mal de Tracepoints, tu as dû voir souvent ce type là. Void veut dire vide. Donc `void *` pourrait traduire littéralement pointeur vide. Mais il serait plus pertinent de traduire pointeur générique :

![Pointeur générique : définition de wikipedia](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5ml7q1b9qs1252zxg061.png)

Ça peut donc être n'importe quel pointeur. On est bien avancé... Il faut donc chercher quel est le pointeur réel pour ensuite pouvoir le traduire en Rust. Comment trouver cela ? Il "suffit" de faire une recherche dans le code source de Linux (j'ai un peu sur-vendu le côté api).

Nous allons maintenant prendre un exemple simple où on doit manipuler un pointeur générique.

### Tracepoint `timer_start`

#### À quoi sert ce Tracepoint ?

Même si ce n'est vraiment l'objet de l'article, il est très utile de rechercher à quoi sert le Tracepoint pour pouvoir interpréter les résultats des différentes sondes.

Je ne suis pas vraiment expert en `timer`. Après une petite recherche j'ai trouvé ça : 

[![Définition d'un timer](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wyg3zycw5sq5rnjpv9h7.png)](https://ftp.traduc.org/doc-vf/gazette-linux/html/2004/103/lg103-G.html)

J'espère que je ne vous ai pas spoilé le pointeur réel.

Si je comprends bien, un timer permet de planifier l'exécution d'une fonction du noyau linux de manière asynchrone.

On voit qu'il y a pas mal de tracepoint dédié au timer selon les cas d'usage :

```
grep 'timer:' /sys/kernel/debug/tracing/available_events
timer:tick_stop
timer:itimer_expire
timer:itimer_state
timer:hrtimer_cancel
timer:hrtimer_expire_exit
timer:hrtimer_expire_entry
timer:hrtimer_start
timer:hrtimer_init
timer:timer_cancel
timer:timer_expire_exit
timer:timer_expire_entry
timer:timer_start
timer:timer_init
alarmtimer:alarmtimer_cancel
alarmtimer:alarmtimer_start
alarmtimer:alarmtimer_fired
alarmtimer:alarmtimer_suspend
```

Maintenant qu'on a dit tout ça, pour le commun des mortels ce tracepoint ne me semble pas utile mais pour debugger le noyau linux ça l'est (par exemple : la retransmission tcp).

Pourquoi j'ai choisi ce tracepoint ? Je cherchais surtout un Tracepoint qui a une sonde ayant une structure pas trop compliqué pour l'article.

#### Les différentes sondes

Si vous regardez le fichiers `/sys/kernel/debug/tracing/events/timer/timer_start/format`, vous verrez les sondes suivantes : 
```
field:void * timer;     offset:8;       size:8; signed:0;
field:void * function;  offset:16;      size:8; signed:0;
field:unsigned long expires;    offset:24;      size:8; signed:0;
field:unsigned long now;        offset:32;      size:8; signed:0;
field:unsigned int flags;       offset:40;      size:4; signed:0;
```

* timer : le timer (type : un pointeur générique)
* function : la fonction du noyau linux qui sera lancé (type : un pointeur générique)
* expires : moment quand la fonction sera lancé (type : un entier)
* now : moment quand le timer a été créé (type : un entier)
* flags : comportement particulier du timer (type : un entier)

#### Hello world programme

Maintenant qu'on a vu la "théorie" pour comprendre le Tracepoint, passons à la pratique.

Commençons par générer le programme Aya :
```Bash
cargo generate --name test-tracepoint \
               -d program_type=tracepoint \
               -d tracepoint_category=timer \
               -d tracepoint_name=timer_start \
               https://github.com/aya-rs/aya-template
```

Testons maintenant le programme :

```Bash
cd test-tracepoint
RUST_LOG=info cargo run
```

Après toutes les étapes de compilations vous verrez apparaître :
```
[INFO  test_tracepoint] tracepoint timer_start called
[INFO  test_tracepoint] tracepoint timer_start called
[INFO  test_tracepoint] tracepoint timer_start called
[INFO  test_tracepoint] tracepoint timer_start called
[INFO  test_tracepoint] tracepoint timer_start called
```

Le programme est bien déclenché ! Cependant le plus dur est devant nous. On va récupérer les résultats des différentes sondes.

#### Récupération des types simples

On va déjà récupérer les sondes suivantes :
```
field:unsigned long expires;    offset:24;      size:8; signed:0;
field:unsigned long now;        offset:32;      size:8; signed:0;
field:unsigned int flags;       offset:40;      size:4; signed:0;
```

Pour récupérer les données il faut utiliser la méthode `read_at` :

![Aya ebpf documentation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x9xr58zlemn76qwv2etm.png)

Donc cela donne :
```Rust
    let expires = unsafe {ctx.read_at::<u64>(24)?};
    let now = unsafe {ctx.read_at::<u64>(32)?};
    let flags = unsafe {ctx.read_at::<u32>(40)?};
    info!(&ctx, "expires={}, now={}, flags={}",
          expires, now, flags);
    Ok(0)
```

* `expires` est à l'offset 24 et a une taille de 8 octets
* `now` est à l'offset 32 et a une taille de 8 octets
* `flags` est à l'offset 40 et a une taille de 4 octets

Maintenant regardons ce que cela donne :
```
RUST_LOG=info cargo run
[...]
[INFO  test_tracepoint] expires=4295181944, now=4295181893, flags=239075328
[INFO  test_tracepoint] expires=4295181915, now=4295181893, flags=117440512
[INFO  test_tracepoint] expires=4295181944, now=4295181893, flags=239075328
[INFO  test_tracepoint] expires=4295181915, now=4295181893, flags=117440512
```

* expires et now sont exprimées en jiffy (rien à voir avec les magasins). J'ai trouvé un [convertisseur jiffy-seconde](https://conversion.org/time/jiffy/second).


![Jiffy définition](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0xt3y04u2hlw0gzuql3l.png)



* avec une simple soustraction on peut trouver dans combien de temps, la fonction va être lancée :

```Rust
    let expires = unsafe {ctx.read_at::<u64>(24)?};
    let now = unsafe {ctx.read_at::<u64>(32)?};
    let flags = unsafe {ctx.read_at::<u32>(40)?};
    let duration = expires - now;
    info!(&ctx, "expires={}, now={}, duration={}, flags={}",
                 expires, now, duration, flags);
```
Ce qui donne :

```
[INFO  test_tracepoint] expires=4295480185, now=4295480163, duration=22, flags=243269633
[INFO  test_tracepoint] expires=4295480214, now=4295480163, duration=51, flags=96468993
[INFO  test_tracepoint] expires=4295480185, now=4295480163, duration=22, flags=243269633
[INFO  test_tracepoint] expires=4295480214, now=4295480163, duration=51, flags=96468993
[INFO  test_tracepoint] expires=4295480185, now=4295480163, duration=22, flags=243269633
[INFO  test_tracepoint] expires=4295480214, now=4295480163, duration=51, flags=96468993
```

On voit que c'est des timers à très courte durée (moins d'une seconde).

#### le pointeur réel de `void *`

Bon tout cela ne résoud pas pour récupérer les sondes qui sont plus compliqués :
```
field:void * timer;     offset:8;       size:8; signed:0;
field:void * function;  offset:16;      size:8; signed:0;
```

Il faut regarder dans le code linux où se trouve : `trace_timer_start`. J'ai un noyau linux 6.1 :

[![linux kernel code](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8bc0cc42w9z2tu93kasi.png)](https://github.com/torvalds/linux/blob/v6.1/kernel/time/timer.c#L609C1-L609C57)

On suppose donc que timer est donc `struct timer_list *`.

On va déjà voir comment récupérer les sondes de timers. On verra function si besoin.

Pour trouver la structure il suffit de faire [une petite recherche](https://github.com/search?q=repo%3Atorvalds%2Flinux+%22struct+timer_list+%7B%22&type=code) dans le code linux :

```C
struct timer_list {
	/*
	 * All fields that change during normal runtime grouped to the
	 * same cacheline
	 */
	struct hlist_node	entry;
	unsigned long		expires;
	void			(*function)(struct timer_list *);
	u32			flags;

#ifdef CONFIG_LOCKDEP
	struct lockdep_map	lockdep_map;
#endif
};
```

On voit que "function" du tracepoint est également dans la structure c'est simplement l'adresse de la fonction qui va être exécuté. Pas super intéressant à superviser.

On voit que cette structure appelle une autre structure : hlist_node :

```C
struct hlist_node {
	struct hlist_node *next, **pprev;
};
```

##### Aya-tool : générer les structures 

Maintenant qu'on a la structure en C. Il faut réussir à le convertir en Rust. On peut le faire à la "main" en regardant le code source de la structure et en le convertissant en Rust. Mais pour débuter il y a un outil qui va le faire pour toi : aya-tool.

###### Installation

Il faut avoir installé `bpftool`. Par exemple sous Debian :
```Bash
apt install bpftool
```

Puis l'installation :

```Bash
cargo install bindgen-cli
cargo install --git https://github.com/aya-rs/aya -- aya-tool
```

###### Utilisation

C'est simple, c'est `aya-tool generate nom_structure`.

Par exemple pour notre cas :
```Bash
aya-tool generate timer_list
```

Cela va t'afficher :
```Bash
/* automatically generated by rust-bindgen 0.72.0 */

pub type __u32 = ::aya_ebpf::cty::c_uint;
pub type u32_ = __u32;
#[repr(C)]
#[derive(Debug, Copy, Clone)]
pub struct hlist_node {
    pub next: *mut hlist_node,
    pub pprev: *mut *mut hlist_node,
}
#[repr(C)]
#[derive(Debug, Copy, Clone)]
pub struct timer_list {
    pub entry: hlist_node,
    pub expires: ::aya_ebpf::cty::c_ulong,
    pub function: ::core::option::Option<unsafe extern "C" fn(arg1: *mut timer_list)>,
    pub flags: u32_,
}
```

Si tu veux afficher toutes les structures disponible :
```Bash
aya-tool generate > all-structures.rs
```

#### Récupération d'une structure complexe

