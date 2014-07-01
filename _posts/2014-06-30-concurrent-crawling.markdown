---
layout: post
title: Concurrent Crawling with Go and Python
category: programming
---

What crawls better, gopher or a snake? :)

I was recently talking to my friend who is also a fellow python programmer, about the system he built to crawl millions of RSS feeds very fast. He has his solution working based on gevent and I was advocating him to take a look at Go, after being impressed with it after hearing [Brad Fitzpatrick's talk](http://talks.golang.org/2013/oscon-dl.slide#1) at the GoSF meetup on how they sped up dl.google.com using Go. At the end of our discussion, I got the itch to build out a sample crawler in both Python and Go to compare.

The primary aim of this exercise was to see how the program design and structure compare across the two languages and to record my experiences while doing it. Note that I am a newbie to both Gevent and Go.

## The Problem

Given a source link, follow through and fetch the links you encounter in every page content. Limit total page fetches to a specified number. Link uniqueness doesn't matter.

## Solution Design

To solve this problem, we spawn a specified number of worker subtasks. Each worker constantly fetches the next link from a queue of links and uses a simple regular expression to look for new links to feed into the queue. Limiting total page fetches is done by keeping track of the number of pages fetched or links added to the queue. Go and Python crawlers do this a bit differently. I initially based my solution on this [Gevent Tutorial](http://blog.hownowstephen.com/post/50743415449/gevent-tutorial) but grew out of it.

In both cases, the underlying scheduler (both Gevent and Go have their own scheduler) interleave execution of workers by starting one worker when some other worker is waiting blocked on something (in our case, a page fetch). This non-blocking makes the workers asynchronous on such network events without the programmer having to actively think about the context switches between workers. Now let's compare the code and see how they look.

The basic flow is:

>Spawn specified number of `worker` tasks

>Each `worker` - continously get links from the queue of links and call `do_work`

> `do_work` fetches the page, extracts links from the page and adds them to the queue of links

Now let's look at the code!

## Python
{% highlight python linenos %}
#!/usr/bin/env python
# monkey-patch
import gevent.monkey
gevent.monkey.patch_all()

import sys
import hashlib
import re

import requests
from requests.exceptions import ConnectionError, MissingSchema
import gevent.pool
from gevent.queue import JoinableQueue

source = sys.argv[1] #source link
num_worker_threads = int(sys.argv[2]) #specifying how many workers fetch concurrently
num_to_crawl = int(sys.argv[3]) #maximum no. of pages to fetch

crawled = 0
links_added = 0
#JoinableQueue lets us wait till all the tasks in the queue are marked as done.
q = JoinableQueue() 

#This function does the actual work of fetching the link and 
#adding the extracted links from the page content into the queue
def do_work(link, crawler_id):
    global crawled, links_added

    #NOTE: uncomment this line to get extra details on what's happening
    #print 'crawling', crawled, crawler_id, link

    #Fetch the link
    try:
        response = requests.get(link) 
        response_content = response.content
    except (ConnectionError, MissingSchema):
        return
    crawled += 1

    #Some sample non-IO bound work on the content. In the real world, there 
    #would be some heavy-duty parsing, DOM traversal here.
    
    m = hashlib.md5()
    m.update(response_content)
    m.digest()

    #Extract the links and add them to the queue. Using links_added
    #counter to keep track of links fetched. Possible race condition.
    for link in re.findall('<a href="(http.*?)"', response_content):
        if links_added < num_to_crawl:
            links_added += 1
            q.put(link) 

#Worker spawned by gevent. Continuously gets links, works on them and marks
#them as done.
def worker(crawler_id):
    while True:
        item = q.get()
        try:
            do_work(item, crawler_id)
        finally:
            q.task_done()

#Spawning worker threads.
crawler_id = 0
for i in range(num_worker_threads):
    gevent.spawn(worker, crawler_id)
    crawler_id += 1 

q.put(source)
links_added += 1

q.join()  # block until all tasks are done
{% endhighlight %}

The comments in the code above should help walk you through what's going on. A queue is used to keep track of links to crawl. The links are crawled depth first. Multiple workers fetch links from the queue at the same time and process them. The workers use a global counter to make sure that the total number of links added to the queue does not cross the maximum page fetch limit, till then they keep adding more links to the queue. The program exits when all the links added to the queue are all marked as processed.

## Go

{% highlight cpp linenos %}

package main

//The builtins are limited. Making a lot of imports necessary
import ("sync"
        "net/http"
        "regexp"
        "io/ioutil"
        "os"
        "bytes"
        "fmt"
        "strconv"
        "runtime"
        "crypto/md5"
        "io"
)

var source = os.Args[1] //source link
var num_worker_threads, _ = strconv.Atoi(os.Args[2]) //specifying how many workers
var num_to_crawl, _ = strconv.Atoi(os.Args[3]) //maximum no. of pages to fetch

var crawled = make(chan int, num_to_crawl) //buffered channel to count page fetches
var links = make(chan string, num_to_crawl) //buffered channel as a queue of links

func do_work(link string, crawler_id int) {
    //fmt.Println("crawling", crawler_id, link)
    re := regexp.MustCompile(`<a href="(http.*?)"`)
    resp, err := http.Get(link)
    if err != nil {
        return
    }
    defer resp.Body.Close()
    content, _ := ioutil.ReadAll(resp.Body)
    contentString := bytes.NewBuffer(content).String()
    h := md5.New()
    io.WriteString(h, contentString)
    var _ = h.Sum(nil)

    //Try to add a link to the queue of links. If it is full, the default case
    //returns as there is no point in adding more links to the queue as our
    //maximum page fetches is limited anyways.
    for _, match := range re.FindAllStringSubmatch(contentString, -1) {
        select {
        case links <- match[1]:
        default:
            return
        }
    }
}

func worker(crawler_id int) {
    //If the crawled channel's buffer is full, no more pages to fetch
    //so no more work to do.
    for {
        select {
        case crawled <- 1:
            do_work(<-links, crawler_id)
        default:
            return
        }
    }
}

func main() {
    var _ = fmt.Println
    //Try to make the workers use all the logical CPUs in the machine.
    runtime.GOMAXPROCS(runtime.NumCPU())
    var wg sync.WaitGroup
    links <- source

    for i:=0; i < num_worker_threads; i++ {
        // Increment the WaitGroup counter.
        wg.Add(1)
        // Launch a goroutine worker.
        go func(crawler_id int) {
                // Decrement the counter when the goroutine completes.
                defer wg.Done()
                worker(crawler_id)
        }(i)
    }
    // Wait for all the workers to finish.
    wg.Wait()
    close(crawled)
    close(links)
}

{% endhighlight %}

The design and structure is pretty much the same as Python/Gevent but there are key differences. The concurrency is built-in. We are not using a third-party library or even a package for that. The code can run goroutines in multiple CPUs at the same time. This is very useful if there was lot of say, text processing and other CPU intensive tasks being done by the workers but ours is pretty much a network bound problem. The use of channels and select statements enables us to control the concurrent workers. Notice the use of channels as a queue to track what links to crawl. Even though the concurrency primitives are built-in and simple, deeper understanding is necessary to exploit the power of Go by combining them in novel ways.

## What I learned

This experiment was not a study in performance but about the ease of designing concurrent solutions for IO or network bound problems. I learned that you gain a lot of powerful expressivity with Go but that comes with a need for deeper understanding and experience. Neither Gevent nor Go is fundamentally complex or complicated to use. With Go, you can also have it work for CPU bound problems.

I guess it will take me some time to think in terms of channels and select statements in Go but till then, Gevent offers all I can ask for to build on top of my Python knowledge and write simple concurrent programs quickly.

The code is available [here](https://github.com/venkat/GopherSnakeCrawlers). It also has a shell script that lets you run the crawler with different number of workers, page fetch limits.
