# Agent team

I am using GitHub Copilot CLI in a Codespace to orchestrate the work of building Mona's Project Pulse dashboard. The CLI is launched with `copilot --allow-all --enable-all-github-mcp-tools`, and every custom agent definition lives under the `.github/agents/` folder in this repository.

## The team

| Agent | Model | Responsibility | Definition file |
| --- | --- | --- | --- |
| Orchestrator | Claude Opus 4.7 (copilot) | Breaks my request into phases and delegates to the specialists. Assigns explicit file scopes, decides what runs in parallel versus sequentially, verifies the integrated result, and reports the outcome. Does not implement anything itself. | `.github/agents/orchestrator.agent.md` |
| Planner | Claude Opus 4.7 (copilot) | Researches the repository, docs, dependencies, and edge cases, then returns an ordered implementation plan with file assignments, dependencies, parallel/sequential guidance, and validation expectations. Writes no code. | `.github/agents/planner.agent.md` |
| Coder | GPT-5.5 (copilot) | Implements dashboard logic and fixes bugs inside its assigned file scope. Also creates runnable-app support such as `.vscode/launch.json` in strict JSON, with `cwd` set to `${workspaceFolder}/app` and `index.html` opened by default. | `.github/agents/coder.agent.md` |
| Designer | Gemini 3.1 Pro (copilot) | Owns UI/UX, accessibility, information hierarchy, and responsive layout. Delivers a polished dashboard with project cards, status badges, priority treatment, and deterministic CSS hooks such as `.dashboard` and `.project-card`. | `.github/agents/designer.agent.md` |

## How the team will build Project Pulse

1. I give the Orchestrator the Project Pulse goal from Copilot CLI.
1. The Orchestrator asks the Planner for a technical plan.
1. The Planner researches the repo and returns ordered steps with file ownership so scopes never collide.
1. The Orchestrator turns the plan into phases and delegates to the Coder and Designer, running them in parallel when their file scopes do not overlap and sequentially when work depends on earlier output.
1. The Coder builds the dashboard behavior in `app/` plus the `.vscode/launch.json` launch configuration; the Designer builds the visual and accessibility layer.
1. The Orchestrator verifies that the integrated dashboard hangs together and summarizes the result back to me.

No agent stages, commits, or pushes changes. I control all git operations myself through Copilot CLI prompts.
