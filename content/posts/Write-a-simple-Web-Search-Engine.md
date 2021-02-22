---
title: "Write a simple Web Search Engine"
date: Mon Feb 22 10:07:26 CET 2021
draft: true
disableComments: true
---

![BananaSearchScreenshot](/images/bananasearch1.png)

A Web Search Engine consists basically of two parts, a crawler that downloads
Websites and builds an index on their words, and an application to use for the
endusers (usually a webserver) with an query-engine that uses the index created
by the crawler.

In this post we will write a very simple crawler and a minimal webserver that
allows the user to find all websites in the index that contain a (single word)
searchterm.

The goal is not to build anything remotly as powerfull as Google, Bing or
DuckDuckGo, but to create a simple codebase to start hacking on. For this blog
we will use [Python](https://www.python.org/), however it should be easy to follow
along with any other language.

# A simple (inverted) index

The [inverted index](https://en.wikipedia.org/wiki/Inverted_index)
is the hearth of every search engine. A normal index (also
known as forward index) maps a document to ints content. With a search engine
however the user knows a piece of the content and wants to know in which
document it appears. An inverted index does exactly that, mapping the content to
a list of documents. From now on I will refer to this as just "the index".

The most basic index build be a HashMap(Dictionary) mapping a word to a list of
URLs in which it appears.

# The crawler

# The webserver

# Next steps
