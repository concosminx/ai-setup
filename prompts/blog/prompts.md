# AI Blog Generation Prompts for Hugo (Repo-aware version)

These prompts are used by the AI agent inside the Hugo source repository.

The AI has access to project files and must read existing posts from the content folder.

Blog posts are located in:

/content
/content/posts
/content/tech
/content/blog

The AI must analyze existing posts before generating a new one.

Workflow:

1. Generate English post
2. Wait for approval
3. Generate Romanian version
4. Generate image prompt

---

## Prompt 1 — Generate New Blog Post (English)

You are an AI working inside a Hugo website repository.

You have access to the source code.

You MUST read existing blog posts from the content folder and match their format.

TASK:

Create a new blog post.

REQUIREMENTS:

- Read at least 2 existing posts from /content
- Detect front matter format
- Detect field order
- Detect writing style
- Detect tag format
- Detect cover format
- Detect markdown style

RULES:

1. Write in ENGLISH
2. Use identical front matter structure
3. Keep same fields
4. Keep same order
5. Keep tags in English
6. Keep categories in English
7. Use same author
8. Use same showToc / TocOpen values
9. Generate new cover image name
10. Follow same writing style
11. Use Markdown
12. Do NOT generate Romanian yet
13. Output ONLY the post

Front matter must look like existing posts.

NEW POST THEME:

<WRITE THE THEME HERE>

OUTPUT:

Return only the new post.
No explanations.
No comments.
No extra text.

---

## Prompt 2 — Generate Romanian Version

Use this ONLY after the English post is approved.

Translate the last generated post to Romanian.

RULES:

- Keep front matter identical
- Keep same date
- Keep same tags (English)
- Keep same categories (English)
- Translate title
- Translate description
- Translate content naturally
- Keep markdown formatting
- Keep code blocks unchanged
- Keep cover image unchanged

OUTPUT:

Return only the Romanian post.

---

## Prompt 3 — Generate Image Prompt

Generate an AI image prompt for the blog cover.

INPUT:

Post title:
<POST TITLE>

Post topic:
<SHORT DESCRIPTION>

STYLE:

- realistic
- tech blog style
- modern
- clean
- cinematic lighting
- high detail
- 16:9
- no text
- no watermark
- no logo

OUTPUT:

Return only the image generation prompt.
