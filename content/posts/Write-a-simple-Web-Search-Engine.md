---
title: "Write a simple Web Search Engine"
date: Thu Feb 25 00:14:36 CET 2021
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
we will use [Golang](https://golang.org/), however it should be easy to follow
along with any other language.

# A simple (inverted) index

The [inverted index](https://en.wikipedia.org/wiki/Inverted_index)
is the hearth of every search engine. A normal index (also
known as forward index) maps a document to its content. With a search engine
however the user knows a piece of the content and wants to know in which
document it appears. An inverted index does exactly that, mapping the content to
a list of documents. From now on I will refer to this as just "the index".

The most basic index build be a HashTable(Dictionary) mapping a word to a list of
URLs in which it appears.

```Go {linenos=true}
index := make(map[string][]string)
index["linux"] = []string{"https://en.wikipedia.org/wiki/Linux"}
index["torvalds"] = []string{"https://en.wikipedia.org/wiki/Linux"}
```

While this would work as an index, we want to improve two things about it.
First, it is kind of a waste to store the whole string of the url with every
word. A quick fix would be to store a pointer to the string with a HashTable
like this `map[string][]*string`. While this would help with the memory
footprint, it would still result in the same wasteful representation when
writing the datastructure to disk. Therefor the better solution is to create
a second list for the documents and in the table for the words we just save
the index of the document.

Second, we want to avoid duplicated documents for a word so we should replace
the list of documentids with a set. (`Note:` as Go has no native set type
I will be using a `map[int]bool` as a replacement.)

```Go {linenos=true}
docsIndex := []string{
    "https://en.wikipedia.org/wiki/Linux",
    "https://en.wikipedia.org/wiki/Microsoft_Windows",
}
wordsIndex := make(map([string]map[int]bool)) {
    "gates": map[int]bool{
        0: true,
    },
    "torvalds": map[int]bool{
        0: true,
        1: true,
    },
}
```

Finally, we should put this code into its own package so that the crawler and
server binaries can both use it without code duplication. While we are at it,
we could define a new `Index` struct and add methods for adding documents,
resolving single word query and reading and writing from disk.

```Go {linenos=true}
// index.go
package index

import (
	"encoding/json"
	"io/ioutil"
)

type Index struct {
	Docs  []string
	Words map[string]map[int]bool
}

// Create a new empty index
func New() *Index {
	return &Index{}
}

// Load an existing index from disk
func Load(path string) (*Index, error) {
	data, err := ioutil.ReadFile("index.json")
	if err != nil {
		return nil, err
	}

	index := Index{}
	err = json.Unmarshal(data, &index)
	return &index, err
}

// Add a document to the index
func (i *Index) AddDoc(url string, words []string) {
	docId := len(i.Docs)
	i.Docs = append(i.Docs, url)
	for _, word := range words {
		if _, ok := i.Words[word]; !ok {
			i.Words[word] = map[int]bool{docId: true}
			continue
		}
		i.Words[word][docId] = true
	}
}

// Retreive all documents that mention a word
func (i *Index) GetDocs(word string) []string {
	if _, ok := i.Words[word]; !ok {
		return []string{}
	}

	docs := make([]string, 0, len(i.Words[word]))
	for id, _ := range i.Words[word] {
		docs = append(docs, i.Docs[id])
	}
	return docs
}

// Save the index to disk
func (i *Index) Save() error {
	data, err := json.Marshal(i)
	if err != nil {
		return err
	}

	err = ioutil.WriteFile("index.json", data, 0666)
	return err
}
```

Before moving on to the crawler, I want to mention that this simple index
has many drawbacks.

- The index cannot solve exact matches
  (in Google this can be achieved to put double quotes around a phrase) as our
  index doesn't save where words appear.

- Saving the index as json is not
  quite as efficient as it could be with a custom file format, but was used here
  for simplicity.

- The index is not scalable as whole thing has to fit into memmory, is at the
  moment not thread-save and cannot easily be spread over multiple
  processes/computers.

- Every website is just represented by its URL, which means this is all we can
  show the user as a result, no websites title, no sniped where the
  searchterm appears just the URL.

# The crawler

Now that we have created the structure of the index, we need to write a program
to build said index. A [Web crawler](https://en.wikipedia.org/wiki/Web_crawler)
starts at a website (this one is called the seed), downloads it, extracts all links to other websites,
adds them to a waitingqueue and then repeats with new URL from the queue.

This way of discovering new websites works quite well, actually it works too
good. For example the [Linux Wikipedia](https://en.wikipedia.org/wiki/Linux)
page has over 1700 hyperlinks, if each of them has again 1700 links we would
have visited almost three milion websites. While this example might be somewhat
extreme, it shows the point that we need to limit the crawler artificially.

There are some quite creative ways to limit the crawler. For example we could
limit the websites we visited to a single URL like `bbc.com` or let the
crawler just allow to visit Austria domains which all end in `.at`.
For now, however, we will only limit the number of websited the crawler can visit.

So with that said our main loop looks something like that:

```Go {linenos=true}
// cmd/crawler/main.go

func main() {
	index := index.New()
	queue := []string{"https://en.wikipedia.org/wiki/Linux"}
	visited := make(map[string]bool)
	limit := 100
	client := http.Client{
		Timeout: time.Second * 10,
	}

	for len(visited) < limit && len(queue) > 0 {
		url, queue := queue[0], queue[1:]
		if _, ok := visited[url]; ok {
			// already visited that url
			continue
		}

		resp, err := client.Get(url)
		if err != nil {
			log.Printf("Unable to download %s: %s", url, err.Error())
			continue
		}
		defer resp.Body.Close()
		log.Printf("Downloaded %s", url)

		visited[url] = true
		links, words := extractData(url, resp.Body)

		index.addDoc(url, words)
		queue = append(queue, links...)
	}
}
```

In this code snipet we used a new function we haven't talked about:
`extractData()`. This function extracts hyperlinks and text from a webpage.
Unfortunatly this means we need to parse HTML which is non-trivial and correct
parsing would exceed the limit of this blog. Luckily this problem is quite
common and there are libraries for almost any programming language that
can parse HTML:

- `Golang`: [goquery](https://github.com/PuerkitoBio/goquery)
- `Python`: [beautifulsoup4](https://pypi.org/project/beautifulsoup4/)
- `Node.js`: [jsdom](https://github.com/jsdom/jsdom)
- `Rust`: [html5ever](https://github.com/servo/html5ever)

If you cannot find a library or don't want to use the one, you can still
achive the most basic results with regular expressions.

With goquery the extract function the complete crawler looks something like
this:

```Go {linenos=true}
// cmd/crawler/main.go
package main

import (
	"io"
	"log"
	"net/http"
	"net/url"
	"strings"
	"time"

	"github.com/PuerkitoBio/goquery"
	"github.com/flofriday/websearchblog/index" // Change this line
)

func extractData(baseUrl string, body io.Reader) ([]string, []string) {
	base, _ := url.Parse(baseUrl)
	doc, err := goquery.NewDocumentFromReader(body)
	if err != nil {
		return []string{}, []string{}
	}

	links := []string{}
	doc.Find("a").Each(func(i int, el *goquery.Selection) {
		href, exists := el.Attr("href")
		if !exists {
			return
		}
		link, err := url.Parse(href)
		if err != nil {
			return
		}
		link = base.ResolveReference(link)
		link.Fragment = ""
		links = append(links, link.String())
	})

	doc.Find("script style").Each(func(i int, el *goquery.Selection) {
		el.Remove()
	})

	words := strings.Fields(doc.Text())
	return links, words
}

func main() {
    ...
}
```

# The webserver

# Next steps
