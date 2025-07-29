+++
date = '2025-07-29T07:56:34+02:00'
draft = true
title = "uProbe : Observons une fonction simple de ton programme"
series = ["Apprenons les programmes uProbe avec eBPF et Aya"]
tags = ["Rust", "eBPF", "aya"]
series_order = 2
+++

Nous avons vu ce qu'était un programme de type uProbe.

---

## Création d'un hello world en Go

Avant de créer le programme eBPF, nous allons déjà créer un petit programme hello world.

Pour changer du Rust, nous allons créer un petit programme hello world en Go.

```go
package main

import "fmt"

func hello() {
    fmt.Println("Hello, world!")
}

func main() {
    hello()
}
```

Le but de l'exercice est d'activer le programme eBPF à chaque fois que l'on rentre dans la fonction hello() du programme.

Pour le compiler il suffit de taper :
```Bash
go build -gcflags="all=-N -l" -o hello hello.go
```

## Trouver le hook

Pour la question :
```
🤷   Target to attach the (u|uret)probe? (e.g libc):
```

Il faut répondre le chemin absolu du binaire. Par exemple je l'ai créer là : `/home/cloud_user/go/hello`.

Il reste donc l'autre question :
```
🤷   Function name to attach the (u|uret)probe? (e.g getaddrinfo):
```

Que faut-il répondre ?

Pour trouver toutes les fonctions il suffit de lancer la commande :
```
nm hello | grep " T "
```

`nm` permet de voir tous les symboles qui sont présents dans un fichier binaire. Le `T` correspond au code source.

Donc pour la fonction `hello` :

```
nm -C hello | grep " T " | grep hello
000000000049af40 T main.hello
```

Il faut donc répondre `main.hello`

Si on devait le faire pour la fonction main, on aurait ?
```
nm -C hello | grep " T " | grep main
000000000049af40 T main.hello
000000000049afa0 T main.main
00000000004405e0 T runtime.main
000000000046d060 T runtime.main.func1
0000000000440980 T runtime.main.func2
```

Il faudrait répondre `main.main`.

## Test du programme eBPF

### Génération du programme eBPF

```Bash
cargo generate --name myhelloapp \
               -d program_type=uprobe \
               https://github.com/aya-rs/aya-template
```

```
⚠️   Favorite `https://github.com/aya-rs/aya-template` not found in config, using it as a git repository: https://github.com/aya-rs/aya-template
🔧   program_type: "uprobe" (value from CLI)
🔧   Destination: /home/cloud_user/go/myhelloapp ...
🔧   project-name: myhelloapp ...
🔧   Generating template ...
🤷   Target to attach the (u|uret)probe? (e.g libc): /home/cloud_user/go/hello
🤷   Function name to attach the (u|uret)probe? (e.g getaddrinfo): main.hello
[ 1/23]   Done: .gitignore                                                                                                                                                    [ 2/23]   Done: Cargo.toml                                                                                                                                                    [ 3/23]   Done: LICENSE-APACHE                                                                                                                                                [ 4/23]   Done: LICENSE-GPL2                                                                                                                                                  [ 5/23]   Done: LICENSE-MIT                                                                                                                                                   [ 6/23]   Done: README.md                                                                                                                                                     [ 7/23]   Ignored: pre-script.rhai                                                                                                                                            [ 8/23]   Done: rustfmt.toml                                                                                                                                                  [ 9/23]   Done: myhelloapp/Cargo.toml                                                                                                                                         [10/23]   Done: myhelloapp/build.rs                                                                                                                                           [11/23]   Done: myhelloapp/src/main.rs                                                                                                                                        [12/23]   Done: myhelloapp/src                                                                                                                                                [13/23]   Done: myhelloapp                                                                                                                                                    [14/23]   Done: myhelloapp-common/Cargo.toml                                                                                                                                  [15/23]   Done: myhelloapp-common/src/lib.rs                                                                                                                                  [16/23]   Done: myhelloapp-common/src                                                                                                                                         [17/23]   Done: myhelloapp-common                                                                                                                                             [18/23]   Done: myhelloapp-ebpf/Cargo.toml                                                                                                                                    [19/23]   Done: myhelloapp-ebpf/build.rs                                                                                                                                      [20/23]   Done: myhelloapp-ebpf/src/lib.rs                                                                                                                                    [21/23]   Done: myhelloapp-ebpf/src/main.rs                                                                                                                                   [22/23]   Done: myhelloapp-ebpf/src                                                                                                                                           [23/23]   Done: myhelloapp-ebpf                                                                                                                                               🔧   Moving generated files into: `/home/cloud_user/go/myhelloapp`...
🔧   Initializing a fresh Git repository
✨   Done! New project created /home/cloud_user/go/myhelloapp
```

### Compilation

```
cd myhelloapp/
RUST_LOG=info cargo run
[...]
Après une longue attente
[...]
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 2m 08s
     Running `target/debug/myhelloapp`
Waiting for Ctrl-C...
```

Sur un autre terminal, lancer le programme que vous voulez examiner :
```
./hello
Hello, world!
```

Et sur le terminal où on a lancé `cargo run`. On voit l'output suivant :
```
[INFO  myhelloapp] function main.hello called by /home/cloud_user/go/hello
```

---

Pour aller plus loin, vous pourriez regarder les arguments d'une fonction, ce qu'il retourne ou le temps d'exécution d'une fonction.

Nous allons voir un autre exemple d'uprobe.
