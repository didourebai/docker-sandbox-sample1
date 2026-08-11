# From Desktop Files to AI Insights: Building a Sandboxed Data Analyst with Docker

AI agents are becoming increasingly capable of doing more than answering questions. They can inspect files, run commands, install tools, transform data, execute code, and combine information from multiple sources.

That creates an interesting architectural question:

**What should happen when an AI agent needs to work with files on a developer's machine?**

For a simple chatbot, the answer is easy: the model receives text and returns text.

For a data-analysis agent, the workflow is very different. A user may drop a folder containing CSV, Excel, PDF, JSON, XML, and text files and ask:

> "Analyze these files, cross-reference the information, and extract the key findings."

The agent now needs tools and an execution environment.

This is where Docker Sandboxes become interesting.

Docker describes Sandboxes as isolated microVM environments for AI coding agents. Each sandbox has its own filesystem, Docker daemon, and network. Docker also provides SSH-based integrations so external editors and desktop applications can connect to a running sandbox. 

In this article, I will build a small desktop proof of concept: an **AI Data Analyst** that accepts files with different extensions, stages them into a temporary workspace, creates a Docker Sandbox, and asks Claude Code to analyze the data inside that sandbox.

The goal is not to build another chatbot.

The goal is to explore an architectural pattern:

**Desktop application → AI agent → isolated execution environment → extracted data**

---

## 1. Why put the agent in a sandbox?

Consider a desktop application that receives:

```text
customer-data/
├── customers.xlsx
├── transactions.csv
├── contracts.pdf
├── configuration.json
└── notes.txt
```

The user asks:

> "Find customers with active contracts whose purchases exceed $10,000 and summarize the relevant account information."

A capable agent may need to:

1. inspect the directory;
2. identify the file formats;
3. parse the Excel workbook;
4. process the CSV;
5. extract information from the PDF;
6. parse the JSON;
7. read the text file;
8. join the datasets;
9. calculate totals;
10. produce a structured answer.

This is fundamentally different from sending the files as plain text to an LLM.

The agent needs an execution environment.

That environment may require Python packages, command-line utilities, temporary files, and potentially network access.

Giving an autonomous agent unrestricted access to the host machine is therefore a poor architectural boundary.

Docker Sandboxes address this by running supported agents inside isolated microVMs. Docker documents five isolation layers: hypervisor, network, Docker Engine, workspace, and credential isolation. 

The important point is not that a sandbox makes an AI agent "safe."

It gives the agent a **defined execution boundary**.

---

## 2. The architecture

The proof of concept uses this architecture:

```text
                         USER
                           |
                           v
                +---------------------+
                |    Desktop App      |
                |                     |
                |  Select files       |
                |  Ask question       |
                +----------+----------+
                           |
                           v
                +---------------------+
                |  Temporary staging  |
                |      workspace      |
                +----------+----------+
                           |
                     sbx create
                           |
                           v
        +-------------------------------------+
        |          Docker Sandbox             |
        |                                     |
        |        Claude Code Agent             |
        |                                     |
        |  CSV / XLSX / PDF / JSON / XML      |
        |  Python / CLI tools / file system   |
        +----------------+--------------------+
                         |
                         | SSH
                         v
                +---------------------+
                |    Desktop App      |
                |                     |
                | Structured result   |
                +---------------------+
```

There are two important boundaries here.

The first is the **AI boundary**:

```text
Desktop Application
        |
        v
     AI Agent
```

The second is the **execution boundary**:

```text
AI Agent
    |
    v
Docker Sandbox
```

The model decides what needs to be done. The sandbox defines where those operations execute.

---

## 3. Why SSH?

Docker Sandboxes now support connecting external editors and desktop applications over SSH. The integration is GA and requires Docker Sandboxes 0.37.0 or later. 

After running:

```bash
sbx setup ssh
```

a managed SSH configuration is created.

A sandbox named `data-analyst` can then be reached as:

```bash
ssh data-analyst.sbx
```

There is an important detail here: this is not a traditional SSH server running inside the sandbox.

Docker's documentation explains that the sandbox daemon handles the SSH connection via a local proxy. Authentication is tied to the active Docker login rather than a stored SSH key. 

