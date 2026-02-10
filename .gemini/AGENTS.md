# Agent Rules

## Style

- Do not use emojis anywhere -- not in code, commit messages, documentation, file content, or responses.
- Write in clear, concise English. Avoid filler phrases and unnecessary verbosity.
- Use consistent markdown formatting: `##` for sections, `###` for subsections, `-` for lists.

## Content

- All interview questions must include a clear, accurate answer.
- Provide practical examples and real commands where applicable (e.g., actual `systemctl`, `docker`, `kubectl` commands).
- Distinguish between general/conceptual questions and scenario-based questions.
- When covering tools (Docker, Kubernetes, CI/CD, etc.), include common flags and options a candidate should know.
- Keep answers interview-appropriate in length -- detailed enough to demonstrate knowledge, but not essay-length.

## Structure

- Each topic lives in its own directory under the project root.
- Use the root `README.md` only as a top-level index; do not put question content directly in it.
- Every topic directory must follow this layout:

| File | Purpose |
|------|---------|
| `README.md` | Topic overview, subtopics covered, links to sub-files |
| `questions.md` | General/conceptual interview questions with answers |
| `scenarios.md` | Scenario-based and troubleshooting questions |
| `notes.md` | Quick-reference notes, commands, and cheat sheets |
| `examples/` | Only where applicable (scripting, terraform) for runnable code |

- Directory naming: lowercase, hyphens for multi-word names (e.g., `containers-and-kubernetes`).
- Keep each file focused on a single concern; do not mix questions and notes in the same file.

## Accuracy

- Do not fabricate commands or flags. Every command must be valid and runnable.
- Specify the OS/distro context when a command is distro-specific (In here we are biased to `RHEL`).
- If a question involves configuration files, use realistic file paths and syntax.

## Workflow

- Topics are not covered in the order listed in the README. The sequence is driven by priority and preference.
- Current starting point: **Containers & Kubernetes**.
