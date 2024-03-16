---
layout: post
title: I/O Multiplexing 101
date: 2024-03-06 11:14:00-0400
description: an example of a blog post with table of contents on a sidebar
tags: Linux
categories: sample-posts
toc:
  sidebar: left
---

In this article, we are going to introduce the basic concept of I/O multiplexing and some of the commonly used mechanisms in real world.

# TL;DR

| Mechanism |                            In a Nutshell                             |                                                                     Use Cases                                                                      |
| :-------: | :------------------------------------------------------------------: | :------------------------------------------------------------------------------------------------------------------------------------------------: |
| `Select`  |                    :crown: OG of I/O multiplexing                    |                                                        To handle a small set of connections                                                        |
|  `Poll`   | :raised_hand: I'm like you but with flexible set of file descriptors |                                                             To handle more connections                                                             |
|  `Epoll`  |          :raised_hands: I'm like you two but more scalable           |                            Commonly used in high-performace server applications when there are thousands of connections                            |
| `Eventfd` |             :v:Here to provide lightweight notification              | Lightweight synchronization between threads. Can be combined with `epoll` to achieve asynchronous event notification in multi-threaded envirnment. |

# What is I/O multiplexing and why do we need it

## Examples of I/O without and with multiplexing

Imagine you are a bartender. It's 3pm and the bar is quite empty right now. You are staring at the ceiling, lost in your mind, when a man comes and asks for Irish Coffee. You start to brew a fresh cup of coffee when he begins to talk about how the recent economic depression hits the company he is working for. You blend one ounce of Irish whiskey into the hot coffee when he complains the shrank bonus making him harder to pay for the mortgage and the life expenses of his family. You pour a large spoon of heavy cream on top of the liquor and watch him swallow the liquid like if it’s the medicine to cure his life. He finishes with a long sigh and you gaze after him as he fades into the shadow of skyscrapers. Then you sink into your thoughts once again.

The bar gets busier as the night falls and there is no time for you to doze off. A group of traders from a nearby startup just close a big deal and they are celebrating with one round of tequila shot. You are busy cutting lemon wedges when a guy with a sullen look comes in and asks for the strongest drink — nasty breakup you guess. You fix him a Long Island and hope this may help him forget about the pain for tonight. Yet he bursts out crying and mumbling about how his girlfriend dumped him for another man. Your best comforting words are “I’m sure you have done all you can possibly do” and “you deserve better than this” because the trading guys asked for another round of margarita and you are too busy shaking the bottles. Eventually, the friends of the heartbroken guy come to pick him up and the traders leave for the next place to continue their drinking marathon. Finally you call it a night with exhausted mind and hurting wrists. 

In the first scenario, you are performing I/O without multiplexing because there is *only one client to be served*. Yet in the second one, you are performing I/O with multiplexing due to the need to *serve multiple clients simultaniously*.

# `Select`: OG of I/O Multiplexing

## How `select` works

## Example code

```c
#include <stdio.h>
#include <sys/select.h>
#include <unistd.h>

int main() {
    fd_set readfds;
    struct timeval tv;
    int retval;
    int stdin_fd = 0;

    // Watch stdin (fd 0) to see when it has input.
    FD_ZERO(&readfds);
    FD_SET(stdin_fd, &readfds);

    // Wait up to five seconds.
    tv.tv_sec = 5;
    tv.tv_usec = 0;

    retval = select(1, &readfds, NULL, NULL, &tv);
    if (retval == -1)
        perror("select()");
    else if (retval)
        printf("Data is available now.\n");
    else
        printf("No data within five seconds.\n");

    return 0;
}
```

## Advantages and limitations

## Typical use cases

# `Poll`: For better connection scalability

## How `poll` works

## Differences and similarities with `select`

## Example code

```c
#include <stdio.h>
#include <poll.h>
#include <unistd.h>

int main() {
    struct pollfd fds[1];
    int timeout_msecs = 5000;
    int retval;
    int stdin_fd = 0;

    // Watch stdin (fd 0) for input.
    fds[0].fd = stdin_fd;
    fds[0].events = POLLIN;

    retval = poll(fds, 1, timeout_msecs);
    if (retval == -1)
        perror("poll()");
    else if (retval)
        printf("Data is available now.\n");
    else
        printf("No data within five seconds.\n");

    return 0;
}
```