That makes the integration particularly useful for applications that already understand SSH.

---

## 4. Creating the sandbox

Docker supports multiple agents, including Claude Code, Codex, Copilot, Cursor, Gemini, OpenCode, Docker Agent, and a shell-only sandbox. 

For this demonstration, I use Claude Code because Docker provides a dedicated Claude Code sandbox agent.

The basic command is:

```bash
sbx create claude --name data-analyst /path/to/workspace
```

Docker documents that the workspace is mounted into the sandbox at the same path. citeturn3search4

That behavior is useful for development, but it is an important consideration for our data-analysis scenario.

The workspace is read-write.

Therefore, I don't point the demo directly at the user's original folder.

Instead, the desktop application creates a temporary staging directory and copies the selected files into it.

This gives us:

```text
Original files
     |
     | copy
     v
Temporary staging directory
     |
     v
Docker Sandbox
```

The original files are therefore not the agent's working directory.

---

## 5. The desktop application

The proof of concept is implemented in Python using Tkinter.

The UI is intentionally simple:

```text
+--------------------------------------------------+
| Docker Sandbox AI Data Analyst                   |
|                                                  |
| [Add files] [Clear]                              |
|                                                  |
| customers.xlsx                                   |
| transactions.csv                                 |
| contracts.pdf                                    |
| configuration.json                               |
|                                                  |
| Question                                         |
| +----------------------------------------------+ |
| | Analyze these files and identify...         | |
| +----------------------------------------------+ |
|                                                  |
| [Analyze in Docker Sandbox]                      |
|                                                  |
| Activity / result                                |
| +----------------------------------------------+ |
| |                                              | |
| |                                              | |
| +----------------------------------------------+ |
+--------------------------------------------------+
```

The application has four responsibilities:

1. collect files from the user;
2. create a temporary staging workspace;
3. create/connect to the Docker Sandbox;
4. send the analysis request to the agent.

It does **not** implement CSV parsing, PDF extraction, Excel processing, or JSON analysis itself.

That is deliberate.

The purpose of the experiment is to see whether an agent can orchestrate those capabilities inside the sandbox.

---

## 6. The core code

The application creates the sandbox with:

```python
runner.run([
    "sbx", "create", "claude",
    "--name", sandbox_name,
    str(workspace),
])
```

Then it connects through the Docker Sandbox SSH integration:

```python
result = runner.run([
    "ssh", f"{sandbox_name}.sbx",
    "claude", "-p", prompt,
    "--output-format", "text",
])
```

Claude Code supports non-interactive print mode through `claude -p`, which makes it suitable for this kind of application-driven workflow. citeturn3search12

This gives us a simple application integration:

```text
Python
  |
  +-- sbx create
  |
  +-- ssh <sandbox>.sbx
             |
             +-- claude -p
```

The complete source code is available with this article.

---

## 7. The agent prompt

The prompt is intentionally designed as an orchestration instruction rather than a file-format-specific algorithm.

The agent receives instructions such as:

```text
Inspect the available files and identify their formats.

Extract relevant data from CSV, Excel, PDF, JSON, XML,
TXT and other supported formats when present.

Prefer deterministic parsing tools where appropriate.

Cross-reference information across files.

Do not invent facts.

Return:
- Files inspected
- Key findings
- Important assumptions or limitations
- Structured extracted data
```

Notice what is missing.

There is no hard-coded:

```text
if extension == ".csv":
    ...
elif extension == ".xlsx":
    ...
elif extension == ".pdf":
    ...
```

The agent determines which tools are appropriate.

That is where the example becomes genuinely agentic.

---

## 8. A real analysis scenario

Imagine these files:

```text
customers.xlsx
transactions.csv
contracts.pdf
notes.txt
```

The user asks:

> "Which customers have an active contract and more than $10,000 in transactions? Return the customer name, contract status, total transaction value, and relevant notes."

The agent can break this down into a workflow:

```text
                    User question
                         |
                         v
                  Understand task
                         |
                         v
                  Inspect files
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
       XLSX             CSV            PDF
          |              |              |
          +--------------+--------------+
                         |
                         v
                   Cross-reference
                         |
                         v
                    Calculate
                         |
                         v
                  Structured result
```

