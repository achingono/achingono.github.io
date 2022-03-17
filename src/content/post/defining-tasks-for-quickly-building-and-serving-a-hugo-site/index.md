---
author: "Alfero Chingono"
title: "Defining Tasks for Quickly Building and Serving a Hugo Site"
date: 2022-03-16T20:44:03Z
draft: true
description: "Using Visual Studio custom tasks, it is possible and easy to create tasks for quickly building and serving a Hugo site."
slug: defining-tasks-quickly-building-serving-hugo-site
tags: [
    "vscode",
    "tasks"
]
categories: [
"How To",
"Setup"
]
image: "cover.jpg"
---

Now that I can run the site locally, I needed a quick way to run the `hugo` command for creating new blog posts. After quick search on the internet, I found the article [Tasks in Visual Studio Code](https://code.visualstudio.com/Docs/editor/tasks) very helpful. In short, I had to define the build task this way:

```json
{
	"version": "2.0.0",
	"tasks": [
		{
			"label": "Build",
			"type": "shell",
			"command": "hugo --source ./src",
			"group": {
				"kind": "build",
				"isDefault": true
			}
		}
	]
}
```

This task executes [the `hugo` command](https://gohugo.io/getting-started/usage/#the-hugo-command) which generates your website to the `public/` directory by default and makes it ready to be deployed to your web server. The `--source` argument ensures the correct folder is built. In addition, setting `"kind": "build"` and `"isDefault": true` ensures that this task is executed with the <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>B</kbd> keyboard shortcut.

Next, I added another task to run the site locally:

```json
{
	"version": "2.0.0",
	"tasks": [
		{
			"label": "Build",
			"type": "shell",
			"command": "hugo --source ./src",
			"group": {
				"kind": "build",
				"isDefault": true
			}
		},
		{
			"label": "Serve",
			"type": "shell",
			"command": "hugo server -D --source ./src",
			"group": {
				"kind": "build"
			},
			"isBackground": true,
			"problemMatcher": []
		}
	]
}
```

This task executes the `hugo server` command. The `-D` flag ensures that we can preview content in draft mode, and again, the `--source` argument ensures the correct folder is served. It's important to note that the `Serve` does not have `"isDefault": true` since we do not want the two tasks to conflict when using the <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>B</kbd> keyboard shortcut.

So far I'm pretty pleased with my experience with the [Hugo static site generator](https://gohugo.io/).