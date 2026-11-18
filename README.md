# workflows

Shared deploy plumbing for my apps: **GitHub builds the image, Coolify runs it.**

| Piece | What it is |
| --- | --- |
| `.github/workflows/build-deploy.yml` | Reusable workflow: build → push to GHCR (`:latest` + `:<sha>`) → trigger Coolify |
| `deploy/` | Composite action that only triggers Coolify — for repos with their own custom build |
| `templates/deploy.yml` | The ~10-line file each app repo gets |
| `bin/new-app` | CLI that creates the Coolify resource and wires the repo |

Nothing here is secret. Each app repo holds:

- `COOLIFY_TOKEN` — **secret**, the same value in every repo (set by `new-app`, rotated by `new-app sync-token`)
- `COOLIFY_WEBHOOK` — **variable**, that repo's deploy URL (useless without the token)

## One-time setup

```sh
ln -s ~/source/personal_stuv/workflows/bin/new-app ~/.local/bin/new-app
new-app setup          # stores Coolify URL + API token in the macOS keychain
```

Coolify must be able to pull from GHCR. Packages from public repos can be made
public; for private ones, log the server in once:
`docker login ghcr.io -u <user>` with a PAT that has `read:packages`.

## New app

```sh
cd ~/source/personal_stuv/my-app
new-app create Otagera/my-app --env .env.production --port 3000 --domain https://my-app.example.com
git add .github/workflows/deploy.yml && git commit -m "Deploy via Coolify" && git push
```

That creates a Coolify project + app pointing at `ghcr.io/otagera/my-app:latest`,
loads the env vars, sets the secret + variable on the repo, and writes the
workflow. The push builds the image and deploys it.

Compose stack instead of a single image: `--compose docker-compose.yml`.
Coolify's magic variables (`SERVICE_PASSWORD_DB`, `SERVICE_FQDN_APP`, …) work
there, so generated passwords don't need to be in the `.env` at all.

## Existing app

```sh
new-app link Otagera/comics <coolify-resource-uuid>     # sets secret + variable
new-app env  <coolify-resource-uuid> .env.production    # reload env vars
```

Then either replace its `deploy.yml` with `templates/deploy.yml`, or keep the
custom build and replace just the curl step:

```yaml
      - uses: Otagera/workflows/deploy@main
        with:
          webhook: ${{ vars.COOLIFY_WEBHOOK }}
          token: ${{ secrets.COOLIFY_TOKEN }}
```

## Reusable workflow inputs

| Input | Default | |
| --- | --- | --- |
| `image` | repo name | GHCR image name |
| `context` / `file` / `target` | `.` / `Dockerfile` / – | Docker build |
| `build-args` | – | Non-secret `KEY=VALUE` lines |
| `platforms` | `linux/amd64` | |
| `deploy` | `true` | `false` when one repo builds several images |
| `webhook` | – | pass `${{ vars.COOLIFY_WEBHOOK }}` |

Secrets: `COOLIFY_TOKEN`, and optional `BUILD_ARGS` (secret build args, one
`KEY=VALUE` per line). `secrets: inherit` passes both.

Several images, one deploy:

```yaml
jobs:
  api:
    uses: Otagera/workflows/.github/workflows/build-deploy.yml@main
    with: { image: lumina-api, target: api, deploy: false }
    secrets: inherit
  worker:
    uses: Otagera/workflows/.github/workflows/build-deploy.yml@main
    with: { image: lumina-worker, target: worker, deploy: false }
    secrets: inherit
  deploy:
    needs: [api, worker]
    runs-on: ubuntu-latest
    steps:
      - uses: Otagera/workflows/deploy@main
        with:
          webhook: ${{ vars.COOLIFY_WEBHOOK }}
          token: ${{ secrets.COOLIFY_TOKEN }}
```

## Rollback

Every build also pushes `:<sha>`. Point the Coolify resource's image tag at a
previous sha and redeploy.
