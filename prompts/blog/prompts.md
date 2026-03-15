---
author: Me
categories:
- tech
cover:
  alt: "`<short alt text>`{=html}"
  image: img/`<generated-image-name>`{=html}.png
date: "\\<AUTO GENERATE CURRENT DATE WITH TIMEZONE +02:00\\>"
description: "`<SHORT DESCRIPTION>`{=html}"
draft: false
showToc: true
tags:
- tag1
- tag2
- tag3
title: "`<GENERATE TITLE>`{=html}"
TocOpen: false
---

# AI Blog Generation Prompts for Hugo

## Prompt 1 --- Generate New Blog Post (English)

You are an AI that writes blog posts for my Hugo static website.

You MUST generate a new blog post that follows EXACTLY the same format,
structure, and writing style as the example I provide.

IMPORTANT RULES:

1.  Write the post in ENGLISH.
2.  Use the EXACT same front matter fields as the example.
3.  Keep the same order of fields.
4.  Keep the same formatting style.
5.  Tags must be in English.
6.  Categories must be in English.
7.  showToc, TocOpen, author, cover format must stay identical.
8.  The cover image name must be generated based on the post topic.
9.  The post must be original but similar in style.
10. Use technical tutorial style like the example.
11. Use Markdown.
12. Do NOT generate Romanian yet.
13. Do NOT explain anything.
14. Output ONLY the post.

FRONT MATTER FORMAT TO FOLLOW EXACTLY:

STYLE EXAMPLE:

`<PASTE EXISTING POST HERE>`{=html}

NEW POST THEME: `<WRITE THE NEW THEME HERE>`{=html}

## Prompt 2 --- Generate Romanian Version

Create the Romanian version of the previously generated blog post.

RULES:

1.  Translate content to Romanian.
2.  Keep front matter format identical.
3.  Keep the same date.
4.  Keep tags in English.
5.  Keep categories in English.
6.  Translate title.
7.  Translate description.
8.  Translate content naturally.
9.  Keep Markdown formatting identical.
10. Do not modify code blocks.
11. Do not change cover image name.
12. Output ONLY the Romanian post.

## Prompt 3 --- Generate Cover Image Prompt

Create an AI image prompt for a blog post cover.

Blog post title: "`<POST TITLE>`{=html}"

Topic: `<SHORT DESCRIPTION>`{=html}

Requirements:

-   realistic
-   modern tech style
-   clean background
-   high detail
-   cinematic lighting
-   16:9 ratio
-   no text
-   no watermark
-   no logo
-   suitable for tech blog cover

Return only the image generation prompt.
