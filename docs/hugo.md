# Build and host these docs with Chainguard Hugo

[Overview](index.md) · [Tev](tev.md) · [d1](d1.md)

Use `cgr.dev/chainguard/hugo:latest` to build this documentation and `cgr.dev/chainguard/nginx:latest` to serve the resulting HTML. `cgr.dev/hugo` is not the full image reference documented by Chainguard; include the `chainguard` namespace.

The standard Hugo image is non-root and has no shell or package manager. The supplied Dockerfile therefore uses exec-form `RUN`, and the site supplies its own small templates. No external theme, Node.js, Python, Go modules or internet-hosted fonts are required by the site build after the base images are available.

## Included files

| Path | Purpose |
|---|---|
| `docs/*.md` | Shared source for Hugo or another Markdown generator |
| `Dockerfile.hugo` | Hugo build stage and nginx production stage |
| `hugo/hugo.toml` | Site configuration and navigation |
| `hugo/layouts/` | Responsive layout, table of contents and link rendering |
| `mkdocs.yml` | Optional alternative generator; unused by Hugo |

The Dockerfile maps `docs/index.md` to Hugo's `content/_index.md`. A render hook turns relative Markdown links such as `tev.md` into generated page URLs and preserves fragments. Source pages stay usable by other generators. There are no third-party theme assets to fetch.

## Build

Run these commands from the extracted ZIP root, beside `Dockerfile.hugo` and `docs/`. Docker must support multi-stage builds. No GPU is needed.

```bash
docker build --pull \
  --file Dockerfile.hugo \
  --tag tev-d1-docs:local \
  --build-arg SITE_BASE_URL=http://localhost:8080/ \
  .
```

The build runs Hugo directly as the image's non-root user. Source files are copied with matching ownership for consistent read access. Generated output and cache go under `/tmp`, avoiding reliance on the working directory being writable. Only the generated HTML and assets enter the nginx stage.

For an actual hostname, change the build argument to your final HTTPS URL, including the trailing slash:

```bash
docker build --file Dockerfile.hugo \
  --tag tev-d1-docs:production \
  --build-arg SITE_BASE_URL=https://docs.example.com/ \
  .
```

This setup assumes serving at the domain root. A URL path prefix also requires matching proxy routing; test it separately.

## Run

```bash
docker run --detach \
  --name tev-d1-docs \
  --publish 127.0.0.1:8080:8080 \
  --read-only \
  --tmpfs /run:rw,nosuid,nodev \
  --tmpfs /var/lib/nginx/tmp:rw,nosuid,nodev \
  --cap-drop ALL \
  --security-opt no-new-privileges:true \
  tev-d1-docs:local
```

Open `http://localhost:8080/`. Docker publishes only on host loopback. For remote access, use a reverse proxy or an SSH tunnel. On k3s, use port 8080 behind a Service/Ingress and writable temporary volumes for `/run` and `/var/lib/nginx/tmp`. The nginx runtime serves static files; it does not start vLLM or load a model.

```bash
curl --fail http://127.0.0.1:8080/
curl --fail http://127.0.0.1:8080/tev/
curl --fail http://127.0.0.1:8080/d1/
curl --fail http://127.0.0.1:8080/hugo/
docker logs --tail 50 tev-d1-docs
```

## Update pages

Edit Markdown in `docs/`, rebuild, then recreate the documentation container with the new image. Existing page changes require no preprocessing. For a new page, add a corresponding `COPY` instruction to `Dockerfile.hugo` and a menu entry to `hugo/hugo.toml`. The build fails on unresolved local `.md` links.

## Pin images and prepare for offline use

The public starter images use moving `latest` tags. After validating a build, record the actual repository digests:

```bash
docker pull cgr.dev/chainguard/hugo:latest
docker pull cgr.dev/chainguard/nginx:latest
docker image inspect cgr.dev/chainguard/hugo:latest --format '{{json .RepoDigests}}'
docker image inspect cgr.dev/chainguard/nginx:latest --format '{{json .RepoDigests}}'
```

Pass the returned full `name@sha256:...` references as `HUGO_IMAGE` and `NGINX_IMAGE` build arguments. Do not assume a numbered public tag is available: Chainguard lists version-tag access separately from free starter tags.

For a disconnected host, building the final documentation image on the internet side and transporting it is simplest:

```bash
docker save --output tev-d1-docs.tar tev-d1-docs:production
# Transfer the tar using your approved process, then on the destination:
docker load --input tev-d1-docs.tar
```

Run the imported image using the same container settings. This imports the built site and runtime; it does not require Hugo on the destination.

## Troubleshooting

| Problem | Check |
|---|---|
| `/bin/sh` missing | Use the supplied exec-form RUN; standard distroless images have no shell |
| Permission denied during Hugo generation | Preserve COPY ownership and the /tmp destination/cache flags; build-stage UID is 65532 |
| Registry access denied | Use the documented namespace; check your registry access for the requested tag |
| Missing page or `.md` link | Add the page COPY entry and retain the render hook |
| nginx read-only filesystem error | Retain both writable tmpfs mounts |
| New content not visible | Rebuild and recreate the container; Markdown is compiled into the image |

## Validation and sources

Container execution requires Docker, which is not available in the authoring workspace. See `hugo/validation.txt` for the exact local build checks completed; the container commands have not been claimed as runtime-tested.

- [Chainguard Hugo image](https://images.chainguard.dev/directory/image/hugo/overview)
- [Hugo image specifications](https://images.chainguard.dev/directory/image/hugo/specifications)
- [Chainguard nginx image and read-only setup](https://images.chainguard.dev/directory/image/nginx/overview)
- [Hugo template types](https://gohugo.io/templates/types/)
- [Hugo link render hooks](https://gohugo.io/render-hooks/links/)

Checked 26 October 2026.
