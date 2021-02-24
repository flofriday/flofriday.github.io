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

type Index struct {
	docsIndex  []string
	wordsIndex map[string]map[int]bool
}

func New() *Index {
	return &Index{}
}

func Load(path string) *Index {

}

func (i *Index) AddDoc(url string, words []string) {
	docId := len(i.docsIndex)
	i.docsIndex = append(i.docsIndex, url)
	for _, word := range words {
		if _, ok := i.wordsIndex[word]; !ok {
			i.wordsIndex[word] = map[int]bool{docId: true}
			continue
		}
		i.wordsIndex[word][docId] = true
	}
}

func (i *Index) GetDocs(word string) []string {
	if _, ok := i.wordsIndex[word]; !ok {
		return []string{}
	}

	docs := make([]string, 0, len(i.wordsIndex[word]))
	for id, _ := range i.wordsIndex[word] {
		docs = append(docs, i.docsIndex[id])
	}
	return docs
}

func Save() error {

}
```

# The crawler

# The webserver

# Next steps