The important architectural observation is that **the agent is orchestrating tools**, while the sandbox provides the place where those tools execute.

---

## 9. What makes this different from a normal LLM application?

A conventional application might do this:

```text
Files
  |
  v
Application parser
  |
  v
Text extraction
  |
  v
LLM
  |
  v
Answer
```

That can work very well.

But it requires the application developer to explicitly implement the processing pipeline.

The agentic approach looks more like:

```text
Files
  |
  v
AI Agent
  |
  +--> choose parser
  +--> execute tool
  +--> inspect result
  +--> choose next operation
  +--> combine results
  |
  v
Answer
```

This is more flexible, but it also creates a much larger trust boundary.

That's precisely why the sandbox matters.

---

## 10. Network access is another architectural boundary

Filesystem isolation is only part of the story.

An agent may also want to access:

- package repositories;
- model APIs;
- GitHub;
- external APIs;
- cloud services.

Docker Sandboxes provide network policies. Docker documents three starting policies: Open, Balanced, and Locked Down. 

For example:

```bash
sbx policy ls
```

and a specific host can be allowed with:

```bash
sbx policy allow network registry.npmjs.org
```

For a production design, I would avoid giving a data-processing agent unrestricted outbound access.

A better model is:

```text
                 AI Agent
                    |
                    v
              Docker Sandbox
                    |
             Network Policy
              /          \
          allowed       blocked
          services      services
```

The principle is simple:

**The agent should have the minimum network access required to perform its task.**

---

## 11. What about credentials?

This is another reason not to simply give an agent access to the host environment.

Docker Sandboxes provide credential handling where the real credentials remain on the host and the sandbox receives the necessary credential through the proxy mechanism. Docker documents this model for API providers and other supported services. citeturn1search5

For example:

```bash
sbx secret set -g anthropic
```

The exact authentication mechanism depends on the agent and provider.

For an enterprise implementation, I would treat credentials as a separate architectural concern:

```text
Identity
   |
   v
Credential broker
   |
   v
Sandbox
   |
   v
Specific external service
```

The agent should not receive a general-purpose collection of long-lived secrets.

---

## 12. One important security caveat

A sandbox is not a magic security button.

In particular, Docker's default workspace behavior matters.

The workspace can be shared read-write with the host. Docker explicitly documents that the workspace is the part of the host filesystem the agent can see in the normal direct workspace mode. citeturn0search1turn0search11

That's why this demo uses a **staging copy**.

The pattern is:

```text
User data
   |
   | copy
   v
Temporary workspace
   |
   v
Sandbox
```

rather than:

```text
Entire user home directory
          |
          v
       Sandbox
```

For Git-based development workflows, Docker also provides clone mode, which keeps the agent's changes inside a sandbox-side clone. citeturn3search6

For arbitrary document analysis, staging is a simple and understandable pattern.

---

## 13. Why this is an interesting desktop architecture

The desktop application doesn't need to know how to parse every file type.

It becomes an orchestration layer:

```text
                 Desktop App
                      |
             "Analyze these files"
                      |
                      v
                 AI Agent
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
      Excel          PDF           CSV
      parser        parser        parser
        |             |             |
        +-------------+-------------+
                      |
                      v
               Structured data
```

This pattern could be extended beyond data analysis.

For example:

### Document intelligence

```text
PDF + Word + Excel
        ↓
AI Agent
        ↓
Extract entities
        ↓
Structured JSON
```

### Compliance analysis

```text
Policies + procedures + evidence
              ↓
          AI Agent
              ↓
       identify gaps
              ↓
        structured report
```

### Financial analysis

```text
Invoices + CSV + Excel
          ↓
       AI Agent
          ↓
   reconciliation
          ↓
      exceptions
```

### Developer workflows

```text
Source code + logs + configuration
              ↓
          AI Agent
              ↓
        diagnose issue
              ↓
          proposed fix
```

The common pattern is the same:

**AI reasoning + executable tools + controlled runtime.**

---

## 14. Where Claude fits

