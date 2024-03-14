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

Imagine you are a bartender. It's 3pm and the bar is quite empty right now. You are staring at the ceiling, totally bored and lost in your mind, when a man comes and asks for Irish Coffee. 

As night falls and it's the happy hour time. The bar is getting busier and there is no time for you to doze off.

# `Select`: OG of I/O Multiplexing

## How `select` works

## Example code

## Advantages and limitations

## Typical use cases

# `Poll`: For better connection scalability

## How `poll` works

## Differences and similarities with `select`

## Example code

## Advantages and limitations

## Typical use cases

# `Epoll`: I'm you, but stronger!

## How `epoll` works

## Edge-triggered vs. level-triggered modes

## Example code

## Advantages and limitations

## Typical use cases

# `Eventfd`: To the world of event-driven programming

## How `eventfd` works

## How `eventfd` complements `epoll`

## Example code

## Use cases for inter-thread or inter-process communication