---
author: "Alfero Chingono"
title: "Hugo blog powered by GitHub Pages and GitHub Codespaces"
date: "2022-03-15"
draft: true
description: "How I created my first Hugo blog powered by GitHub pages"
slug: "hugo-blog-powered-by-github-pages-codespaces"
tags: [
"github",
"hugo",
"devcontainer",
"vscode",
"codespaces"
]
categories: [
"How To",
"Setup"
]
image: "cover.jpg"
---

1. Clone repo in Codespaces
2. Add Hugo devcontainer

   1. Click the "Codespaces" button in the bottom-left corner of Visual Studio Code
   2. Click "Add Development Container Configuration Files"
   3. Click "Show All Definitions"
   4. Click "Hugo (Community)"
   5. Click "OK"
3. Create new hugo site:

   ```bash
   >hugo new site blog
   ```
4. Try to create new hugo module [https://docs.stack.jimmycai.com/getting-started](https://docs.stack.jimmycai.com/getting-started)

   ```bash
   >hugo mod init github.com/achingono/achingono.github.io
   Error: failed to init modules: binary with name "go" not found
   ```
5. Add go binary to devcontainer docker file

   ```docker
   # GO version
   ARG GO_VERSION=18

   #Download Go
   RUN wget https://golang.org/dl/go1.${GO_VERSION}.linux-amd64.tar.gz && \
       tar -C /usr/local -xzf go1.${GO_VERSION}.linux-amd64.tar.gz && \
       rm go1.${GO_VERSION}.linux-amd64.tar.gz

   ENV GOPATH /go
   ENV PATH $GOPATH/bin:/usr/local/go/bin:$PATH
   ```
6. Check go version

   ```bash
   >go version
   go version go1.18 linux/amd64
   ```
7. Turn new site into a Hugo module:

   ```bash
   >hugo mod init github.com/achingono/achingono.github.io
   go: creating new go.mod: module    github.com/achingono/achingono.github.io
   go: to add module requirements and sums:
       go mod tidy
   ```
8. Declare the hugo-theme-stack module as a dependency of your site:

   ```bash
   >hugo mod get github.com/CaiJimmy/hugo-theme-stack/v3
   go: downloading github.com/CaiJimmy/hugo-theme-stack/v3 v3.10.0
   go: downloading github.com/CaiJimmy/hugo-theme-stack    v2.6.0+incompatible
   go: added github.com/CaiJimmy/hugo-theme-stack/v3 v3.10.0
   ```
9. Grab config file from example site

   ```bash
   >wget https://raw.githubusercontent.com/CaiJimmy/hugo-theme-stack/master/exampleSite/config.yaml
   ```
10. Update `config.yaml` file

```yaml
 baseurl: https://achingono.github.io
 languageCode: en-us
 theme: github.com/CaiJimmy/hugo-theme-stack/v3
 paginate: 5
 title: Alfero Chingono

 languages:
     en:
         languageName: English
         title: Alfero Chingono
         weight: 1

 # Change it to your Disqus shortname before using
 disqusShortname: achingono

 # GA Tracking ID
 googleAnalytics:

 # Theme i18n support
 # Available values: en, fr, id, ja, ko, pt-br, zh-cn, zh-tw, es, de, nl, it, th, el, uk, ar
 DefaultContentLanguage: en

 # Set hasCJKLanguage to true if DefaultContentLanguage is in [zh-cn ja ko]
 # This will make .Summary and .WordCount behave correctly for CJK languages.
 hasCJKLanguage: false

 permalinks:
     post: /blog/:slug/
     page: /:slug/

...

module:
  # uncomment line below for temporary local development of module
  # replacements: "github.com/CaiJimmy/hugo-theme-stack/v3 -> ../../hugo-theme-stack"
  imports:
    - path: github.com/CaiJimmy/hugo-theme-stack/v3
      disable: false
```

11. Delete `config.toml` file
12. Create new post
13. Run `hugo server -D`
14. Browse to `http://127.0.0.1:1313`
