# SpatialReal skills

Guidance for AI coding assistants that build with SpatialReal realtime 3D avatars and voice
agents. Two skills:

| Skill | What it teaches |
| --- | --- |
| `spatialreal` | Putting an avatar or a voice agent into a Web, iOS or Android app: which kind of session to use, the server endpoint that issues session tokens, the SDK code, what to check before shipping, and where each part of the documentation is |
| `spatialreal-cli` | Using the `spatialreal` command-line tool to manage the account: log in, link a project, generate and edit agents, list avatars, embed links, webhooks, billing |

## Install

Any coding assistant that supports skills (Claude Code, Codex, Cursor, Gemini CLI and others):

```bash
npx skills add SpatialReal-ai/skills
```

With the SpatialReal CLI (`npm install -g @spatialreal/cli`), which carries the same two files:

```bash
spatialreal skill install --global
```

Claude Code, as a plugin:

```
/plugin marketplace add SpatialReal-ai/skills
/plugin install spatialreal@spatialreal
```

Or copy the `SKILL.md` files under `plugins/spatialreal/skills/` to where your assistant reads
its instructions.

Documentation: https://docs.spatialreal.ai
