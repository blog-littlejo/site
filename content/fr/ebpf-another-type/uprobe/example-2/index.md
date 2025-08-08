+++
date = '2025-07-23T07:56:34+02:00'
draft = true
title = "uProbe : Sondons une bibliothèque"
series = ["Apprenons uProbe avec eBPF et Aya"]
tags = ["Rust", "eBPF", "aya"]
showTableOfContents = true
series_order = 3
+++

Nous avons vu ce qu'était un programme de type uProbe dans [la première partie]({{< relref "ebpf-another-type/uprobe/intro/index.md" >}}) : ça peut être un moyen de sonder une bibliothèque.

Nous allons créer un programme eBPF qui va observer la `libc` et une de ses fonctions : `execve`.

{{< alert "bell" >}}Je suppose que vous êtes déjà dans un [environnement pour développer avec Aya](https://aya-rs.dev/book/start/development/). {{< /alert >}}

---

## Hello world de uprobe

Nous avons donc déjà les réponses aux deux questions :
```
🤷   Target to attach the (u|uret)probe? (e.g libc):
🤷   Function name to attach the (u|uret)probe? (e.g getaddrinfo):
```

Voyons comment créer un programme eBPF Hello world pour ce *hook*.

### Génération et compilation du programme Aya

La commande `cargo generate` suivante permet ainsi de génerer le programme eBPF :

```Bash
cargo generate --name test-uprobe-2 \
               -d program_type=uprobe \
               -d uprobe_target=libc \
               -d uprobe_fn_name=execve \
               https://github.com/aya-rs/aya-template
```

Maintenant compilons le :

```Fish
cd test-uprobe-2/
RUST_LOG=info cargo run
```

### Test du programme

Sur un autre terminal lancer un programme quelconque :
```Fish
ls
```

Sur le terminal `cargo run` vous verrez :
```
[INFO  test_uprobe] function execve called by libc
```

Ce programme eBPF rappelle beaucoup le programme eBPF de type *tracepoint* que j'avais créé lors [des articles d'initiation pour eBPF](https://medium.com/@littel.jo/sinitier-%C3%A0-ebpf-avec-aya-c9d570560261). Est-il possible également de récupérer le nom du binaire ?

## Récupérons le nom du binaire

### Comment trouver les arguments du syscall ?

Quels sont les arguments du syscall execve ? Pour le savoir, il suffit de taper :
```Fish
man execve
```

![man execve](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rjyu4ran2cxh5gpldyoq.png)

La partie importante pour nous est :

```C
int execve(const char *pathname, char *const _Nullable argv[],
                  char *const _Nullable envp[]);
```


On voit que la fonction a trois arguments :
* `pathname`: le nom de la commande avec le chemin complet (exemple `/bin/bash`). Il est de type `const char *` (équivalent en Rust à `*const u8`).
* `argv`: un ensemble d'arguments de la commande. Il est de type `char *const _Nullable[]` (équivalent en Rust à `*const *const u8`)
* `envp` : un ensemble de variables d'environnement de la commande. Il est de type `char *const _Nullable[]` (équivalent en Rust à `*const *const u8`).

Il faut donc récupérer le premier argument.

### Modifions le code Aya

Nous devons modifier la fonction suivante du fichier `test-uprobe-2-ebpf/src/main.rs`:

```Rust
fn try_test_uprobe_2(ctx: ProbeContext) -> Result<u32, u32> {
    info!(&ctx, "function execve called by libc");
    Ok(0)
}
```

Il faut donc chercher à manipuler la variable `ctx`. Voici [la documentation](https://docs.rs/aya-ebpf/latest/aya_ebpf/programs/probe/struct.ProbeContext.html) :

![Documentation of ProbeContext](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fsd4lhe89zgzpqito4bd.png)

Il n'y a qu'une méthode qui nous intéresse : [arg](https://docs.rs/aya-ebpf/latest/aya_ebpf/programs/probe/struct.ProbeContext.html#method.arg)

![Documentation of ProbeContext: arg](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s2dvmyb8derdhr1luf6e.png)

Le premier élément est le nom du binaire qui est exécuté.

Il faut donc rajouter un truc comme ça :

```Rust
let arg0: *const u8  = ctx.arg(0).ok_or(1u32)?;
```

De manière similaire qu'avec les Tracepoints, on se retrouve avec :

```Rust
fn try_test_uprobe_2(ctx: ProbeContext) -> Result<u32, i64> {
    let arg0: *const u8  = ctx.arg(0).ok_or(1u32)?;
    let mut buf = [0u8; 128];
    let filename = unsafe {
        let filename_bytes = bpf_probe_read_user_str_bytes(arg0, &mut buf)?;
        from_utf8_unchecked(filename_bytes)
    };
    info!(&ctx, "function execve called by libc {}", filename);
    Ok(0)
}
```

{{< alert "lightbulb" >}}Notez le nom de la fonction `bpf_probe_read_user_str_bytes` qui prend tout son sens ici.{{< /alert >}}

### Testons maintenant la modification

Vérifions que le code fonctionne toujours :

```fish
RUST_LOG=info cargo run
```

Sur un autre terminal, lançons une commande quelconque :
```fish
ls
```

Sur le terminal `cargo run` vous verrez :
```
[INFO  test_uprobe_2] function execve called by libc /usr/bin/ls
```

Nous étions resté à ce niveau là pour les articles d'initiation avec les Tracepoints. Mais nous aurions également pu aller plus loin en récupérant les arguments et les variables d'environnement. Voyons comment le faire.

## Récupérons les arguments de la commande

### Quel argument du syscall ?

Nous l'avons déjà vu, il suffit de lancer cette commande :

```Fish
man execve
```

La partie importante pour nous est :

```C
int execve(const char *pathname, char *const _Nullable argv[],
                  char *const _Nullable envp[]);
```

Il faut donc récupérer le deuxième argument de la fonction `execve`.

### Modifions le code Aya

Ainsi il faut rajouter ce bout de code :
```Rust
    let argv: *const *const u8   = ctx.arg(1).ok_or(1u32)?;
```

Comment récupérer le nième argument de la fonction ? Il faut utiliser la fonction [add](https://doc.rust-lang.org/core/primitive.pointer.html#method.add) :

![Method add documentation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zx5o7dq3u99o2tl5mtog.png)

Par exemple, pour récupérer le deuxième argument (le premier argument est le nom du binaire, on l'a déjà) :

```Rust
let argv1 = argv.add(1);
```

Par contre arg1 est encore de structure `*const *const u8`. Il faut réussir à le déréférencer pour obtenir `*const u8`.

Il y a une fonction toute prête pour cela : `bpf_probe_read_user`. En voici [la documentation](https://docs.rs/aya-ebpf/latest/aya_ebpf/helpers/fn.bpf_probe_read_user.html) :

![bpf_probe_read_user documentation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/d4azamrdz6sgxrnvwp90.png)

```Rust
let argv1_deref: *const u8 = bpf_probe_read_user(arg1_1)?;
```
Maintenant que `argv1_deref` est de structure `*const u8`, il faut le transformer en `&str`. Pour cela le code est similaire au code pour la récupération du nom du binaire.

On se retrouve avec le code suivant :
```Rust
let argv: *const *const u8  = ctx.arg(1).ok_or(0u32)?;
let mut buf = [0u8; 16];
let argname = unsafe {
    let argv1 = argv.add(1);
    let argv1_deref: *const u8 = bpf_probe_read_user(argv1)?;
    let argname_bytes = bpf_probe_read_user_str_bytes(argv1_deref, &mut buf)?;
    from_utf8_unchecked(argname_bytes)
};
info!(&ctx, "function execve called by libc {}", argname);
```

### Testons maintenant la modification

Vérifions que le code fonctionne toujours :

```fish
RUST_LOG=info cargo run
```

Sur un autre terminal, lançons une commande avec une option :
```fish
ls -lrt
```

Sur le terminal `cargo run` vous verrez :
```
[INFO  test_uprobe_2] function execve called by libc /usr/bin/ls
[INFO  test_uprobe_2] function execve called by libc -lrt
```

C'est le comportement qu'on voulait.

Si on lance une commande sans option que se passe-t-il ?
```fish
man
```

Sur le terminal `cargo run` vous ne verrez que :
```
[INFO  test_uprobe_2] function execve called by libc /usr/bin/man
```

Que s'est-il passé ?

Cette partie du code ne s'est pas affiché :

```Rust
info!(&ctx, "function execve called by libc {}", argname);
```

Comme la commande n'a pas d'argument, cette partie du code est partie en erreur :
 
```Rust
let argname = unsafe {
    let argv1 = argv.add(1);
    let argv1_deref: *const u8 = bpf_probe_read_user(argv1)?;
    let argname_bytes = bpf_probe_read_user_str_bytes(argv1_deref, &mut buf)?;
    from_utf8_unchecked(argname_bytes)
};
```

Et donc il n'est jamais passé dans le dernier `info`.

---

## Le code

La récupération des variables d'environnement de la commande est très similaire vu qu'il est du même type que pour les arguments.

Voici le code complet :

TODO

---


Cet épisode est maintenant terminé ! J'espère que vous l'avez apprécié et qu'il n'était pas trop compliqué.

Dans [le prochain épisode]({{< relref "ebpf-another-type/uprobe/example-3/index.md" >}}), nous allons voir comment profiler une fonction d'un programme.

