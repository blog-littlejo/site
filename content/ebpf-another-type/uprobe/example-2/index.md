+++
date = '2025-07-23T07:56:34+02:00'
draft = true
title = "uProbe : Sondons une bibliothèque"
series = ["Apprenons uProbe avec eBPF et Aya"]
tags = ["Rust", "eBPF", "aya"]
series_order = 3
+++

## Hello world de uprobe

Nous allons regarder la libc et quand le syscall execve est lancé.

La commande cargo generate suivante permet de génerer le programme eBPF :

```Bash
cargo generate --name test-uprobe \
               -d program_type=uprobe \
               -d uprobe_target=libc \
               -d uprobe_fn_name=execve \
               https://github.com/aya-rs/aya-template
```

```Bash
cd test-uprobe/
RUST_LOG=info cargo run
```

Sur un autre terminal lancer un programme quelconque :
```Bash
ls
```

Sur le terminal cargo run vous verrez :
```
[INFO  test_uprobe] function execve called by libc
```

Ce programme eBPF rappelle beaucoup le programme eBPF de type tracepoint que j'avais créé lors [des articles d'initiation pour eBPF](https://medium.com/@littel.jo/sinitier-%C3%A0-ebpf-avec-aya-c9d570560261). Est-il possible également de récupérer le nom du binaire ?

## Récupérons le nom du binaire

Quels sont les arguments du syscall execve ? Pour le savoir, il suffit de taper :
```
man execve
```

![man execve](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rjyu4ran2cxh5gpldyoq.png)

On voit que la fonction a trois arguments :
* `pathname`: le nom de la commande avec le chemin complet (exemple `/bin/bash`). Il est de type `const char *` (équivalent en rust à `*const u8`).
* `argv`: un ensemble d'arguments de la commande. Il est de type `char *const _Nullable[]` (équivalent en rust à `*const *const u8`)
* `envp` : un ensemble de variables d'environnement de la commande. Il est de type `char *const _Nullable[]` (équivalent en rust à `*const *const u8`).

Il faut donc récupérer le premier argument.

Nous devons modifier la fonction suivante du fichier `test-uprobe-ebpf/main.rs`:

```Rust
fn try_test_uprobe(ctx: ProbeContext) -> Result<u32, u32> {
    info!(&ctx, "function execve called by libc");
    Ok(0)
}
```

Il faut donc chercher à manipuler la variable ctx. Voici [la documentation](https://docs.rs/aya-ebpf/latest/aya_ebpf/programs/probe/struct.ProbeContext.html) :

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
fn try_test_uprobe(ctx: ProbeContext) -> Result<u32, i64> {
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

Nous étions resté à ce niveau là pour les articles d'initiation avec les Tracepoints. Mais nous aurions également pu aller plus loin en récupérant les arguments et les variables d'environnement.

## Récupérons les arguments de la commande

Regardons maintenant comment récupérer les différents arguments de la fonction. Pour cela, il faut rajouter ce bout de code :
```Rust
    let argv: *const *const u8   = ctx.arg(1).ok_or(1u32)?;
```

Comment récupérer le nième argument de la fonction ? Il faut utiliser la fonction [add](https://doc.rust-lang.org/core/primitive.pointer.html#method.add) :

![Method add documentation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zx5o7dq3u99o2tl5mtog.png)

Par exemple :

```Rust
let argv1 = argv.add(1);
```

Par contre arg1 est encore de structure `*const *const u8`. Il faut réussir à le déréférencer pour obtenir `*const u8`.

La bonne nouvelle c'est qu'il y a une fonction toute prête pour cela : `bpf_probe_read_user`. En voici [la documentation](https://docs.rs/aya-ebpf/latest/aya_ebpf/helpers/fn.bpf_probe_read_user.html) :

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

---

Je vous laisse le travail pour afficher les n arguments et les n variables d'environnement.
