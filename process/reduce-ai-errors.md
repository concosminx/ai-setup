# 1. Writing Requirements

Obviously, the AI doesn't know what you want to build. Ideally, you'll spend 10-15 minutes on this step, writing as much detail as possible about the app you have in mind.

Be sure to include:
  * App name
  * Tech stack
  * Core features
  * Database
  * API integrations
  * Design style
  * Things that you don't want to build
  * Ask it to research a comparable, existing app (if applicable)

We then take these and head over to ChatGPT (o3 or better recommended) and ask it to:

```sh
I would like to create concise functional requirements for the following application:
...
Output as markdown code.
```

Example:

```markdown
I would like to create concise functional requirements for the following application:

The app is called ImgxAI and is a midjourney clone, but using OpenAI's image model.
Research midjourney to get a better understanding of the app.

My Requirements:
- It should integrate with the OpenAI APIs. The image model used is gpt-image-1
- The app should have a unified interface with a chat input and a timeline of results
- The timeline should be scrollable and have infinite loading with pagination
- The timeline should be responsive, a grid of 1 on mobile, 2 on tablet and 4 on desktop
- There should be minimal filters on the timeline, with the ability to filter by
  - date
  - status
  - aspect ratio
  - order by newest first or oldest first
- You should be able to download each image by clicking on it
- There should be a details view for the entire prompt which shows:
  - all images for the prompt
  - the jobId
  - created
  - status
  - image count
  - dimensions
  - model
  - quality
  - allow to easily re-run the prompt and download each of the images
- The images should be shown in their correct aspect ratio but within a square container
- You are able to submit the prompt multiple times; more items will be added to the timeline (as background jobs)
- Each prompt can have the following options:
  - quality: (low, mid, high)
  - aspect ratio: 1024x1024, 1536x1024, 1024x1536
  - output_compression ((0-100%)) - default is 50%
  - output_format should be webp
  - moderation should be "low"
  - n (number of images to generate)
  - response_format should be b64_json
  - model should be "gpt-image-1"
- You should be able to see a previous prompt and easily rerun it by clicking on it
- The response images should be stored locally in the browser storage
- You should have a simple navigation bar with a settings button
- In the settings menu you can set your OpenAI API key which is also stored locally in the browser storage
- There is already a codebase using Next.js 15, TailwindCSS, Lucide Icons, React Hook Form with zod and Shadcn UI.

Output as markdown code.
```

Example: the result (ommited)

Go through these in detail and ensure there's nothing in there that you don't want.
Keep it as precise as possible.

Next, we'll take this output and create a complete Product Requirements Document.
This will be the input of the task management system.

# 2. Creating a PRD file (and prompts)

A good PRD file is key to the process. It helps the task management system break down tasks, analyze complexity and understand dependencies between tasks.

Unfortunately, most people don't want to share this prompt ― if you nail this, you got the secret sauce to olympic-level vibe coding. Those people can shove it.

Let me show you. Open up Claude / Anthropic Console, select the claude-3.7-sonnet model or newer and use this prompt to create a PRD file:

```sh
You are an expert technical product manager specializing in feature development and creating comprehensive product requirements documents (PRDs). Your task is to generate a detailed and well-structured PRD based on the following instructions:

<prd_instructions>
{{PRD_INSTRUCTIONS}}
</prd_instructions>

Follow these steps to create the PRD:

1. Begin with a brief overview explaining the project and the purpose of the document.

2. Use sentence case for all headings except for the title of the document, which should be in title case.

3. Organize your PRD into the following sections:
   a. Introduction
   b. Product Overview
   c. Goals and Objectives
   d. Target Audience
   e. Features and Requirements
   f. User Stories and Acceptance Criteria
   g. Technical Requirements / Stack
   h. Design and User Interface

4. For each section, provide detailed and relevant information based on the PRD instructions. Ensure that you:
   - Use clear and concise language
   - Provide specific details and metrics where required
   - Maintain consistency throughout the document
   - Address all points mentioned in each section

5. When creating user stories and acceptance criteria:
   - List ALL necessary user stories including primary, alternative, and edge-case scenarios
   - Assign a unique requirement ID (e.g., ST-101) to each user story for direct traceability
   - Include at least one user story specifically for secure access or authentication if the application requires user identification
   - Include at least one user story specifically for Database modelling if the application requires a database
   - Ensure no potential user interaction is omitted
   - Make sure each user story is testable

6. Format your PRD professionally:
   - Use consistent styles
   - Include numbered sections and subsections
   - Use bullet points and tables where appropriate to improve readability
   - Ensure proper spacing and alignment throughout the document

7. Review your PRD to ensure all aspects of the project are covered comprehensively and that there are no contradictions or ambiguities.

Present your final PRD within <PRD> tags. Begin with the title of the document in title case, followed by each section with its corresponding content. Use appropriate subheadings within each section as needed.

Remember to tailor the content to the specific project described in the PRD instructions, providing detailed and relevant information for each section based on the given context.
```

