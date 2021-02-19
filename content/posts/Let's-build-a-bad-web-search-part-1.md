---
title: "Let's build a bad Web Search - Part 1"
date: 2021-02-18 09:21:00 +0100
draft: false
disableComments: true
---

In this series we explore how a simple Web Search works (like Google in the 90s).
We will use [Go](https://golang.org/) to build the backend, however I
will not use too specific Go features or packages and it should be easy to
follow along with any other language.

# Extracting Text

To create an search-index we first need pure text-file which means we need to
extract the text from an HTML file.

Lets look at a example:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Page Title</title>
  </head>
  <body>
    <h1>This is a Heading</h1>
    <p>This is a paragraph.</p>
  </body>
</html>
```

A naive way to extract the text from this file is to remove everything between
`< >`. This aproach has many drawbacks like also extracting Javascript between
`<script></script>` tags, but should be good enough for now.

```go
import (
	"html"
	"net/url"
	"regexp"
	"strings"
)

func ExtractText(text string) string {
	replaceScriptStyle := regexp.MustCompile("(<style[^<>]*>.*?<\\/style>|<script[^<>]*>.*?<\\/script>)")
	replaceTags := regexp.MustCompile("<.*?>")
	replaceMultipleSpace := regexp.MustCompile("\\s\\s+")

	text = strings.ReplaceAll(text, "\n", " ")
	text = replaceScriptStyle.ReplaceAllString(text, "")
	text = replaceTags.ReplaceAllString(text, " ")
	text = replaceMultipleSpace.ReplaceAllString(text, " ")
	text = html.UnescapeString(text)

	return text
}
```
