# bookops skills

Claude Code plugin with two skills by akukiki (https://akukiki.com):

- `akukiki` — a production audit of your project in plain words; it only reads the project, and anything it sends or changes needs your explicit yes;
- `akukiki-telemetry` — connects your app's errors and logs to akukiki, only with your yes.

Install into your agent (Claude Code, Codex, Cursor, OpenCode, Gemini CLI, OpenClaw, Hermes and others), in a terminal:

```
npx skills add bookops/skills
```

In Claude Code you can install it as a plugin instead, which updates itself:

```
/plugin marketplace add bookops/skills
/plugin install akukiki@bookops
```

Then ask your agent: "check my project with akukiki".

This repository is generated from the skills that akukiki.com serves; changes here are overwritten.
