# ElarisLabs Skills

Agent skills for [ElarisLabs](https://studio.elarislabs.ai) Creative Studio — reusable workflows that drive the ElarisLabs MCP tools from Cursor, Claude, or any MCP client.

## Skills

| Skill | What it does |
|-------|----------------|
| [`static-to-video`](skills/static-to-video/) | Seed still → optional AR reframe / close-ups → I2V (Seedance) → HyperFrames first+end cards + instrumental BGM (no VO) |

## Install (Cursor)

### Option A — clone into personal skills

```bash
git clone https://github.com/1sarthakbhardwaj/elarislabs-skills.git ~/elarislabs-skills
mkdir -p ~/.cursor/skills
ln -sfn ~/elarislabs-skills/skills/static-to-video ~/.cursor/skills/static-to-video
```

### Option B — project skill (one repo)

```bash
mkdir -p .cursor/skills
ln -sfn /path/to/elarislabs-skills/skills/static-to-video .cursor/skills/static-to-video
```

Then invoke with `/static-to-video` or ask the agent to run the **static-to-video** skill.

## Prerequisites

1. An [ElarisLabs API key](https://studio.elarislabs.ai/en/api-keys) bound to a brand.
2. ElarisLabs MCP configured in Cursor (`~/.cursor/mcp.json`) or Claude Desktop:

```json
{
  "mcpServers": {
    "elarislabs": {
      "url": "https://studio.elarislabs.ai/api/mcp",
      "headers": {
        "Authorization": "Bearer elx_live_YOUR_KEY_HERE"
      }
    }
  }
}
```

Docs: [studio.elarislabs.ai/mcp-docs.html](https://studio.elarislabs.ai/mcp-docs.html)

## Repo layout

```
skills/
  static-to-video/
    SKILL.md              # Agent instructions
    reference.md          # MCP calls + model IDs
    assets/
      end-screen.html     # Branded outro template
```

## Contributing

Add a new skill under `skills/<skill-name>/` with a `SKILL.md` (YAML frontmatter: `name`, `description`). Keep the body under ~500 lines; put long reference material in sibling files.

## License

MIT — see [LICENSE](LICENSE).
