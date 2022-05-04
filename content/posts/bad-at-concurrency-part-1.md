---
title: "I'm Bad at Concurrency in Go"
date: 2022-05-06
description: "Improving at concurrency in Go (Part 1)"
tags: ["golang","concurrency"]
# weight: 1
# aliases: ["/first"]
author: "Mike"
showToc: true
TocOpen: false
draft: true
hidemeta: false
comments: false
canonicalURL: "https://penick.io/posts/bad-at-concurrency-part-1/"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
#cover:
#    image: "<image path/url>" # image path/url
#    alt: "<alt text>" # alt text
#    caption: "<text>" # display caption under cover
#    relative: false # when using page bundles set this to true
#    hidden: true # only hide on current single page
#editPost:
#    URL: "https://github.com/<path_to_repo>/content"
#    Text: "Suggest Changes" # edit text
#    appendFilePath: true # to append file path to Edit link
---

Recently, I came across a hypothetical Go concurrency problem. It's a simple, toy problem that's a
potentially useful thought experiment for solving real-world problems. It's also interesting because
it has potential opportunity to be built on and improved. The problem goes like this, given:

```go
type Point struct {
	// Stuff, doesn't matter
}

func ExpensiveServiceCall(point Point) bool {
	// Really expensive call to remote service
}

func ScatterShot(points []Point) bool {
	// Implement me
}
```

you're suppose to implement `ScatterShot(point []Point) bool` which returns `true` when one of the
points "hits", that is, when a call to`ExpensiveServiceCall(point Point) bool` returns `true`.  It
should be implemented in such a way to minimize the run time when given a large number of points
which (likely) means making multiple concurrent calls to the "expensive" service. After little bit
of thought, I started work on the problem, and what I came up with was not very good.

###  Unbounded channels

My first attempt, uses a goroutine for each point in `points` sending the results back, to the
outer function, on a unbounded channel. A `sync.WaitGroup` is used to prevent the outer function
from returning before calculating all the results. Here's what that looks like:

```go
func ScatterShotReallyBad(points []Point) bool {
	ch := make(chan bool)
	var wg sync.WaitGroup

	wg.Add(len(points))

	go func() {
		wg.Wait()
		close(ch)
	}()

	for i, p := range points {
		go func(idx int, point Point) {
			r := ExpensiveServiceCall(point)
			if r {
				ch <- r // This can hang, causing a goroutine leak
			}
			wg.Done()
			fmt.Printf("Point %d done\n", idx)
		}(i, p)
	}

	for r := range ch {
		return r
	}

	return false
}
```

If you run this, there will be times when goroutines will hang (and never return) because the
unbounded channel blocks when there's no longer a receiver which happens when the `ScatterShot()`
function returns. So if you run something like:

```go
func main() {
	points := []Point{{}, {}, {}, {}, {}, {}, {}, {}, {}, {}} // 10 points
	ScatterShotReallyBad(points)
}
```

the following can happen and some goroutines will fail to print that they're done:

```
Point 2 done
Point 3 done
Point 9 done
Point 8 done
Point 4 done
Point 5 done
Point 7 done
Point 0 done
Point 1 done
```

Where's `Point 6 done`? Uh oh!

One potentially bad solution is to use a `select{}` block with a `default:` case to prevent the
channel from blocking:

```go
// ...
go func(idx int, point Point) {
	r := ExpensiveServiceCall(point)
	if r {
		select {
		case ch <- r:
		default: // If the channel blocks, take the default case
 		}
	}
	wg.Done()
	fmt.Printf("Point %d done\n", idx)
}(i, p)
```

It prevents the goroutines from hanging (and leaking), but it creates another
problem! Can you spot the problem?

It causes `ScatterShotSimple()` to, sporadically, return incorrect results. This is because it's
possible for goroutines to miss points that "hit" if the channel isn't receiving yet in the outer
function. In that situation, the `select{}` block chooses the `default:` case causing it to
never send the result. 

Unbounded channels are not a good fit for this problem! 

### Bounded channels

Not only does the initial solution not work correctly, it's overly complex. It turns out that if you
use a bounded channel then things fall into place and the solution is quite elegant. A bounded
channel makes complete sense here because we know the total size of the problem we're trying to
solve: `len(points)`, and this is size to use when creating the buffered channel:

```go
func ScatterShotSimple(points []Point) bool {
	ch := make(chan bool, len(points))

	for i, p := range points {
		go func(idx int, point Point) {
			ch <- ExpensiveServiceCall(point)
			fmt.Printf("Point %d done\n", idx)
		}(i, p)
	}

	for range points {
		if <-ch {
			return true
		}
	}

	return false
}
```

With this solution, the sender never blocks because the buffered channel is always big enough. The
outer function doesn't return prematurely because we wait for `len(points)` results on the shared
channel before returning. This simple solution is quite nice, but it still has problems: 

* It has a goroutine leak. If a point "hits" then the outer function returns immediately potentially
  leaving spawned goroutines running in the background.
* It creates an unbounded number of concurrent calls to the "expensive" service, potentially
  overwhelming it.
* Calls to the "expensive" service are made in a tight loop potentially causing a very high request
  rate that is also likely to overwhelm the service.

