# Diagrams

Mermaid sources are embedded where they are used, so they render in GitHub/VS Code previews:

| Diagram | Location |
|---|---|
| System context | `../00-system-overview/ARCHITECTURE.md` section 2 |
| Paid checkout sequence | `../00-system-overview/ARCHITECTURE.md` section 4 |
| Door entry sequence | `../00-system-overview/ARCHITECTURE.md` section 4 |
| Browser auth/BFF sequence | `../04-auth-and-rbac/README.md` |
| Contract ownership and parity | `../03-contracts/README.md` |
| Deploy topology | `../11-infrastructure-and-cicd/README.md` |

Standalone `.mmd` exports can be generated with `npx @mermaid-js/mermaid-cli -i <md> -o <svg>` if the partner wants images.
