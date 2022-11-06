---
title: '[Go基础]goroutine的一个精彩用法'
author: 土川
tags:
  - goroutine
categories:
  - Golang基础
slug: 3042075373
date: 2018-07-22 21:03:00
---
> 这是一个利用goroutine输出素数的一个算法，找个时间写个java版的实现...

<!--more-->
# goroutine实现
![upload successful](/images/pasted-128.png)
算法的核心是，利用合数必定是素数的乘积的性质。用一个协程作为生成器（generate）开始生成“2、3、4.。。”的自然数，利用通道将自然数流入（filter的in、out参数传递）素数过滤器（filter）中，这样生成器生成的自然数如果能从第一个素数流经所有已经生成的filter，那么他是素数，并生成一个新的素数过滤器。  
**第一版**
```go
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.package main
package main

import "fmt"

// Send the sequence 2, 3, 4, ... to channel 'ch'.
func generate(ch chan int) {
	for i := 2; ; i++ {
		ch <- i // Send 'i' to channel 'ch'.
	}
}

// Copy the values from channel 'in' to channel 'out',
// removing those divisible by 'prime'.
func filter(in, out chan int, prime int) {
	for {
		i := <-in // Receive value of new variable 'i' from 'in'.
		if i%prime != 0 {
			out <- i // Send 'i' to channel 'out'.
		}
	}
}

// The prime sieve: Daisy-chain filter processes together.
func main() {
	ch := make(chan int) // Create a new channel.
	go generate(ch)      // Start generate() as a goroutine.
	for {
		prime := <-ch
		fmt.Print(prime, " ")
		ch1 := make(chan int)
		go filter(ch, ch1, prime)
		ch = ch1
	}
}
```
第二版
```go
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

package main

import (
	"fmt"
)

// Send the sequence 2, 3, 4, ... to returned channel
func generate() chan int {
	ch := make(chan int)
	go func() {
		for i := 2; ; i++ {
			ch <- i
		}
	}()
	return ch
}

// Filter out input values divisible by 'prime', send rest to returned channel
func filter(in chan int, prime int) chan int {
	out := make(chan int)
	go func() {
		for {
			if i := <-in; i%prime != 0 {
				out <- i
			}
		}
	}()
	return out
}

func sieve() chan int {
	out := make(chan int)
	go func() {
		ch := generate()
		for {
			prime := <-ch
			ch = filter(ch, prime)
			out <- prime
		}
	}()
	return out
}

func main() {
	primes := sieve()
	for {
		fmt.Println(<-primes)
	}
}
```



