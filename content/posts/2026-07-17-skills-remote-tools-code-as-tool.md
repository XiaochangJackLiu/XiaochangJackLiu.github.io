---
title: "Skills, Remote Tools, and Code-as-Tool: Choosing the Execution Boundary"
description: "How skills, remote tools, and executable code place agent capabilities across different information, execution, credential, and state boundaries."
date: 2026-07-17T10:00:00+08:00
draft: false
tags:
  - AI agents
  - agent engineering
  - skills
  - MCP
  - tools
  - code-as-tool
---

## Disclaimer

These are my personal observations and opinions.

## Background

At a time when agent engineering discussions are increasingly about meta-harnesses, loop engineering, dynamic workflows, and context engineering, writing about skills and tools may sound slightly old-fashioned. They seem like the basic vocabulary of agent systems: useful, familiar, and perhaps no longer especially interesting.

I have started to think the opposite.

A harness can decide when an agent should reason, act, retry, verify, or stop. A loop can organize feedback and recovery. But neither has much meaning until the system answers a more basic set of questions: Where does the agent's knowledge come from? What can it act on? Where does that action execute? Who holds the credentials? Where does the resulting state live?

Without skills, tools, or some other way to deliver capability, a harness is orchestrating an empty loop.

In my first post, I described prompts and code as the two fundamental ingredients of agent engineering. In the second, I discussed who should own the structure of an agent workflow. This post moves one level lower. It is about where an agent's capability should live and where it should execute.

This question became practical for us while building a domain-specific coding assistant. At one point, converting our MCP tools into locally executable code looked like the most flexible design. The agent could chain operations, process intermediate results without repeatedly placing them in the model's context, and create new combinations without requiring a new remote tool for every query shape.

In the end, we kept the MCP tools remote.

The reason was not that remote tools were universally better. It was that our proposed combination of generated code and locally exposed remote capabilities depended on two pieces of infrastructure that are easy to underestimate: a runtime in which the code can execute, and a safe way for that runtime to access credentials. Once those constraints became explicit, the architectural trade-off looked different.

## Three Ways to Deliver Capability

Skills, tools, and code-as-tool are often discussed as competing agent features. I find it more useful to treat them as different ways of delivering capability to an agent.

They differ in what is delivered, where execution happens, and who owns the surrounding infrastructure.

### Skill: Packaged Information

My simplest mental model is:

> A skill is packaged information.

That information may include instructions, domain knowledge, examples, constraints, decision rules, templates, and recipes for using tools. Packaging it as a skill makes the information reusable and gives the agent guidance about when and how it should be applied.

A skill primarily changes how the agent understands and approaches a task. It can teach the agent how an experienced practitioner debugs a problem, reviews a design, searches a codebase, or follows a domain convention.

This does not mean a skill must contain only prose. A skill package may include scripts, templates, or other executable support. But once a script is executed, a separate system question appears: Where does it run, what can it access, and which credentials can it use? The package may contain both information and code, but the execution boundary still exists.

Packaged does not mean isolated.

When designing our skill system, I cared as much about the connections between skills as the content inside each individual skill. A domain task rarely maps cleanly to one self-contained package. One skill may help the agent understand the task, another may describe a domain resource, and a third may explain how to validate the result. If the agent loads only the first skill and has no path to the others, the packages may be individually well written but collectively incomplete.

We connected skills in three ways.

First, a router skill could act as an entry point for a family of tasks. Its purpose was not to contain all of the domain knowledge itself, but to recognize the shape of a task and point the agent toward more specialized skills.

Second, a skill could directly reference another skill when the relationship was known and important. For example, an implementation skill could explicitly direct the agent to a validation skill before considering the task complete. These references formed deliberate paths through the skill system rather than relying on the model to rediscover every relationship.

Third, every skill had tags. Tags provided looser connections than direct references. They allowed skills to be grouped by domain, task type, resource, or concern, and gave the agent a way to move laterally from one relevant skill to other potentially relevant ones.

The resulting system looked less like a collection of independent prompt packages and more like a connected information space:

```text
Router skill -> task-specific skills
Skill A      -> directly referenced Skill B
Skill tags   -> other skills with related concerns
```

This changed how we approached skill loading.