How might this be improved? Hint: Go gives us the tools to deal with this.

### Using `context.Context`

That's right! We should be using a `context.Context` to manage the lifetimes of goroutines,
but that means we need to redefine our problem a bit; the expensive call also needs respect the use
of a `context.Context` we pass to it:

```go
func ExpensiveServiceCallWithContext(ctx context.Context, point Point) bool {
  // Really expensive call to remote service, but with a context
}
```

Now with `context.Context` in place we can re-write our function:

```go
func ScatterShotSimpleWithContext(ctx context.Context, points []Point) bool {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel() // This will cancel any in progress calls

	ch := make(chan bool, len(points))

	for i, p := range points {
		go func(idx int, point Point) {
			ch <- ExpensiveServiceCallWithContext(ctx, point)
			fmt.Printf("Point %d done\n", idx)
		}(i, p)
	}

	for range points {
		if <-ch {
			return true
		}
	}

	return false
}
```

The first time this was written I did not use `context.WithCancel()`, and `defer cancel()` in the
first few lines; however, this is necessary to make sure we stop any in-progress goroutines.
`ScatterShotSimpleWithContext()` shouldn't rely on the caller to cancel goroutines that it's
created. 

It's looking better, but it could still overwhelm our "expensive" service! One way we could prevent
that is by limiting the amount of concurrency by not creating a goroutine per call, and instead use
a fixed pool of goroutines.

### Limiting concurrency

Here's the solution I came up with for limiting concurrency. The caller is able to control the
amount of concurrency by passing a `maxConcurrency` parameter which limits the number of goroutines.
Goroutines are no longer created and passed a point for each element in `points` so some other
mechanism of passing points, to the already running goroutines, needs to be used. Sounds like a job for a
channel (it's important that it's buffered, more below). 

```go
func ScatterShotMaxConcurrency(ctx context.Context, maxConcurrency int,
	points []Point) bool {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel() // This will cancel any in progress calls

	// It's important that this is also buffered; otherwise, this will block
	// and prevent an early return when we get a "hit"
	pointsCh := make(chan Point, len(points))

	resultCh := make(chan bool, len(points))

	// Avoid creating more goroutines than are necessary
	if maxConcurrency > len(points) {
		maxConcurrency = len(points)
	}

	for i := 0; i < maxConcurrency; i++ {
		go func() {
			for p := range pointsCh {
				resultCh <- ExpensiveServiceCallWithContext(ctx, p)
			}
		}()
	}

	for _, p := range points {
		pointsCh <- p
	}
	close(pointsCh)

	for range points {
		if <-resultCh {
			return true
		}
	}

	return false
}
```

When implementing this there were some subtleties I missed on the first pass. The first, was I didn't
make `pointCh` a buffered channel. It won't cause incorrect results, but it can prevent an early
return from the function because unbuffered channels block while the worker goroutines are making
service calls. Second, I didn't limit `maxConcurrency` when it exceeds the number of points which
results in creating more goroutines than are necessary to solve the problem. 

There's at least one major improvement we can make to our `ScatterShot()` function and that's to
limit the request rate in each of the worker goroutines. 

### Rate limiting

In all previous solutions, we process the `points` by calling the "expensive" service as fast
as the local machine can make the requests. This is likely to overwhelm the service. One possible
solution is to limit the request rate of each goroutine so that the total request rate is:
`reqsPerSec * maxConcurrency`. This is accomplished by keeping track of the request time and waiting
for remaining time difference required to achieve the desired requested rate:

```go
func ScatterShotRateLimited(ctx context.Context, maxConcurrency int,
	reqsPerSec float64, points []Point) bool {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel() // This will cancel any in progress calls

	// It's important that this is also buffered; otherwise, this will block
	// and prevent an early return when we get a "hit"
	pointsCh := make(chan Point, len(points))

	resultCh := make(chan bool, len(points))

	// Avoid creating more goroutines than are necessary
	if maxConcurrency > len(points) {
		maxConcurrency = len(points)
	}

	timePerRequest := time.Duration(float64(time.Second) / reqsPerSec)

	for i := 0; i < maxConcurrency; i++ {
		go func() {
			for p := range pointsCh {
				b := time.Now()
				resultCh <- ExpensiveServiceCallWithContext(ctx, p)
				e := time.Now().Sub(b)
				d := timePerRequest - e
				if d > 0 { // Finished faster than requested
					t := time.NewTimer(d)
					select {
					case <-t.C:
					case <-ctx.Done(): // Check the context
					}
				}
			}
		}()
	}

	for _, p := range points {
		pointsCh <- p
	}
	close(pointsCh)

	for range points {
		if <-resultCh {
			return true
		}
	}

	return false
}
```

One nuance here, is while waiting on the timer we also need a `select{}` block case for the
context to break out when it's done.


### Summary

My initial solution to this problem was pretty rough, but I didn't give up! I
kept iterating and improving. After giving it a bit of thought, we now have solution that's
(hopefully) no more complex than it needs to be, plays nice with `context.Context`, doesn't leak
goroutines, and won't overwhelm the "expensive" service.