## Advantages and limitations

## Typical use cases

# `Epoll`: I'm you, but stronger!

## How `epoll` works

## Edge-triggered vs. level-triggered modes

## Example code

```c
#include <stdio.h>
#include <sys/epoll.h>
#include <unistd.h>

int main() {
    int n, i, stdin_fd = 0;
    int epoll_fd = epoll_create1(0);
    struct epoll_event ev, events[10];

    ev.data.fd = stdin_fd;
    ev.events = EPOLLIN;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, stdin_fd, &ev);

    n = epoll_wait(epoll_fd, events, 10, 5000); // 5 seconds
    for (i = 0; i < n; i++) {
        if (events[i].data.fd == stdin_fd)
            printf("Data is available to read.\n");
    }

    close(epoll_fd);
    return 0;
}
```

## Advantages and limitations

## Typical use cases

# `Eventfd`: To the world of event-driven programming

## How `eventfd` works

## How `eventfd` complements `epoll`

## Example code

```c
#include <err.h>
#include <inttypes.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/eventfd.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
  int       efd;
  uint64_t  u;
  ssize_t   s;

  if (argc < 2) {
    fprintf(stderr, "Usage: %s <num>...\n", argv[0]);
    exit(EXIT_FAILURE);
  }

  efd = eventfd(0, 0);
  if (efd == -1)
    err(EXIT_FAILURE, "eventfd");

    switch (fork()) {
    case 0:
        for (size_t j = 1; j < argc; j++) {
            printf("Child writing %s to efd\n", argv[j]);
            u = strtoull(argv[j], NULL, 0);
                    /* strtoull() allows various bases */
            s = write(efd, &u, sizeof(uint64_t));
            if (s != sizeof(uint64_t))
                err(EXIT_FAILURE, "write");
        }
        printf("Child completed write loop\n");

        exit(EXIT_SUCCESS);

    default:
        sleep(2);

        printf("Parent about to read\n");
        s = read(efd, &u, sizeof(uint64_t));
        if (s != sizeof(uint64_t))
            err(EXIT_FAILURE, "read");
        printf("Parent read %"PRIu64" (%#"PRIx64") from efd\n", u, u);
        exit(EXIT_SUCCESS);

    case -1:
        err(EXIT_FAILURE, "fork");
    }
}
```

```c
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>
#include <sys/eventfd.h>
#include <unistd.h>
#include <pthread.h>

int efd; // Event file descriptor

// Thread function for the consumer
void* consumer_thread(void* arg) {
    uint64_t u;
    ssize_t s;

    printf("Consumer waiting for signal...\n");

    // Wait for event (counter becomes non-zero)
    s = read(efd, &u, sizeof(uint64_t));
    if (s != sizeof(uint64_t)) {
        perror("read");
    } else {
        printf("Consumer received signal: %llu\n", (unsigned long long)u);
    }

    return NULL;
}

// Thread function for the producer
void* producer_thread(void* arg) {
    uint64_t u = 1;
    ssize_t s;

    printf("Producer sending signal...\n");
    s = write(efd, &u, sizeof(uint64_t));
    if (s != sizeof(uint64_t)) {
        perror("write");
    }

    return NULL;
}

int main() {
    pthread_t producer, consumer;

    // Create an eventfd object with an initial value of 0
    efd = eventfd(0, 0);
    if (efd == -1) {
        perror("eventfd");
        exit(EXIT_FAILURE);
    }

    // Create consumer thread
    if (pthread_create(&consumer, NULL, consumer_thread, NULL) != 0) {
        perror("pthread_create");
        exit(EXIT_FAILURE);
    }

    // Sleep for a bit to ensure the consumer is waiting
    sleep(1);

    // Create producer thread
    if (pthread_create(&producer, NULL, producer_thread, NULL) != 0) {
        perror("pthread_create");
        exit(EXIT_FAILURE);
    }

    // Join threads
    pthread_join(producer, NULL);
    pthread_join(consumer, NULL);

    close(efd);
    return 0;
}
```

## Use cases for inter-thread or inter-process communication