[Anthropic's original description of Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) uses progressive disclosure by preloading the `name` and `description` of every installed skill, then loading the full skill only when it appears relevant. We kept the progressive-disclosure principle, but did not place the metadata for every skill in pinned context.

Instead, we exposed the skill catalog through a virtual-filesystem mental model. The agent received a stable discovery interface and could explore the skill space with familiar operations:

- `ls`: list skills and return compact metadata;
- `grep`: search names, tags, and metadata using an exact term or regular expression;
- `find`: search semantically when the agent knew the intent but not the skill name or vocabulary;
- `cat`: load the content of a selected skill.

The first response was intentionally metadata-first. It returned enough information for the agent to estimate relevance without paying the token cost of loading complete skill instructions. The tags in that metadata served a second purpose: they exposed nearby skills that the initial query might not have discovered directly.

This gave progressive disclosure two dimensions. The agent could move vertically from compact metadata to full skill content, and horizontally through router skills, direct references, and shared tags.

```text
Metadata -> Full skill                     deeper disclosure
Relevant skill -> Connected skills        broader discovery
```

At startup, the agent did not need a complete catalog in context. It needed to know how to explore the catalog. This replaced a growing pinned list with a small and stable discovery contract.

### Remote Tool: A Controlled Capability Boundary

A tool gives the agent a bounded action it can take. A remote tool, in particular, provides controlled access to resources or execution that remain outside the user's local environment.

The agent sees an interface: a name, description, input schema, and result. The service behind that interface owns the implementation, dependencies, permissions, and often the credentials required to reach the underlying resource.

This makes a remote tool more than a remote function call. It is also a system boundary. It determines what the agent is allowed to request, how that request is validated, and what portion of the underlying system is exposed.

Remote tools are useful for exploration, but they are not limited to reading information. They may query metadata, run validation, create an artifact, or perform an operation with external side effects. The important property is that the capability remains bounded and remotely managed.

### Code-as-Tool and Tool-as-Code: Runtime Versus Interface

The two phrases are easy to confuse, so I use them to describe two different parts of the design.

**Code-as-tool is a runtime capability.** Instead of selecting only from predefined operations, the agent can write a program for the task at hand and ask a general code-execution environment to run it. For example, the agent may generate Python to transform a CSV or Bash to inspect a project. The generated program is the action.

**Tool-as-code is an interface design.** A predefined domain capability is delivered as a library, CLI, or script that ordinary code can call. For example, `search_resources()` and `read_resource()` may be functions in a client library rather than two tools invoked directly through MCP. The implementation already exists; what changes is how the agent can access it.

This distinction is close to the architecture Anthropic describes in [*Code execution with MCP: Building more efficient agents*](https://www.anthropic.com/engineering/code-execution-with-mcp). In that design, MCP capabilities are presented as code APIs that an agent can discover and invoke from generated programs. Using the terminology of this post, the code-execution environment provides code-as-tool, while the program-accessible MCP interfaces provide tool-as-code. The article also shows why the combination matters: tool definitions can be loaded on demand, intermediate results can be filtered before entering the model's context, and multiple operations can be composed in one program.

The two patterns are not interchangeable, but they are often combined:

```text
Code-as-tool                     Tool-as-code
Agent writes a program    +     Program imports or invokes predefined capabilities
```

An agent uses its code-execution capability to generate a program, and that program calls several tools exposed as code. In this combined design, ordinary programming constructs become available across the tools: variables, loops, branches, filters, concurrency, caching, retries, and aggregation.

A concrete comparison helps:

```text
Code-as-tool only:
Model -> generated Python -> calculate over a local CSV -> result

Tool-as-code only:
Human-written program -> domain SDK or CLI -> remote capability

Combined:
Model -> generated program -> domain SDK A -> domain SDK B -> result
```

The difference can be summarized as follows:

| Model | What is primarily delivered | Where it operates |
|---|---|---|
| Skill | Information and guidance | Model context |
| Remote tool | A bounded capability | Remote service |
| Code-as-tool | A general code-execution capability | Agent runtime |
| Tool-as-code | A predefined tool exposed as a library, CLI, or script | Program-accessible environment |

These models are not mutually exclusive. Most practical agent systems combine them. The design question is which responsibility should be placed in each one.

## Why Combining Code-as-Tool with Tool-as-Code Is So Attractive

Code-as-tool is useful by itself for computation over local data. Tool-as-code is useful by itself for conventional programs. For agent systems, the particularly attractive design is their combination: the agent generates code, and that code can programmatically compose predefined domain capabilities.

The benefit is not simply that an agent can write code. The more important benefit is that deterministic work and tool orchestration can move out of the model loop.

Imagine that an agent needs to search several resources, retrieve the most relevant entries, filter their contents, and summarize the result. With discrete remote tools, the interaction may look like this:

```text
Model -> Search Tool -> Model -> Read Tool -> Model -> Filter Tool -> Model
```

The result of every operation returns to the model. The model reads the intermediate data, decides on the next call, and places another request. This is flexible, but the flexibility is paid for through repeated inference, tool-call latency, and context consumption.

If the same capabilities are available as code, the agent can instead generate a small program:

```text
Model -> Generated Program -> Search -> Read -> Filter -> Aggregated Result -> Model
```

Intermediate values remain inside the program. A loop can inspect one hundred results without asking the model to make one hundred decisions. Filtering and aggregation do not need to occupy the context window. Only the final result, or the small subset that requires judgment, needs to return to the model.

This matters particularly for complex queries. Tool schemas consume context. Tool results consume more context. Repeatedly asking the model what to do next also consumes tokens and creates more opportunities for an irrelevant intermediate result to distract the agent.

Their combination changes the division of labor:

> Let the model decide what the computation should accomplish. Let code perform the parts that are already deterministic.

For our project, this suggested an appealing direction. Almost every MCP capability could, in principle, be represented as a local code library: tool-as-code. The agent could then use code-as-tool to chain resource discovery, retrieval, and transformation without returning to the model between every step. New combinations would emerge from generated programs rather than from an ever-growing collection of narrowly specialized MCP tools.

But this design assumed that the local environment was capable of supporting it.

## The Environment That Changed the Decision

The users of our system work in a programming environment whose configuration is constrained. They cannot always install a new language runtime, add dependencies, change versions, or debug environment problems easily. Modifying the environment merely to support the agent could interfere with the environment they need for their actual work.

Under those conditions, telling the agent to generate a Python or JavaScript program is not enough. The program needs an appropriate runtime. Its dependencies must exist. Network access must be available. The user or product must be able to diagnose failures that come from the environment rather than from the program itself.

In a Unix-like environment, Bash looked like the lowest-friction code-as-tool runtime. It is widely available, easy for an agent to generate, and well suited to composing files, HTTP requests, and command-line programs exposed as tool-as-code. Pipes make simple chaining natural. A script can also remain visible and auditable instead of hiding the process behind a large framework.

But Bash does not make the environment disappear. Useful scripts may still depend on `curl`, `jq`, or other commands. Shell behavior differs across platforms. Error handling becomes difficult as the logic grows. Most importantly, a script that accesses a protected remote resource still needs a credential.

Bash was therefore not an abstract best practice. It was the least disruptive local option available under a particular set of constraints. Even that option could not solve the two deeper requirements of combining generated code with protected remote capabilities.

## The Two Hidden Requirements of the Combined Design

### Credentials: Who Is Allowed to Know the Secret?

If generated code needs to access a remote resource, it must authenticate somehow. This may require an API key, OAuth token, session credential, client certificate, or another form of user or service identity.

Once the credential is available to the code, the code runtime becomes part of the credential trust boundary.

That raises practical questions:

- Where is the credential stored?
- How is it injected into the process?
- Can generated code read credentials unrelated to the current task?
- Can a credential appear in logs, error messages, shell history, or model-visible output?
- How are credentials isolated across users and sessions?
- Who rotates, revokes, and audits them?
- Can the credential be restricted to only the resource and operation the task requires?

These are not secondary deployment details. They determine how much execution freedom the agent can safely receive.

A remote tool can keep the underlying credential behind the service boundary. The agent is allowed to invoke a specific capability, but it does not need direct access to the secret that makes that capability possible. A local code runtime provides greater composability, but safely giving it the same remote access requires a more sophisticated credential system.

The flexibility of code-as-tool comes from giving the runtime more control. That same control expands the credential attack surface.

### Runtime: Where Does the Code Actually Run?

Code is not an action by itself. It is a description of an action that must be executed somewhere.

A useful code runtime needs more than a language interpreter. It may need dependency management, filesystem access, network access, CPU and memory limits, timeouts, sandboxing, permission isolation, logs, cleanup, and a way to recover from failure.

Someone must own that infrastructure.

If it runs in the user's environment, the user inherits compatibility and debugging problems. If the product provides a local sandbox, the product must distribute and maintain it. If the sandbox is remote, code-as-tool has moved back toward a remotely managed execution service, with its own questions about credentials, state, and data transfer.

This led me to a simple conclusion:

> Code-as-tool does not remove infrastructure. It moves infrastructure into the agent runtime.

For a general-purpose coding agent running in a well-managed development environment, that may be a good trade. For users whose environments are constrained and difficult to modify, it may not be.

## Why We Kept the MCP Tools Remote

Given those constraints, we kept the MCP tools remote.

This allowed credentials to remain behind the service boundary. Dependencies and computation could be managed in one environment rather than reproduced in every user's environment. Updates could be deployed centrally. The same capabilities could be reused by different clients without requiring each client to install and maintain the implementation.

Remote execution also avoided consuming significant local computing resources. More importantly for us, it avoided making the user's configured programming environment part of the agent platform.

The choice had costs. Remote calls add network latency and another source of failure. Intermediate results often return to the model before the next action is selected. A query that would be a short loop in code may become several model-mediated tool calls. The granularity of the remote interface limits which combinations are easy to express.

We did not eliminate the composability problem. We changed where we addressed it.

## Using the Same Virtual Filesystem for Remote Resources

MCP tools can be called sequentially, but they are not natively composable from the agent's perspective in the same way as functions in a program or commands in a shell pipeline. Unless the client provides a programmatic orchestration layer, each call is usually mediated by the model: call a tool, observe the result, decide what to call next.

This makes remote tool design especially important. If every small operation becomes a separate MCP tool, the agent must discover a large tool catalog and repeatedly move through the model-tool loop. If one tool tries to encode an entire domain workflow, it becomes difficult to reuse and adapt.

For read-only resources, I found a useful middle ground: expose different operations over the same resource model through one tool.

We used the same virtual-filesystem mental model for these resources that we had used for skill discovery. We represented remote domain resources as if they were files and directories in a common namespace. The resources did not need to be stored in a literal filesystem. The filesystem was an interface and an ontology that the agent already understood.

The tool exposed several modes:

- `ls`: inspect the resources available under a path;
- `cat`: retrieve a known resource;
- `grep`: perform regular-expression search over resource contents;
- `find`: perform semantic search when the agent knows the meaning it is looking for but not the exact words or location.

This created a progressive exploration pattern:

```text
ls    -> understand the available structure
grep  -> locate an exact term or syntactic pattern
find  -> discover semantically related resources
cat   -> retrieve the selected resource in detail
```

The distinction between `grep` and `find` was deliberate. Regular-expression search is predictable and precise when the agent knows the vocabulary. Semantic search is useful when terminology varies or the agent is still discovering how the domain represents a concept. Both return paths into the same virtual namespace, so the next operation can use the result directly.

This design had several practical advantages.

First, it reduced the number of top-level tools the agent needed to understand. The agent learned one resource abstraction and a small set of familiar operations instead of many domain-specific query shapes.

Second, it separated discovery from retrieval. Listing and searching could return compact metadata or paths, while `cat` returned the detailed content only after the agent had narrowed the search space. This helped control tool-result size and context consumption.

Third, it made heterogeneous remote resources feel composable even though the calls were still model-mediated. Once documentation, metadata, examples, definitions, and other domain objects shared a namespace, the agent could apply the same exploration strategy to all of them.

Fourth, the abstraction was compatible with the habits of a coding agent. Coding agents already know how to navigate filesystems: inspect broadly, search, open a small number of relevant files, and refine the query. We could reuse that learned interaction pattern instead of teaching a new API for every resource category.

Using the same operations for skills and remote resources also reduced the number of interaction patterns the agent needed to learn:

```text
/skills/...       -> packaged knowledge and instructions
/resources/...    -> remote domain resources

ls / grep / find  -> discovery
cat               -> detailed retrieval
```

The contents and permissions differed, but the exploration grammar remained consistent. The agent could first understand the information landscape, then retrieve only the pieces required for the current decision.

The fact that these operations were read-only mattered. Combining `ls`, `grep`, `find`, and `cat` behind one interface did not create much ambiguity about side effects. Write operations have different safety and authorization requirements. Creating, modifying, deleting, approving, and publishing resources should not be hidden behind a vague mode parameter merely to reduce the number of tools. Their consequences need to remain explicit.

The virtual filesystem did not reproduce the full flexibility of local code. It did, however, recover much of the exploratory flexibility we needed while preserving a remote credential and execution boundary.

## Stateless Tools, Stateful Sessions

Keeping tools remote led to another architectural question: state.

An individual MCP tool call can be designed to be stateless. It receives a request, performs an operation, and returns a result. But the user is not making an isolated tool call. The user is using the system to complete a domain task over time.

As soon as that task begins, a session emerges. It may contain the user's intent, resources already discovered, intermediate artifacts, workflow stage, tool history, permissions, failures, retries, and decisions that later steps depend on.

> Stateless tools do not make a stateless system. State merely moves into the harness, the filesystem, or the remote service.

One option is to store state locally. Files are transparent, easy for a coding agent to inspect, and naturally connected to the user's project. Local state can work well when the filesystem is reliable, writable, and owned by the user.

But local state also depends on the local operating system and environment. It may be difficult to share across clients, restore on another device, or manage consistently across users. Cleanup, synchronization, and recovery become local responsibilities.

The other option is remote state. A remote service may use Redis, a database, or object storage to preserve session information and intermediate artifacts. This makes cross-client recovery, centralized lifecycle management, and shared observability easier. It also introduces tenant isolation, session identity, expiration, retention, permission, and consistency requirements.

Remote tools do not force all state to be remote. In practice, a layered design is often useful: project-local artifacts remain near the code, while remote workflow and integration state remain near the service. The important point is that state placement should be designed explicitly rather than left as an accidental consequence of tool calls.

## Choosing the Execution Boundary

There is no universal ranking in which code-as-tool is more advanced than remote tools, or remote tools are more robust than skills. They solve different parts of the system.

The choice depends on the capability:

- If the agent primarily needs information, judgment guidance, examples, or a reusable method, package it as a skill.
- If the capability is stable, permission-sensitive, remotely owned, or dependent on managed infrastructure, expose it as a remote tool.
- If the task requires dynamic composition and the runtime can be trusted and maintained, use code-as-tool.
- If a reusable capability should remain transparent and locally composable, and the user environment can support it, deliver it as code, a CLI, or a script.

Useful design questions include:

1. Does the capability provide information, execution, or both?
2. Does it require a credential, and can that credential safely enter the runtime?
3. Can the user's environment run and debug the required code?
4. How much programmatic chaining does the task require?
5. How large are the intermediate results?
6. Does the operation need centralized permissions, audit, or updates?
7. Where should the session recover after a failure?
8. Which state belongs to the user's project, and which belongs to the remote service?

Most real systems will answer these questions differently for different capabilities. An interconnected skill system may help the agent discover how to approach a domain. A virtual-filesystem MCP tool may expose both skill metadata and the domain's remote resources through a consistent exploration model. A local Bash script may inspect the user's project. A remote workflow service may preserve shared session state.

The system is not choosing one abstraction. It is placing responsibilities across several boundaries.

## Conclusion

Skills, tools, and code-as-tool may sound like basic agent concepts, but they determine much of what a more sophisticated harness can actually do.

A skill packages information, but a useful skill system also makes relationships between packages discoverable. A remote tool exposes a controlled capability. Code-as-tool gives the agent flexible execution and composition. Each choice places information, computation, credentials, dependencies, and state in a different part of the system.

For our project, local code offered attractive composition and token-efficiency benefits. But the user's constrained environment and the credential trust boundary made remote MCP tools a more practical choice. A virtual-filesystem interface then gave skills and remote resources the same familiar, metadata-first exploration model without moving credentials or runtime requirements into the user's environment.

This is why I do not think harness engineering has made skills and tools obsolete.

> Harness engineering is not separate from skills, tools, and code. A harness coordinates capabilities whose information, execution, credentials, computation, and state have already been placed somewhere.

Sometimes, however, the missing capability cannot—or should not—be placed in a skill, a remote service, or a code runtime. The agent may need information only the user knows, an action only the user can perform, feedback only a person can provide, or authorization that should never belong to the agent.

From the agent's local perspective, asking a human can look surprisingly similar to calling a tool. But that analogy has important limits. That is the subject of the next post.
