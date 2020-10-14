<!DOCTYPE html>
<html>

<head>
    <title>Title</title>
    <meta charset="utf-8">
    <style>
        @import url(https://fonts.googleapis.com/css?family=Yanone+Kaffeesatz);
        @import url(https://fonts.googleapis.com/css?family=Droid+Serif:400,700,400italic);
        @import url(https://fonts.googleapis.com/css?family=Ubuntu+Mono:400,700,400italic);

        body {
            font-family: 'Droid Serif';
        }

        h1,
        h2,
        h3 {
            font-family: 'Yanone Kaffeesatz';
            font-weight: normal;
        }

        .remark-code,
        .remark-inline-code {
            font-family: 'Ubuntu Mono';
        }
    </style>
</head>

<body>
    <textarea id="source">

class: center, middle

# Title

---

## Features

1. Expressive: Complex scenario, not just a simple HTTP endpoint
2. Protocol diversification: MQTT, NATS, and more
3. Intuitive: Realtime graph
4. Scalable: One million concurrent connection (tba)

---

## Expressive

- No DSL

- Use Go to write scenario

---
class: middle

```go
package main

import "github.com/gobench-io/gobench/scenario"

func export() scenario.Vus { // required
    return scenario.Vus{
        scenario.Vu{     // a virtual user class
            Nu:   1000,  // number of virtual users
            Rate: 200,   // startup rate (Poisson dist)
            Fu:   f1,    // user ref function
        },
    }
}

func f1(ctx context.Context, vui int) {
    for {
        log.Println("tic")
        time.Sleep(1 * time.Second)
    }
}
```

---

## How does Gobench work, then?

<!-- <img src="./gobench-model.png" alt="gobench model" style="width: 100%;"/> -->
<img src="./gobench-model.svg" alt="gobench model" style="width: 100%;"/>

---

## Lesson 1: Go plugin is a joke

--
First attempt for scenario is Go plugin

```shell
go build -buildmode=plugin -o scenario.so scenario.go
```

--
Error when build with different version or build from different path

```shell
panic: plugin.Open("plugin"): plugin was built with a different version of package
```

Still an opened issue https://github.com/golang/go/issues/27751

---

## Lesson 2: User program must run at different kernel process

--
Can we reliability kill a child process (goroutine)?

--
No. Phantom routines.

```go
func f1(ctx context.Context, vui int) {
    for {
        log.Println("tic")
        time.Sleep(1 * time.Second)
    }
}
```

--
Can we recover from a panic?

--
No. And should not.

# Introduction

    </textarea>
    <script src="https://remarkjs.com/downloads/remark-latest.min.js">
    </script>
    <script>
        var slideshow = remark.create({
            highlightLanguage: 'go',
            highlightStyle: 'tomorrow-night-eighties',
            highlightLines: true
        });
    </script>
</body>

</html>