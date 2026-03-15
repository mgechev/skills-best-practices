# Best Practices for Creating Agent Skills

This guide explains how to write professional-grade skills for agents, validate them using LLMs, and maintain a lean context window.

This guide is a concentrated set of best practices for creating agent skills. If you're looking for a comprehensive documentation see [Claude's docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices).

**To evaluate if your skills do well and prevent regressions, check out [skillgrade]((https://github.com/mgechev/skillgrade).**

## Structure of a skill

Every skill must follow this directory structure:

Plaintext

```
skill-name/
├── SKILL.md              # Required: Metadata + core instructions (<500 lines)
├── scripts/              # Executable code (Python/Bash) designed as tiny CLIs
├── references/           # Supplementary context (schemas, cheatsheets) 
└── assets/               # Templates or static files used in output
```

* **SKILL.md:** Acts as the "brain." Use it for navigation and high-level procedures.  
* **References:** Link directly from SKILL.md. Keep them **one level deep** only.  
* **Scripts:** Use for fragile/repetitive operations where variation is a bug. **Do not bundle library code here**;

## Optimize the frontmatter for discoverability

The `name` and `description` in the frontmatter of your `SKILL.md` are the only fields that the agent sees before triggering a skill. If they are not optimized for discoverability and specific enough, your skill is invisible.

* **Adhere to Strict Naming:** The name field must be 1-64 characters, contain only lowercase letters, numbers, and hyphens (no consecutive hyphens), and **must exactly match the parent directory name** (e.g., name: `angular-testing` must live in `angular-testing/SKILL.md`).  
* **Write Trigger-Optimized Descriptions:** (Max 1,024 characters). This is the only metadata the agent sees for routing. Describe the capability in the third person and include "negative triggers."  
  * **Bad:** "React skills." (Too vague).
  * **Good:** "Creates and builds React components using Tailwind CSS. Use when the user wants to update component styles or UI logic. Don't use it for Vue, Svelte, or vanilla CSS projects."

## Progressive disclosure and resource management

Maintain a pristine context window by loading information only when needed. **SKILL.md** is the "brain" for high-level logic; offload details to subdirectories.

* **Keep SKILL.md Lean:** Limit the main file to **\<500 lines**. Use it for navigation and primary procedures.  
* **Use Flat Subdirectories:** Move bulky context to standard folders. Keep files exactly **one level deep** (e.g., `references/schema.md`, not `references/db/v1/schema.md`).  
  * `references/`: API docs, cheatsheets, domain logic.  
  * `scripts/`: Executable code for deterministic tasks.  
  * `assets/`: Output templates, JSON schemas, images.  
* **Just-in-Time (JiT) Loading:** Explicitly instruct the agent when to read a file. It will not see these resources until you direct it to (e.g., *"See `references/auth-flow.md` for specific error codes"*).  
* **Explicit Pathing:** Always use **relative paths** with forward slashes (`/`), regardless of the OS.

Skills are for agents, not humans. To keep the context window lean and avoid unnecessary token consumption. **Do not create:**

* **Documentation files:** `README.md`, `CHANGELOG.md`, or `INSTALLATION_GUIDE.md`.  
* **Redundant logic:** If the agent already handles a task reliably without help, delete the instruction.  
* **Library code:** Skills should reference existing tools or contain tiny, single-purpose scripts. Long-lived library code belongs in standard repo CLI directories.

## Use specific procedural instructions instead of prose

Create instructions for LLMs instead of humans.

* **Use Step-by-Step Numbering:** Define the workflow as a strict chronological sequence. If there is a decision tree, map it out clearly (e.g., *"Step 2: If you need source maps run `ng build --source-map`. Otherwise, skip to Step 3."*).  
* **Provide Concrete Templates:** Agents pattern-match exceptionally well. Instead of spending paragraphs describing how a JSON output should look, place a template in the assets/ folder and instruct the agent to copy its structure.  
* **Write in the Third-Person Imperative:** Frame instructions as direct commands to the agent (e.g., *"Extract the text..."* rather than *"I will extract..."* or *"You should extract..."*).

Be specific and consistent in the way you reference concepts in your skill files.

* **Use identical terminology:** Pick a single term to refer to a specific concept.  
* **Specificity**: Use the most specific terminology that’s native to the domain that you describe. For example, in Angular use the concept “template” instead of “html”, “markup”, or “view”.

## Bundle deterministic scripts for repetitive operations

Don't ask the LLM to write complex parsing logic or boilerplate code from scratch every time it runs a skill.

* **Offload fragile/repetitive tasks:** If the agent needs to parse a complex dataset or query a specific database, give it a tested Python, Bash, or Node script to run in the scripts/ directory.  
* **Handle edge cases gracefully:** An agent relies on standard output (stdout/stderr) to know if a script succeeded. Write scripts that return highly descriptive, human-readable error messages so the agent knows exactly how to self-correct without needing user intervention.

# Validation Guide

Since LLMs will be using your skills, the best way I’ve identified to ensure they are useful is in collaboration with LLMs.

It’s critical to have evals for your skills to make sure the changes you’re making have a positive impact and don’t lead to regression. A popular benchmark for skills is [SkillsBench](https://arxiv.org/abs/2602.12670) which could help with some inspiration.

Once you draft the initial version of your skills, you can validate your work going through the following steps:

### Discovery Validation

Agents load skills based strictly on the YAML frontmatter. Test how an LLM interprets your description in isolation to prevent false triggers (like firing for a React app when it's meant for Angular).

Paste exactly the text below into a fresh LLM chat:

> I am building an Agent Skill based on the agentskills.io spec. Agents will decide whether to load this skill based entirely on the YAML metadata below.
>
> ```
> name: angular-vite-migrator
> description: Migrates Angular CLI projects from Webpack to Vite and esbuild. Use when the user wants to update builder configurations, replace webpack plugins with rollup equivalents, or speed up Angular compilation.
> ```
>
> Based strictly on this description:
> 1. Generate 3 realistic user prompts that you are 100% confident should trigger this skill.
> 2. Generate 3 user prompts that sound similar but should NOT trigger this skill (e.g., migrating a React app to Vite, or just updating Angular versions).
> 3. Critique the description: Is it too broad? Suggest an optimized rewrite.

In addition, prompt agents with assignments that you expect to trigger a skill read and inspect the thought process. Go back and forth with the agent to identify why it picked (or didn’t) particular skills.

### Logic Validation

Ensure your step-by-step instructions are deterministic and don't force the agent to hallucinate missing steps.

Feed the LLM your entire `SKILL.md` and directory structure:

> Here is the full draft of my SKILL.md and the directory tree of its supporting files.
> 
> ```
> ├── SKILL.md
> ├── scripts/esbuild-optimizer.mjs
> └── assets/vite.config.template.ts
> ```
> 
> [Paste your SKILL.md contents here]
> Act as an autonomous agent that has just triggered this skill. Simulate your execution step-by-step based on a request to Migrate my Angular v17 app to Vite.
>
> For each step, write out your internal monologue:
> 1. What exactly are you doing?
> 2. Which specific file/script are you reading or running?
> 3. Flag any Execution Blockers: Point out the exact line where you are forced to guess or hallucinate because my instructions are ambiguous (e.g., how to map Angular environment files to Vite's import.meta.env).

### Edge Case Testing

Force the LLM to hunt for vulnerabilities, unsupported configurations, and failure states inherent to web tooling.

Ask the LLM to attack your logic:

> Now, switch roles. Act as a ruthless QA tester. Your goal is to break this skill.
> Ask me 3 to 5 highly specific, challenging questions about edge cases, failure states, or missing fallbacks in the SKILL.md. Focus on:
>
> * What if `scripts/esbuild-optimizer.mjs` fails due to a legacy CommonJS dependency?
> * What if the user's `angular.json` contains heavily customized Webpack builders (`@angular-builders/custom-webpack`) that Vite doesn't support?
> * Are there implicit assumptions I made about the user's Node environment?
>
> Do not fix these issues yet. Just ask me the numbered questions and wait for me to answer them.

### Architecture Refinement

LLMs often try to stuff large config files directly into the main prompt. Use this step to enforce progressive disclosure and shrink the token footprint.

Have the LLM apply your fixes and restructure the skill:

> Based on my answers to your edge-case questions, rewrite the SKILL.md file, strictly enforcing the Progressive Disclosure design pattern:
>
> 1. Keep the main `SKILL.md` strictly as a high-level set of steps using third-person imperative commands (e.g., Execute the esbuild script, Read the Vite config template).
> 2. If there are dense rules, large `vite.config.ts` templates, or complex `angular.json` schemas currently in the file, remove them. Tell me to create a new file in `references/` or `assets/`, and replace the text in `SKILL.md` with a strict command to read that specific file only when needed.
> 3. Add a dedicated Error Handling section at the bottom incorporating my answers about Webpack fallbacks and CommonJS resolution.
