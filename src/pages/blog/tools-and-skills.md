---
layout: ../../layouts/PostLayout.astro
title: "Tools and Skills: How Agents Interact With the World"
date: "2025-01-27"
description: "A deep dive into tool use — how it works at the API level, how to design tools well, and why the distinction between tools and skills matters as your agent grows."
tags: ["tools", "skills", "architecture", "tutorial"]
draft: false
---

In the previous post I walked through my overall architecture. This post goes deeper on one part of it: tool use. It's the mechanism that turns a language model into something that can actually do things — read files, call APIs, write to GitHub, send messages.

I'll cover how tools work at the API level, what makes a tool well-designed, and then get into a question worth taking seriously: when does a flat list of tools stop being enough, and what do you do about it?

---

## What a Tool Actually Is

A tool is a function your agent can call. You define it, your agent decides when to use it, and your code executes it.

From the API's perspective, a tool is a JSON schema — a name, a description, and a list of parameters. You pass these schemas to the model alongside the conversation. The model reads them, decides whether a tool call is needed, and if so, returns a structured `tool_use` block rather than a plain text response.

Here's a minimal tool definition:

```json
{
  "name": "read_document",
  "description": "Read the contents of a document from a repository workspace.",
  "input_schema": {
    "type": "object",
    "properties": {
      "file_path": {
        "type": "string",
        "description": "Path to the file to read, e.g. 'notes/meeting.md'"
      },
      "repo_name": {
        "type": "string",
        "description": "Name of the repository. Defaults to 'workspace'."
      }
    },
    "required": ["file_path"]
  }
}
```

That's it. The model reads this schema and learns: there is a tool called `read_document`, it takes a file path, and it returns the file's contents. Everything else — when to call it, what to do with the result — the model figures out from context.

---

## The Tool Call Cycle

When the model decides to use a tool, it returns a `tool_use` block:

```json
{
  "type": "tool_use",
  "id": "tool_abc123",
  "name": "read_document",
  "input": {
    "file_path": "notes/meeting.md",
    "repo_name": "workspace"
  }
}
```

Your code intercepts this, calls the actual function, and returns a `tool_result` back to the model:

```json
{
  "type": "tool_result",
  "tool_use_id": "tool_abc123",
  "content": "## Meeting Notes\n\nAttendees: Hugh, Stevens..."
}
```

The model then continues — either producing a final response, or making another tool call. This cycle repeats until the task is complete.

In Python, that loop looks roughly like this:

```python
while True:
    response = client.messages.create(
        model="claude-opus-4-5",
        tools=TOOLS,
        messages=messages
    )

    if response.stop_reason == "end_turn":
        # Final response — we're done
        break

    if response.stop_reason == "tool_use":
        # Process all tool calls in the response
        for block in response.content:
            if block.type == "tool_use":
                result = handle_tool_call(block.name, block.input)
                messages.append({
                    "role": "user",
                    "content": [{
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    }]
                })
        # Loop back — send results to the model
        continue
```

The key thing to notice: the conversation history grows with each tool call. The model sees the full chain — what it asked for, what it got back, what it decided to do next. This is what allows multi-step tasks to work.

---

## Designing Tools Well

The tool schemas you write have a direct effect on how well your agent performs. A vague description or a badly structured parameter set leads to the model using tools incorrectly, or not using them when it should. A few things that matter:

**One tool, one responsibility.** I have separate tools for `save_document` and `commit_and_push`. I could have combined them into a single "save and commit" tool, but keeping them separate means the model (and I) have control over the sequence. Sometimes you want to save multiple files before committing. A combined tool would make that awkward.

**Write descriptions for the model, not for a human reader.** The description field is what the model uses to decide whether and how to call your tool. Be specific about what the tool does, what its parameters mean, and what it returns. If there are constraints — file paths must use hyphens, certain parameters default to specific values — say so in the description.

**Return something the model can use.** Every tool call should return a result that gives the model useful information. File contents, confirmation of success, an error message with enough detail to act on. A tool that silently returns nothing leaves the model guessing.

