# AgentX-Docs

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![MkDocs](https://img.shields.io/badge/docs-MkDocs-blue)](https://www.mkdocs.org/)
[![Website](https://img.shields.io/website?url=https%3A%2F%2Fdocs.agentx.io)](https://docs.agentx.io)

> Official documentation, whitepaper, and API reference for the AgentX Protocol — the open framework for autonomous AI agents on Solana.

---

## Documentation

Live docs: **[docs.agentx.io](https://docs.agentx.io)**

| Resource | Description |
|----------|-------------|
| [Whitepaper](docs/whitepaper.md) | Technical architecture, tokenomics, and roadmap |
| [API Reference](docs/api_reference.md) | AgentX-Core Python SDK reference |
| [Architecture Diagram](diagrams/architecture.drawio) | System architecture (open with draw.io) |

---

## Related Repositories

| Repo | Description |
|------|-------------|
| [AgentX-Core](https://github.com/agentx-protocol/AgentX-Core) | Python SDK — build agents |
| [AgentX-Solana-Deploy](https://github.com/agentx-protocol/AgentX-Solana-Deploy) | Rust/Anchor on-chain program |
| [AgentX-Examples](https://github.com/agentx-protocol/AgentX-Examples) | Example agents (trading, social, research) |

---

## Build Docs Locally

```bash
git clone https://github.com/agentx-protocol/AgentX-Docs.git
cd AgentX-Docs
pip install mkdocs mkdocs-material
mkdocs serve
# Open http://localhost:8000
```

---

## Contributing

Found an error? Want to improve the docs?

1. Open an issue describing what's wrong/missing
2. Fork and create a branch: `git checkout -b docs/fix-api-reference`
3. Edit the relevant Markdown file in `docs/`
4. Submit a PR — docs PRs are merged quickly!

---

## License

MIT — see [LICENSE](LICENSE).
