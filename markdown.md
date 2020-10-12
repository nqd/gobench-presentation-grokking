## Gobench - Load Test Tool

Nguyễn Quốc Đính

17/10/2020



## Features

1. Expressive: Complex scenario, not just a simple HTTP endpoint
2. Protocol diversification: MQTT, NATS, and more
3. Intuitive: Realtime graph
4. Scalable: One million concurrent connection (tba)



## Expressive

- No DSL

- Use Go to write scenario




```go [5|7|8|9|10]
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
```



```go
func f1(ctx context.Context, vui int) {
    for {
        log.Println("tic")
        time.Sleep(1 * time.Second)
    }
}
```


## External 1.2

Content 1.2



## External 2

Content 2.1



## External 3.1

Content 3.1


## External 3.2

Content 3.2


## External 3.3

![External Image](https://s3.amazonaws.com/static.slid.es/logo/v2/slides-symbol-512x512.png)
