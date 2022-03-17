---
author: "Alfero Chingono"
title: "Defining A Task for Quickly Creating Hugo Posts In Visual Studio Code"
date: 2022-03-17T18:39:12Z
draft: true
description: "Using Visual Studio custom tasks, it is possible and easy to create tasks for quickly creating new content."
slug: defining-task-quickly-creating-hugo-posts-vscode
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

Now that I have [custom tasks for building and serving my Hugo site in Visual Studio Code]({{< relref "/post/defining-tasks-for-quickly-building-and-serving-a-hugo-site/index.md" >}}), I wanted to create posts quickly without having to remember the `hugo` command. So, this is what I did:

```json
{
	"version": "2.0.0",
	"tasks": [
		{
			"label": "New Post",
			"type": "shell",
			"command": "title=\"${input:title}\" && slug=\"${title// /-}\" && hugo new content/post/${slug,,}/index.md --source ./src",
			"problemMatcher": []
		}
	],
	"inputs": [
		{
			"id": "title",
			"description": "Enter the title of the new post",
			"type": "promptString"
		}
	]
}
```

The [Variable Substitution](https://code.visualstudio.com/Docs/editor/tasks#_variable-substitution) section of the Visual Studio docs provides information on getting input from the user.

First, I defined the `inputs` by providing the `id`, `description`, and `type`. Then the most interesting part:

The command:

```bash
title="${input:title}" && slug="${title// /-}" && hugo new content/post/${slug,,}/index.md --source ./src
```

does three (3) things using bash:

1. Save the input text into a variable named `title`

    ```bash
    title="${input:title}"
    ```
2. Replace spaces in the `title` variable with dashes and store the output in another variable named `slug`:

    ```bash
    slug="${title// /-}"
    ```
3. Execute `hugo` command to create the content file:

    ```bash
    hugo new content/post/${slug,,}/index.md --source ./src
    ```
    *This last command converts the contents of the `slug` variable to lowercase. Pretty neat!*
    
So far I'm pretty pleased with my experience with the [Hugo static site generator](https://gohugo.io/).
