# SpatialReal skills

Skills that teach an AI coding assistant to build with [SpatialReal](https://docs.spatialreal.ai):
realtime 3D avatars that speak with matching face and body motion, rendered on the user's
device, and voice agents that listen, think and answer by themselves.

Install them once, then ask your assistant in plain words:

> Add a SpatialReal avatar to this page and make it speak the audio from my TTS.

> Build a SpatialReal video agent that answers questions about our pricing, and embed it in my app.

## The skills

| Skill | What your assistant learns |
| --- | --- |
| [`spatialreal`](skills/spatialreal/SKILL.md) | Putting an avatar or a voice agent into a **Web, iOS or Android** app: which kind of session to use, the server endpoint that issues session tokens, the SDK code for each platform, what to check before shipping, and where each topic is in the documentation |
| [`spatialreal-cli`](skills/spatialreal-cli/SKILL.md) | Managing the account from the terminal with the `spatialreal` command: signing in, linking a project and getting an API key, generating an agent from a description, editing agents as files, avatars, embed links, webhooks, usage and billing |

They follow the open [Agent Skills](https://agentskills.io) format (`SKILL.md`), so they work in
any assistant that supports it.

## Install

### With the SpatialReal CLI (recommended)

```bash
npm install -g @spatialreal/cli
```

That is all: a global install puts both skills in `~/.claude/skills` and `~/.agents/skills`,
where Claude Code, Codex, Cursor and Gemini CLI read them. It needs Node.js 20 or newer. To
install them into one project instead (for example to commit them for your team), run
`spatialreal skill install` in it; `spatialreal skill show` prints them for an assistant session
that was already running.

### Claude Code plugin

```
/plugin marketplace add SpatialReal-ai/skills
/plugin install spatialreal@spatialreal
```

### Any assistant, with the `skills` installer

```bash
npx skills add SpatialReal-ai/skills
```

### By hand

Copy the two folders under [`skills/`](skills) to where your assistant reads skills:

| Assistant | For one project | For every project |
| --- | --- | --- |
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| Codex, Cursor, Gemini CLI | `.agents/skills/` | `~/.agents/skills/` |

## What you need

- A SpatialReal account: [app.spatialreal.ai](https://app.spatialreal.ai). Your assistant signs
  you in through the browser the first time it uses the CLI.
- For your own app: a server that can keep an API key secret. The browser or mobile app never
  sees the key; it gets a short-lived session token from your server.

## Links

- Documentation: https://docs.spatialreal.ai
- CLI on npm: https://www.npmjs.com/package/@spatialreal/cli
- Web console: https://app.spatialreal.ai

## License

[MIT](LICENSE)