Obviously, remember to replace `{{PRD_INSTRUCTIONS}}` with the output markdown from the previous step.

Check the output.

Similar to the previous step, go through the output and make sure there's nothing in there that you don't want.

Next, we'll use this PRD to generate tasks, subtasks, analyze complexity and set up dependencies between tasks.

# 3. Setting up the task management system via MCP

In this step, we'll set up the task management system via MCP.

These days, there are a handful of tools that help with AI task management. The most popular ones are:

* Taskmaster AI (prev. Claude Task Master)
* Roocode Boomerang Tasks
* Shrimp Task Manager
* Cline Task Tool
* Built your own .md file with task lists and set up rules to run them in Cursor/Windsurf/etc.

I've started by making my own .md file and then gave each of these a shot, trying to build an app. They all work, but the one that I found to be the sweet spot at the moment is Taskmaster AI.

Taskmaster AI has a CLI and is platform/text editor independent. However, it also has support for Cursor & Windsurf and comes with a MCP server too.
The MCP server is great, because you don't need to switch context to a different window and as such can provide task context to your text editor, too.

So let's set up the Taskmaster AI MCP server.

## 3.1 Installing Taskmaster AI

For this, you'll need an Anthropic API key and a Perplexity API key.
Once you got those, head over to your MCP settings and open up your settings JSON.
Install the actual MCP Server

Prepping codebase

If you're doing this on an existing codebase, you can just skip this step.
But be warned! ⚠️

Do NOT start from an empty codebase. There will be dragons. And mostly, lots of tokens wasted for a wonky foundation. Use a CLI, your favorite template or just use Shipixen, but whatever you do, do not start from scratch!

## 3.2 Setting up Cursor Rules

## 3.3. Initializing TaskMaster AI

Before getting into this make sure to:
* Select "Agent mode" in your text editor
* Select "Claude 3.7 Sonnet" as your model

Now that we have the MCP server & codebase set up, we can initialize TaskMaster AI.
1. Paste the PRD output from above into a new file under scripts/prd.txt.
2. In a Cursor Chat, paste the following prompt:

```sh
I've initialized a new project with Claude Task Master. I have a PRD at scripts/prd.txt.
Can you parse it and set up initial tasks?
```

This will transform your PRD file into a series of files (under /tasks), plus a /tasks/tasks.json file that contains a structured list of tasks, subtasks and metadata.

3. Analyze Complexity. Now, we will ask Taskmaster AI to analyze the complexity of the tasks. This will use Perplexity to do web research and decide on a complexity score from 1-10.

```sh
Can you analyze the complexity of our tasks to help me understand which ones need to be broken down further?
```

You can reply with Please break down the identified tasks into subtasks. and it will do just that.

4. Further breaking down tasks.

It is also possible to break down individual tasks if you see some that are too complex.
Ideally, you'd break down tasks in units that seem easy to implement to you as a hooman. If somethings seems complex to you, it'll probably be complex for the AI too.

```sh
Task 3 seems complex. Can you break it down into subtasks?
```

And that's the setup! You can continue to break down tasks as needed.
If you missed something, don't worry. You can always add tasks later or change direction in the implementation.

### Recipes

* add a new task

```sh
Let's add a new task. We should implement sorting of the timeline.
Here are the requirements:

- you should be able to sort the timeline by date
- a dropdown should be available to select the sorting direction
- the sorting should be persisted when a new page is loaded
```

* change direction of a task

```sh
There should be a change in the image generation task.
Can you update task 3 with this and set it back to pending?

The image generation should use gpt-image-1 as the model.
```

* deprecate a task

```sh
Task 8 is not needed anymore. You can remove it.
```

# 4. Building the app (prompt loop)

1. Open a new chat in Cursor and prompt it: `Show tasks`
2. Then, prompt it: `What's the next task I should work on? Please consider dependencies and priorities.`
3. Implement the task. `Implement task 2 and all of its subtasks.`
4. Iterate.

# 5. Pro tips

1. You can still add extra context to each task run

When prompting the AI to work on the next task, ensure to provide additional context on e.g. UI preferences, API docs etc. You can also attach images!

This will guide it on the path you want to go.

2. Break down files that are large than 500 lines

AI is not great at handling large files. So if you have a file that is larger than 500 lines, break it down into smaller files.

Here's a prompt you can use:

```sh
Break down this file into logical modules so it's easier to ready.
Create directories if needed and move utils and interfaces to separate files, maintaining a domain-driven file structure.
```

3. Bugs are also tasks!

With a complex enough system, you will eventually run into a bug that requires a change to the underlying architecture.
AI will try to apply a superficial fix, but you'll just end up going in circles.

So what you can do is create a new task for the bug and implement it.

Here's a prompt you can use:

```sh
The filter feature is not working as expected. Create a new task to fix it:
- the filter should be case insensitive
- it should work with pagination
- it should work with the debounce
```

4. Sometimes too much context will make things worse. So always start a new chat when implementing the next task.
