---
title: "Go Asynchronous Capabilities"
date: 2022-10-04
slug: "go-async-capabilities"
description: "A quick intro to using Go routines for simple asynchronous execution in Go."
summary: "A quick intro to using Go routines for simple asynchronous execution in Go."
categories: ["golang"]
draft: false
---

Recently while working with Go, one thing that drew my attention was the ease of achieving asynchronous capabilities. One way to do so is using Go routines. Here in this blog, we’ll be looking at implementing a very simple Go routine.

Let’s say you’re writing a simple script to search in a news headlines feed.

# The Synchronous Way

The easiest way would be to simply iterate over all news headlines and using regex to find if the news headline contains the search text. Following code snippet depicts the same.
```golang
package main

import (
	"fmt"
	"log"
	"regexp"
)

var (
	newsPosts = []string{
		"The arrival of October is likely to bring showers across Karnataka's coastal region and Bangalore",
		"To beat the traffic of Bangalore, you can now book a 12-minute helicopter ride",
		"Mercedes India CEO takes an auto-rickshaw after getting stuck in Pune traffic",
		"Pune | 35mm of rain in less than 2 hrs: Downpour disrupts city traffic",
	}

	searchText = "rain"
)

func main() {
	regex, err := regexp.Compile(searchText)
	if err != nil {
		log.Fatal("Failed to create regex")
	}

	for _, post := range newsPosts {
		checkMatch(regex, post)
	}
}

func checkMatch(regex *regexp.Regexp, post string) {
	if regex.MatchString(post) {
		matchIndex := regex.FindStringIndex(post)
		fmt.Printf("Match found in: %v\nMatch Index: %v\n", post, matchIndex)
	} else {
		fmt.Printf("No match found in: %v\n", post)
	}
}
```

The code is pretty simple where while iterating over `newsPosts` we check if the `post` has any regex match. Hence we get the following output:
```shell
No match found in: The arrival of October is likely to bring showers across Karnataka's coastal region and Bangalore
No match found in: To beat the traffic of Bangalore, you can now book a 12-minute helicopter ride
No match found in: Mercedes India CEO takes an auto-rickshaw after getting stuck in Pune traffic
Match found in: Pune | 35mm of rain in less than 2 hrs: Downpour disrupts city traffic
Match Index: [15 19]
```

This was quite simple! But as two news headline searches are independent, waiting was one search to finish to start the another, makes no sense. Thus, we can perform the searches in parallel and that’s where go routines comes into picture.

# The Go Routine Way

Go routine is simply the lightweight thread of execution. Suppose we’ve a function `func()`, then to convert it to a Go routine we’ll just have to add go keyword in front of it as `go func()`.

So in our case, simply changing checkMatch(regex, post) as go checkMatch(regex, post) should do the job asynchronously.

```golang
package main

import (
	"fmt"
	"log"
	"regexp"
	"time"
)

var (
	newsPosts = []string{
		"The arrival of October is likely to bring showers across Karnataka's coastal region and Bangalore",
		"To beat the traffic of Bangalore, you can now book a 12-minute helicopter ride",
		"Mercedes India CEO takes an auto-rickshaw after getting stuck in Pune traffic",
		"Pune | 35mm of rain in less than 2 hrs: Downpour disrupts city traffic",
	}

	searchText = "rain"
)

func main() {
	regex, err := regexp.Compile(searchText)
	if err != nil {
		log.Fatal("Failed to create regex")
	}

	for _, post := range newsPosts {
		go checkMatch(regex, post)
	}
}

func checkMatch(regex *regexp.Regexp, post string) {
	if regex.MatchString(post) {
		matchIndex := regex.FindStringIndex(post)
		fmt.Printf("Match found in: %v\nMatch Index: %v\n", post, matchIndex)
	} else {
		fmt.Printf("No match found in: %v\n", post)
	}
}
```

On executing the above code snippet we get to the following output

Wait, something is off!! How come we didn’t got anything in result ?!

This is because the `checkMatch` is running asynchronously as a go routine and our `main()` do not wait for the go routines to terminate. Resulting, `main()` might get terminated way before the go routines ends. As a solution to it, we could make our `main()` to sleep for say 5 sec using `time.Sleep(5 * time.Second)` while rest of the go routine executes.

Though a better way to do the same would be to use Go WaitGroups. A waitgroup is used to wait for multiple go routines to finish. We’ll simply add `1` to the waitgroup delta value for each go routine initiation and decrement the same using `Done` at the end. We’ve also added `wait()` in `main()` that’ll wait until waitgroup delta becomes zero i.e all go routines are executed completely. Following code snippet depicts the same.

> *Note: if a WaitGroup is explicitly passed into functions, it should be done by pointer.*

```golang
package main

import (
	"fmt"
	"log"
	"regexp"
	"sync"
)

var (
	newsPosts = []string{
		"The arrival of October is likely to bring showers across Karnataka's coastal region and Bangalore",
		"To beat the traffic of Bangalore, you can now book a 12-minute helicopter ride",
		"Mercedes India CEO takes an auto-rickshaw after getting stuck in Pune traffic",
		"Pune | 35mm of rain in less than 2 hrs: Downpour disrupts city traffic",
	}

	searchText = "rain"
)

func main() {
	var wg sync.WaitGroup

	regex, err := regexp.Compile(searchText)
	if err != nil {
		log.Fatal("Failed to create regex")
	}

	for _, post := range newsPosts {
		wg.Add(1)
		go checkMatch(regex, post, &wg)
	}
	wg.Wait()
}

func checkMatch(regex *regexp.Regexp, post string, wg *sync.WaitGroup) {
	defer (*wg).Done()
	if regex.MatchString(post) {
		matchIndex := regex.FindStringIndex(post)
		fmt.Printf("Match found in: %v\nMatch Index: %v\n", post, matchIndex)
	} else {
		fmt.Printf("No match found in: %v\n", post)
	}
}
```

And Ta-da!
We’ve the following output. Here the order of news posts might differ as we’re executing them asynchronously.

```shell
No match found in: To beat the traffic of Bangalore, you can now book a 12-minute helicopter ride
No match found in: The arrival of October is likely to bring showers across Karnataka's coastal region and Bangalore
Match found in: Pune | 35mm of rain in less than 2 hrs: Downpour disrupts city traffic
Match Index: [15 19]
No match found in: Mercedes India CEO takes an auto-rickshaw after getting stuck in Pune traffic
```

I hope this blog was helpful. Happy Gophing!