Claude is not the sandbox.

This distinction is important.

Docker Sandboxes are the isolation layer. Docker currently supports several agents, including Claude Code, Codex, Copilot, Cursor, Gemini, OpenCode, Docker Agent, and others. 

Conceptually:

```text
                  Docker Sandbox
                        |
       +----------------+----------------+
       |                |                |
    Claude            Codex           Gemini
       |                |                |
       +----------------+----------------+
                        |
                 isolated runtime
```

That means the architecture is not fundamentally tied to one model provider.

For this proof of concept, Claude Code is simply the agent selected for the implementation.

---

## 15. Could the model run locally?

Yes.

This is one of the most interesting extensions.

Docker documents a configuration where Claude Code runs inside a Docker Sandbox while Docker Model Runner provides a local model on the host through an Anthropic-compatible API. citeturn3search1

The architecture becomes:

```text
                    Desktop App
                         |
                         v
                  Docker Sandbox
                         |
                    Claude Code
                         |
                         v
                host.docker.internal
                         |
                         v
                Docker Model Runner
                         |
                         v
                    Local LLM
```

That creates a very interesting local AI scenario:

**The agent executes inside an isolated microVM while model traffic remains on the local machine.**

This could be the next experiment.

---

## 16. Lessons learned

This small project highlights several architectural lessons.

### Lesson 1 — Agents need an execution boundary

Once an AI agent can execute code, the execution environment becomes part of the architecture.

### Lesson 2 — Sandboxing is about blast radius

The objective isn't to make the agent inherently trustworthy.

The objective is to limit what happens when the agent does something unexpected.

### Lesson 3 — Filesystem access needs deliberate design

Don't casually mount an entire home directory into an autonomous agent.

Use the smallest workspace possible.

For arbitrary documents, a staging directory is a useful pattern.

### Lesson 4 — Network access matters

An isolated filesystem with unrestricted network access is still a significant trust boundary.

Network policies should be part of the design.

### Lesson 5 — Credentials are architecture

Don't treat API keys and service credentials as environment variables that magically appear inside an agent.

Define how they are stored, injected, scoped, rotated, and audited.

### Lesson 6 — The model and the runtime are separate concerns

Claude, Codex, Gemini, and other agents are replaceable components.

The execution boundary is a separate architectural decision.

---

# 17. The bigger picture

The most interesting evolution isn't:

> "AI can read my files."

We've had document-processing AI for years.

The more interesting evolution is:

> **AI agents can decide what tools to execute against those files.**

That changes the architecture.

We move from:

```text
Input → Model → Output
```

to:

```text
Input
  ↓
Agent
  ↓
Plan
  ↓
Tools
  ↓
Execution
  ↓
Observation
  ↓
Next action
  ↓
Result
```

Once that happens, **where the agent executes becomes just as important as which model it uses.**

That's the architectural value of Docker Sandboxes.

---

# Conclusion

A desktop AI data analyst is a relatively small example, but it exposes a much bigger architectural pattern.

The desktop application provides the user experience.

The AI agent provides reasoning and orchestration.

The tools provide deterministic execution.

The Docker Sandbox provides an isolated execution environment.

And the network and credential policies define what the agent is allowed to reach.

The result is not simply:

**"Chat with your files."**

It is:

**"Give an AI agent controlled access to an execution environment where it can inspect, transform, and analyze data."**

And that leads to the question I think is worth exploring next:

> **If AI agents are becoming capable of executing arbitrary workflows, should the default architecture be to run them directly on our machines — or to give every agent its own controlled runtime?**

For me, the second option is becoming increasingly compelling.

---

## Try the demo

The accompanying proof of concept contains:

```text
docker-sandbox-ai-data-analyst/
├── README.md
├── article.md
├── src/
│   └── app.py
└── sample-data/
```

Run:

```bash
python src/app.py
```

Then select your files, enter an analysis question, and click:

**Analyze in Docker Sandbox**

Before using real or sensitive data, review the Docker Sandbox filesystem, network, and credential policies for your environment. Docker's documentation should be treated as the source of truth for current commands and supported integrations. citeturn2view0turn0search9turn1search5
