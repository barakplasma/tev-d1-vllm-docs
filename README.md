# Tev and d1 deployment documentation

Copy `docs/` into your static documentation site. Start at `docs/index.md`. An optional MkDocs navigation file is included. No GPU resources are created by these files.

For Chainguard Hugo, follow `docs/hugo.md`. Build with `docker build -f Dockerfile.hugo -t tev-d1-docs:local .`.
