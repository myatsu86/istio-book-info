# Book Info

A small microservices demo (the Istio Bookinfo sample). Each app lives in its own folder and builds into its own Docker image. All of them listen on port `9080`.

## How the apps fit together

```
browser -> productpage -> details
                      \-> reviews -> rating   (only if ENABLE_RATINGS=true)
```

`productpage` is the only app a user talks to. It calls the others over HTTP and builds the page.

| App | Language | What it does |
|---|---|---|
| `productpage` | Python (Flask, gunicorn) | Web front page. Calls `details` and `reviews`, and `rating` for the stars display. |
| `details` | Ruby | Returns book details (author, pages, ISBN). |
| `reviews` | Java (Open Liberty) | Returns book reviews. Can call `rating` to add star ratings. |
| `rating` | Node.js | Returns star ratings for a book. |

## Settings

Set these as Docker build args (`--build-arg name=value`) or as environment variables at run time.

| App | Variable | Default | Meaning |
|---|---|---|---|
| all but `productpage` | `SERVICE_VERSION` (build arg `service_version`) | `v1` | Version label of the service |
| `productpage` | `FLOOD_FACTOR` (build arg `flood_factor`) | `0` | Extra load to generate per request |
| `productpage` | `DETAILS_HOSTNAME`, `REVIEWS_HOSTNAME`, `RATINGS_HOSTNAME` | `details`, `reviews`, `ratings` | Where to find the other services |
| `productpage` | `DETAILS_SERVICE_PORT`, `REVIEWS_SERVICE_PORT`, `RATINGS_SERVICE_PORT` | `9080` | Ports of the other services |
| `productpage` | `SERVICES_DOMAIN` | empty | Domain suffix added to the hostnames |
| `details` | `ENABLE_EXTERNAL_BOOK_SERVICE` (build arg `enable_external_book_service`) | `false` | Fetch details from an external book API |
| `reviews` | `ENABLE_RATINGS` (build arg `enable_ratings`) | `false` | Call `rating` and show stars |
| `reviews` | `STAR_COLOR` (build arg `star_color`) | `black` | Color of the stars |

## Dockerfiles

Each app has two:

- `Dockerfile` is the original.
- `Dockerfile.scratch` is the smaller version that CI builds.

What `Dockerfile.scratch` changes:

- `productpage`: multi-stage build. Dependencies install in one stage, unit tests run in another, and the final image holds only the app and its runtime packages. A failing test still fails the build.
- `reviews`: the Liberty feature install, permission fix and cache cleanup happen in one layer, so nothing extra stays in the image.
- `details` and `rating`: no tests, so no test stage.

Build and run one locally:

```sh
docker build -f productpage/Dockerfile.scratch -t productpage-app productpage
docker run -p 9080:9080 productpage-app
```

## CI (GitHub Actions)

Workflows are in `.github/workflows/`:

| File | Purpose |
|---|---|
| `build-app.yml` | Shared steps used by all the others (see below). |
| `ci-details.yml`, `ci-productpage.yml`, `ci-rating.yml`, `ci-reviews.yml` | One pipeline per app. Each runs on push, pull request or manual trigger, only when that app's folder (or its own workflow file) changes. |
| `ci.yml` | Main pipeline. Builds all four apps in parallel. Manual only: run it from the Actions tab. |

Steps in `build-app.yml`:

1. Check out the code.
2. Work out the image tag: the short git commit SHA.
3. Build the image from `<app>/Dockerfile.scratch`.
4. Scan the image with Trivy. For now this only reports CRITICAL/HIGH findings and never fails the build. Set `exit-code` to `'1'` in the workflow to make it block.
5. Log in to Docker Hub and push. Skipped on pull requests, which only build and scan.

Images are named `myatsumon/<app>-app:<short-sha>`, for example `myatsumon/rating-app:a1b2c3d`.

Pushing needs these repository secrets: `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`.
