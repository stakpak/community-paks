# Stakpak Paks

Official collection of rulebook-based paks maintained by the Stakpak team. These paks provide standardized procedures, best practices, and operational guides for cloud infrastructure, DevOps, and software deployment.

## What are Paks?

Paks are self-contained knowledge packages that bundle rulebooks—structured operational procedures and guidelines—for use with Stakpak tooling. Each pak contains:

- **`pak.toml`** — Metadata including name, version, description, keywords, and content references
- **`rulebooks/`** — Markdown files containing the actual procedures, guides, and best practices

## Available Paks

| Pak | Description |
|-----|-------------|
| **[aws-architecture-design](./aws-architecture-design)** | Standardize how to design and optimize AWS architectures for greenfield and brownfield projects |
| **[aws-sso-auth-guide](./aws-sso-auth-guide)** | AWS SSO discovery, configuration, and terminal usage with IAM Identity Center |
| **[confighub-usage-guide](./confighub-usage-guide)** | Manage Kubernetes configuration across multiple environments using Configuration as Data principles |
| **[dockerization](./dockerization)** | Official Stakpak application containerization standard operating procedure |
| **[documentation-rulebook](./documentation-rulebook)** | Standard deployment procedures for production |
| **[how-to-write-rulebooks](./how-to-write-rulebooks)** | Step-by-step guide on authoring, structuring, and publishing rulebooks in Stakpak |
| **[infrastructure-cost-estimation](./infrastructure-cost-estimation)** | Cloud cost calculations and resource discovery for AWS/GCP/Azure with FinOps best practices |
| **[infrastructure-software-upgrades](./infrastructure-software-upgrades)** | Generic guidelines for performing infrastructure component upgrades reliably |
| **[migrating-bitnami-to-bitnami-legacy](./migrating-bitnami-to-bitnami-legacy)** | Migrate Bitnami Helm charts and container images to the bitnamilegacy repository |
| **[nextjs-on-aws](./nextjs-on-aws)** | Compare AWS deployment options for Next.js applications with cost analysis and recommendations |
| **[simple-deployment-on-vm](./simple-deployment-on-vm)** | Simple but secure deployments using virtual machines on different cloud providers |

## Pak Structure

Each pak follows a consistent structure:

```
<pak-name>/
├── pak.toml                    # Pak metadata and configuration
└── rulebooks/
    └── <pak-name>.md           # Rulebook content
```

### pak.toml Schema

```toml
[pak]
name = "pak-name"
version = "1.0.0"
description = "Brief description of the pak"
authors = ["Stakpak <team@stakpak.dev>"]
license = "MIT"
repository = "https://github.com/stakpak/paks"
path = "pak-name"
tags = ["keyword1", "keyword2"]
visibility = "public"

[contents]
[[contents.rulebooks]]
path = "rulebooks/pak-name.md"
description = "Description of the rulebook"
tags = ["tag1", "tag2"]
```

## Usage

### With Stakpak CLI

```bash
# Install a pak
stakpak pak install aws-architecture-design

# List installed paks
stakpak pak list

# Use a rulebook
stakpak rulebook use aws-architecture-design
```

### Direct Access

You can also browse and reference the rulebooks directly from this repository. Each rulebook is a standalone Markdown document that can be read and followed independently.

## MCP Integration

Stakpak paks are available via Model Context Protocol (MCP), allowing any compatible AI client to access rulebooks directly.

> **Note:** If you're using [Stakpak Agent](https://stakpak.dev), paks are already built-in and available by default — no additional configuration needed.

**Endpoint:** `https://apiv2.stakpak.dev/v1/paks/mcp`

### Claude

#### Pro, Max, Team & Enterprise (Claude.ai and Claude Desktop)

1. Go to **Settings → Connectors → Add custom connector**
2. Enter:
   - **Integration name:** `Stakpak Paks`
   - **Integration URL:** `https://apiv2.stakpak.dev/v1/paks/mcp`
3. Click **Add**, then **Connect** to authorize

#### Claude Code CLI

```bash
claude mcp add --transport http stakpak-paks https://apiv2.stakpak.dev/v1/paks/mcp
```

### Cursor

1. Press **⌘/Ctrl Shift J**
2. Go to **MCP & Integrations → New MCP server**
3. Add this configuration:

```json
{
  "mcpServers": {
    "stakpak-paks": {
      "url": "https://apiv2.stakpak.dev/v1/paks/mcp"
    }
  }
}
```

### Visual Studio Code

Add to your VS Code `settings.json`:

```json
{
  "mcpServers": {
    "stakpak-paks": {
      "url": "https://apiv2.stakpak.dev/v1/paks/mcp",
      "type": "http"
    }
  }
}
```

Or use the UI:
1. Press **⌘/Ctrl Shift P** → search **MCP: Add Server**
2. Select **HTTP (HTTP or Server-Sent Events)**
3. Enter: `https://apiv2.stakpak.dev/v1/paks/mcp`
4. Name the server **Stakpak Paks**

### Windsurf

1. Press **⌘/Ctrl ,** to open settings
2. Navigate **Cascade → MCP servers → View raw config**
3. Add:

```json
{
  "mcpServers": {
    "stakpak-paks": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://apiv2.stakpak.dev/v1/paks/mcp"]
    }
  }
}
```

### Zed

1. Press **⌘/Ctrl ,** to open settings
2. Add:

```json
{
  "context_servers": {
    "stakpak-paks": {
      "source": "custom",
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://apiv2.stakpak.dev/v1/paks/mcp"]
    }
  }
}
```

### Other MCP Clients

For clients that support remote MCP via stdio:

```json
{
  "stakpak-paks": {
    "command": "npx",
    "args": ["-y", "mcp-remote", "https://apiv2.stakpak.dev/v1/paks/mcp"]
  }
}
```

## Topics Covered

- **Cloud Architecture** — AWS design patterns, cost optimization, SSO authentication
- **Containerization** — Docker best practices, application containerization SOPs
- **Kubernetes** — ConfigHub, configuration management, GitOps alternatives
- **Deployments** — VM deployments, Next.js on AWS, production procedures
- **Operations** — Infrastructure upgrades, Helm chart migrations, FinOps
- **Knowledge Management** — How to write and structure rulebooks

## Contributing

We welcome contributions to improve existing paks or add new ones.

### Creating a New Pak

1. Create a new directory with your pak name (use kebab-case)
2. Add a `pak.toml` with required metadata
3. Create `rulebooks/<pak-name>.md` with your content
4. Submit a pull request

### Guidelines

- Follow the existing pak structure
- Use clear, actionable language in rulebooks
- Include practical examples where applicable
- Tag paks with relevant keywords for discoverability

## License

All paks in this repository are licensed under the [MIT License](LICENSE).

---

<p align="center">
  <strong>Maintained by <a href="https://stakpak.dev">Stakpak</a></strong><br>
  <a href="https://github.com/stakpak/paks">GitHub</a> · <a href="mailto:team@stakpak.dev">Contact</a>
</p>

