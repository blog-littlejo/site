+++
date = '2025-07-22T07:56:34+02:00'
draft = true
title = "uProbe : observons une fonction simple de ton programme"
series = ["Apprenons uProbe avec eBPF et Aya"]
tags = ["Rust", "eBPF", "aya"]
showTableOfContents = true
series_order = 2
+++

Nous avons vu ce qu'était un programme de type uProbe dans [la première partie]({{< relref "ebpf-another-type/uprobe/intro/index.md" >}}) : un moyen de sonder les fonctions de vos programmes.

Nous allons créer un petit programme et nous allons le faire réagir avec un programme eBPF de type uProbe avec Aya.

{{< alert "bell" >}}Je suppose que vous êtes déjà dans un [environnement pour développer avec Aya](https://aya-rs.dev/book/start/development/). {{< /alert >}}

---

## Création d'un *hello world* en Go

Pour changer du Rust, nous allons créer un petit programme *hello world* en Go.

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


{{< alert "lightbulb" >}}Vous pouvez tester pour un autre programme compilé comme C/C++ ou Rust. Pour d'autres langages tels que le Python, Java ou C#, il vaut mieux utiliser [USDT](https://docs.ebpf.io/linux/concepts/usdt/){{< /alert >}}

Le but de l'exercice est d'activer le programme eBPF à chaque fois que l'on rentre dans la fonction hello() du programme.

Pour le compiler il suffit de taper :
```Bash
go build -gcflags="all=-N -l" -o hello hello.go
```

{{< alert "lightbulb" >}}Par défaut, le compilateur Go "inline" les fonctions, c'est à dire elle remplace la fonction par le contenu de l'instruction permettant ainsi aux programmes d'être plus performant. Il faut donc empêcher cela si on veut observer la fonction hello.{{< /alert >}}

## Comment trouver les hooks ?

Maintenant qu'on a créé et compilé le petit programme. Il faut maintenant trouver comment déclencher le programme eBPF. Il faut répondre à deux questions : où se trouve le nom du binaire et quelle fonction observée. Voyons cela en détail.

Pour la question :
```
🤷   Target to attach the (u|uret)probe? (e.g libc):
```

Il faut répondre le chemin absolu du binaire. Par exemple je l'ai créer là : `/home/cloud_user/hello`.

Il reste donc l'autre question :
```
🤷   Function name to attach the (u|uret)probe? (e.g getaddrinfo):
```
Que faut-il répondre ? On serait tenté de répondre hello. Mais c'est un peu plus compliqué.

Pour trouver toutes les fonctions du binaire, il suffit de lancer la commande :
```Bash
nm hello | grep " T "
```
{{< alert "lightbulb" >}}La commande `nm` permet de voir tous les symboles qui sont présents dans un fichier binaire. Le `T` correspond au code source.{{< /alert >}}

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

Maintenant qu'on a la réponse aux deux questions, nous pouvons créer notre programme eBPF.

## Création du programme eBPF


### Génération du programme eBPF

Définissons déjà les différentes réponses pour le hook dans des variables, pour mon cas ça sera :
```Bash
uprobe_target=/home/cloud_user/hello
uprobe_fn_name=main.hello
```

Lançons maintenant la commande pour générer le programme Aya :

```Bash
cargo generate --name test-uprobe \
               -d program_type=uprobe \
               -d uprobe_target=$uprobe_target \
               -d uprobe_fn_name=$uprobe_fn_name \
               https://github.com/aya-rs/aya-template
```

Vous aurez la sortie suivante :

```
🔧   program_type: "uprobe" (value from CLI)
🔧   uprobe_target: "/home/cloud_user/hello" (value from CLI)
🔧   uprobe_fn_name: "main.hello" (value from CLI)
🔧   Destination: /home/cloud_user/test-uprobe ...
🔧   project-name: test-uprobe ...
🔧   Generating template ...
[ 1/23]   Done: .gitignore
[ 2/23]   Done: Cargo.toml
[ 3/23]   Done: LICENSE-APACHE
[ 4/23]   Done: LICENSE-GPL2
[ 5/23]   Done: LICENSE-MIT
[ 6/23]   Done: README.md
[ 7/23]   Ignored: pre-script.rhai
[ 8/23]   Done: rustfmt.toml
[ 9/23]   Done: test-uprobe/Cargo.toml
[10/23]   Done: test-uprobe/build.rs
[11/23]   Done: test-uprobe/src/main.rs
[12/23]   Done: test-uprobe/src
[13/23]   Done: test-uprobe
[14/23]   Done: test-uprobe-common/Cargo.toml
[15/23]   Done: test-uprobe-common/src/lib.rs
[16/23]   Done: test-uprobe-common/src
[17/23]   Done: test-uprobe-common
[18/23]   Done: test-uprobe-ebpf/Cargo.toml
[19/23]   Done: test-uprobe-ebpf/build.rs
[20/23]   Done: test-uprobe-ebpf/src/lib.rs
[21/23]   Done: test-uprobe-ebpf/src/main.rs
[22/23]   Done: test-uprobe-ebpf/src
[23/23]   Done: test-uprobe-ebpf
🔧   Initializing a fresh Git repository
✨   Done! New project created /home/cloud_user/test-uprobe
```

### Compilation

Maintenant qu'on a généré le programme, il faut compiler le programme :
```Bash
cd test-uprobe/
RUST_LOG=info cargo run
```

Cela va prendre un peu de temps la première fois :

```
Updating crates.io index
     Locking 103 packages to latest compatible versions
      Adding which v6.0.3 (available: v8.0.0)
  Downloaded anstyle v1.0.11
  Downloaded cfg-if v1.0.1
  Downloaded anyhow v1.0.98
  Downloaded either v1.15.0
  Downloaded cargo_metadata v0.19.2
  Downloaded version_check v0.9.5
  Downloaded which v6.0.3
  Downloaded socket2 v0.6.0
  Downloaded mio v1.0.4
[...]
warning: test-uprobe@0.1.0:     Finished `release` profile [optimized] target(s) in 19.59s
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 50.16s
     Running `/root/build-cache/debug/test-uprobe`
Waiting for Ctrl-C...
```

### Testons maintenant

Laisser le programme Aya tourné et sur un autre terminal, lancer le programme que vous voulez examiner. Pour mon cas :
```Bash
./hello
```

Sur le terminal où on a lancé `cargo run`, Vous devriez voir la sortie suivante à chaque fois que vous lancez le programme :

```
[INFO  test_uprobe] function main.hello called by /home/cloud_user/hello
```

---

Cet épisode est maintenant terminé ! J'espère que vous l'avez apprécié et qu'il n'était pas trop compliqué.

Dans [le prochain épisode]({{< relref "ebpf-another-type/uprobe/example-2/index.md" >}}), nous allons voir un exemple un peu plus complexe en sondant une bibliothèque.
