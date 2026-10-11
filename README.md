# Book Info

A small microservices demo (the Istio Bookinfo sample). Each app lives in its own folder and builds into its own Docker image.

| App | Language | What it does |
|---|---|---|
| `productpage` | Python | Web front page, calls details and reviews |
| `details` | Ruby | Book details |
| `reviews` | Java (Open Liberty) | Book reviews, can call ratings |
| `rating` | Node.js | Star ratings |

## Dockerfiles

- `Dockerfile` is the original.
- `Dockerfile.scratch` is the size-optimized version that CI uses. For example, productpage runs its tests in a separate build stage so test files stay out of the final image.

Build one locally:

```sh
docker build -f productpage/Dockerfile.scratch -t productpage-app productpage
```

## CI (GitHub Actions)

Workflows are in `.github/workflows/`:

- `build-app.yml`: shared steps. It builds the image, scans it with Trivy (report only, never fails the build), then pushes to Docker Hub. Pull requests only build and scan; they don't push.
- `ci-<app>.yml`: one pipeline per app. It runs when that app's folder changes.
- `ci.yml`: main pipeline that builds all apps. Run it manually from the Actions tab.

Images are tagged with the short git commit SHA, for example `myatsumon/rating-app:a1b2c3d`.

Pushing needs the repository secrets `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`.
