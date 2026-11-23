# mindwork-chat-skills

Agent skills (`SKILL.md` bundles) for MindWork's chat agents. MindWork' chat mirrors this repository with GitHub Skill Sync, so everything under `skills/` becomes a read-only skill for agents.

## MindWork' chat setup

```yaml
skillSync:
  github:
    enabled: true
    intervalMinutes: 60
    runOnStartup: true
    sources:
      - id: mindwork-chat-skills
        owner: MindWork-AI
        repo: mindwork-chat-skills
        ref: main
        paths:
          - skills
        skillDiscoveryDepth: 2
        token: '${GITHUB_SKILLS_TOKEN}'
```

`GITHUB_SKILLS_TOKEN` is a fine-grained PAT with read-only Contents and Metadata.