**Fail clearly.** When a tool fails, the error should say what went wrong. "File not found: notes/meeting.md" is useful. "Error: 500" is not. The model will try to recover from failures — give it enough information to do that intelligently.

---

## The Routing Layer

The model tells you which tool to call and with what parameters. Your code has to actually do it. The standard pattern is a routing function — a dispatcher that maps tool names to handler functions:

```python
def handle_tool_call(name: str, inputs: dict) -> str:
    if name == "save_document":
        return handle_save_document(inputs)
    elif name == "read_document":
        return handle_read_document(inputs)
    elif name == "commit_and_push":
        return handle_commit_and_push(inputs)
    # ... and so on
    else:
        return f"Unknown tool: {name}"
```

Each handler validates its inputs, calls the underlying service, and returns a string result. The string goes back to the model as the `tool_result` content.

This is deliberately simple. A flat if/elif chain gets unwieldy as you add tools, but it's easy to understand and debug. You can refactor to a registry pattern once the list grows long enough to warrant it.

---

## When a Flat List of Tools Stops Being Enough

My current toolset has around twenty tools. Each does one thing. The model decides which to call based on the task at hand, and often chains several together — read a file, modify the content, save it, commit it.

This works. But as I've used it, a pattern has become visible: some tasks are really sequences of tools that always get called together in a particular order, with particular logic connecting them. "Research a topic" might mean: search, read several results, synthesise. "Audit a codebase" might mean: examine structure, read key files, create issues for problems found.

Right now, the model figures out these sequences from scratch each time, guided by the system prompt and the task description. That's fine when the sequences are short and the logic is simple. It gets less reliable as sequences get longer and the decision logic between steps gets more complex.

This is the problem that **Skills** address.

---

## What Skills Are

Anthropic describes Skills as reusable, composable units of capability — higher-order abstractions built on top of tools. Where a tool is atomic ("call this API"), a skill is orchestrated ("do this compound thing"). A skill might combine multiple tool calls into a named, repeatable pattern with defined inputs and outputs.

The distinction is worth being precise about:

- A **tool** is an interface to a single capability. The model calls it directly.
- A **skill** is a defined procedure for accomplishing a goal. It may use multiple tools in sequence, apply logic between steps, and return a structured result.

In implementation terms, a skill is a function in your codebase that orchestrates tool calls rather than exposing a raw API. You might expose the skill itself as a tool, so the model can invoke it — but the model doesn't need to know or manage the internal steps.

```python
def skill_research_and_draft(topic: str) -> str:
    # Step 1: search for relevant material
    search_results = handle_search({"query": topic})
    
    # Step 2: read the most relevant sources
    content = []
    for url in parse_top_results(search_results, n=3):
        content.append(handle_fetch_url({"url": url}))
    
    # Step 3: synthesise into a draft
    return handle_draft_from_sources({
        "topic": topic,
        "sources": content
    })
```

The model sees one tool call: `research_and_draft`. The orchestration happens in your code, not in the model's reasoning.

---

## Should You Add Skills?

The honest answer is: it depends on the complexity of your agent's tasks.

For simple agents doing straightforward tasks, a flat tool list is fine. The model handles the sequencing, and that's appropriate — it has the context to make good decisions about ordering and logic.

Skills become worth adding when:

- You have sequences of five or more tool calls that always follow the same pattern
- The logic between steps is non-trivial and you're seeing the model get it wrong
- You want to encapsulate a compound capability so it can be reused reliably

The risk of adding skills too early is the same as any premature abstraction: you build infrastructure for complexity that doesn't exist yet. Start with tools. Watch how the model uses them. Add skills when a pattern is clear enough that it's worth encoding.

---

## What Comes Next

I don't currently have Skills in my architecture. I have a flat toolset that handles my current workload well enough that adding a Skills layer would be solving a problem I don't quite have yet.

But it's the obvious next step — and in the next post, I'll implement it. We'll pick a concrete compound capability, build a skill for it, and look at what changes in both the codebase and the model's behaviour.

The right way to understand why an abstraction is useful is to arrive at it through experience, not to install it upfront and work backwards. That's what we're doing here.
