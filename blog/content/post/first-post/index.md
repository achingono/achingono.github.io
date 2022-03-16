+++
author = "Alfero Chingono"
title = "My first post"
date = "2019-03-10"
description = "How I created my first Hugo blog powered by GitHub pages"
tags = [
    "GitHub",
    "Hugo",
]
+++

1. Add devcontainer
2. Create new hugo site:
	`hugo new site blog`

3. Try to create new hugo module
	https://docs.stack.jimmycai.com/getting-started
	`hugo mod init github.com/achingono/achingono.github.io`
	Error: failed to init modules: binary with name "go" not found

4. Add go binary to devcontainer docker file

5. Check go version
	`>go version`
	`go version go1.18 linux/amd64`

6. Turn new site into a Hugo module:
	`>hugo mod init github.com/achingono/achingono.github.io`
	`go: creating new go.mod: module 	github.com/achingono/achingono.github.io
	go: to add module requirements and sums:
        go mod tidy`

7. Declare the hugo-theme-stack module as a dependency of your site:
	`>hugo mod get github.com/CaiJimmy/hugo-theme-stack/v3`
	`go: downloading github.com/CaiJimmy/hugo-theme-stack/v3 v3.10.0
	go: downloading github.com/CaiJimmy/hugo-theme-stack 	v2.6.0+incompatible
	go: added github.com/CaiJimmy/hugo-theme-stack/v3 v3.10.0`

8. `wget https://raw.githubusercontent.com/CaiJimmy/hugo-theme-stack/master/exampleSite/config.yaml`