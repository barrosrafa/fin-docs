# Antigravity Vault Agent Rules — fin-docs

You are operating inside the **fin-docs** Obsidian Vault, the central documentation hub for the Fin platform ecosystem.

Adhere to the following conventions when generating or editing documentation:

1. **Wikilinks & Cross-References**: Prefer Obsidian wikilinks `[[Note Name]]` or `[[Note Name|Custom Text]]` for internal documentation linking.
2. **Architecture Modules**:
   - `[[fin-api/README|fin-api]]`: FastAPI, Python 3.12+, Uvicorn, ARQ Workers.
   - `[[fin-db/README|fin-db]]`: PostgreSQL 18 (Neon Serverless), TimescaleDB, pgvector, 12 Schemas, 303 Tables.
   - `[[fin-infra/README|fin-infra]]`: Traefik, Caddy, Cloudflare Tunnel, Anti-bot proxy.
   - `[[fin-ingestion/README|fin-ingestion]]`: Scrapers, dbt, asynchronous ingestion workers.
   - `[[fin-web/README|fin-web]]`: Next.js 15, React 19, Tailwind CSS, shadcn/ui.
3. **Diagrams & Visuals**: Use Mermaid diagrams (`flowchart TD`, `sequenceDiagram`, `erDiagram`) for architectural, data flow, and workflow specifications.
4. **Metadata & Frontmatter**: Preserve YAML frontmatter at the top of markdown notes (`title`, `tags`, `updated_at`, `status`).
5. **Safety**: Do not overwrite or delete existing documentation notes unless explicitly requested. Always make targeted, clean additions.
