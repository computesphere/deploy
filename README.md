# computesphere/deploy

Deploy an app to [ComputeSphere](https://computesphere.com) from a
`computesphere.yaml` manifest (or a prebuilt image) and **wait for it to reach
Running** — the exit code reflects whether the deploy actually became healthy,
so a red job means a bad deploy.

It installs the `csph` CLI (via [`computesphere/setup-csph`](https://github.com/computesphere/setup-csph)),
runs `csph deploy`, and surfaces the deployment id, URL, and status as action
outputs.

> **You don't add a `setup-csph` step yourself** — this action installs and
> authenticates `csph` for you. Control the CLI version with the `version` input.
> If you'd rather run `csph` directly (e.g. several `csph` commands in one job),
> use [`setup-csph`](https://github.com/computesphere/setup-csph) once and call
> `run: csph deploy …` — see [Do I need setup-csph?](#do-i-need-setup-csph) below.

```yaml
- uses: computesphere/deploy@v1
  with:
    token: ${{ secrets.COMPUTESPHERE_API_TOKEN }}
    file: computesphere.yaml
    environment: ${{ vars.COMPUTESPHERE_ENV_ID }}
```

## Quick starts

### Deploy a manifest

The manifest describes your project, environments, and services. Apply is
additive and idempotent — re-running never deletes a resource.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: computesphere/deploy@v1
        with:
          token: ${{ secrets.COMPUTESPHERE_API_TOKEN }}
          file: computesphere.yaml
          environment: ${{ vars.COMPUTESPHERE_ENV_ID }}
```

### Build, then promote the image by digest

Deploy-by-**digest** means the artifact you tested is provably the one you ship.

```yaml
jobs:
  ship:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - id: build
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          provenance: true
          sbom: true
      - id: deploy
        uses: computesphere/deploy@v1
        with:
          token: ${{ secrets.COMPUTESPHERE_API_TOKEN }}
          image: ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
          name: api
          project: ${{ vars.COMPUTESPHERE_PROJECT_ID }}
          environment: ${{ vars.COMPUTESPHERE_ENV_ID }}
      - run: echo "Deployed ${{ steps.deploy.outputs.service-url }}"
```

### Deploy a private-registry image

For an image that requires authentication, pass registry credentials. Store the
password/token as a repository or environment **secret** — the action masks it
before use.

```yaml
- uses: computesphere/deploy@v1
  with:
    token: ${{ secrets.COMPUTESPHERE_API_TOKEN }}
    image: ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
    name: api
    registry-username: ${{ github.actor }}
    registry-password: ${{ secrets.GHCR_PULL_TOKEN }}
    image-provider: ghcr
    environment: ${{ vars.COMPUTESPHERE_ENV_ID }}
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `token` | **yes** | | ComputeSphere API token. A project-scoped `csph_` token is recommended. Masked and exported as `COMPUTESPHERE_API_TOKEN` by the install step — never interpolated into a shell command. |
| `version` | no | `latest` | csph version to install. Pin a concrete version (e.g. `0.12.3`) for reproducible pipelines. |
| `file` | no | `computesphere.yaml` | Manifest to deploy. Ignored when `image` is set. |
| `image` | no | | A prebuilt image to deploy as a one-service web app instead of a manifest. Prefer an immutable digest. |
| `name` | no | | Service name when deploying with `image` (defaults to a name derived from the image). |
| `port` | no | `8080` | Container port for the `image` service. |
| `registry-username` | no | | Username for the private registry hosting `image`. Only applies with `image`. |
| `registry-password` | no | | Password or access token for the private registry. Only applies with `image`. **Secret** — pass a repository/environment secret; the action masks it. |
| `registry-url` | no | | Registry server URL (e.g. `ghcr.io`). Only applies with `image`; defaults to the registry in the image reference. |
| `image-provider` | no | | Registry provider hint (e.g. `ghcr`, `dockerhub`, `acr`). Only applies with `image`. |
| `project` | no | | Target project ID. Falls back to the token's default. |
| `environment` | no | | Target environment ID. |
| `atomic` | no | `false` | Roll every resource back if any fails (all-or-nothing). |
| `wait-timeout` | no | `5m` | How long to wait for Running, as a Go duration (`5m`, `90s`). |
| `no-wait` | no | `false` | Return as soon as resources are accepted. **Not recommended in CI** — the job can't then reflect deploy health. |
| `working-directory` | no | `.` | Directory to run the deploy in (where the manifest lives). |

## Outputs

| Output | Description |
|--------|-------------|
| `deployment-id` | Deployment ID of the single deployed/updated service (empty for a multi-service manifest). |
| `service-url` | Public URL of the deployed service. |
| `status` | Apply outcome — `created`, `updated`, or `unchanged`. |
| `result` | The full `csph deploy --output json` result document. |

```yaml
- id: deploy
  uses: computesphere/deploy@v1
  with:
    token: ${{ secrets.COMPUTESPHERE_API_TOKEN }}
    environment: ${{ vars.COMPUTESPHERE_ENV_ID }}
- name: Comment the preview URL
  run: gh pr comment "$PR" --body "Deployed → ${{ steps.deploy.outputs.service-url }}"
```

## Waiting and exit codes

By default the action waits for the deployment to reach **Running** and exits
non-zero if it doesn't within `wait-timeout` — a zero-downtime redeploy is
followed to its new version's outcome, so a broken image fails the job instead
of silently reporting success. Pass `no-wait: true` to opt out (the deploy still
starts server-side; the job just won't reflect its health).

## Do I need setup-csph?

**No — not with this action.** `computesphere/deploy` installs and authenticates
`csph` internally (it runs `computesphere/setup-csph` for you), so a single step
is all you need:

```yaml
- uses: computesphere/deploy@v1
  with:
    token: ${{ secrets.COMPUTESPHERE_API_TOKEN }}
    version: 0.12.3        # optional — pin the csph CLI version
    environment: ${{ vars.COMPUTESPHERE_ENV_ID }}
```

Reach for [`setup-csph`](https://github.com/computesphere/setup-csph) directly
only when you want to drive `csph` yourself — for example running several `csph`
commands in one job, or scripting a flow this action doesn't cover:

```yaml
- uses: computesphere/setup-csph@v1
  with:
    token: ${{ secrets.COMPUTESPHERE_API_TOKEN }}
- run: csph deploy --file computesphere.yaml --environment "$ENV_ID" --output json
- run: csph deployment logs "$DEPLOYMENT_ID" --kind runtime --tail 50
```

The two are consistent: `computesphere/deploy` is exactly `setup-csph` + a
hardened `csph deploy` call with parsed outputs.

## Versioning & pinning

Released with a moving major tag. Use `@v1` for automatic patch/minor updates:

```yaml
- uses: computesphere/deploy@v1
```

Or pin to a full-length commit SHA for a hardened supply chain:

```yaml
- uses: computesphere/deploy@<40-char-sha>  # v1.0.0
```

Pinning `deploy@<sha>` freezes the whole toolchain: this action pins its own
`setup-csph` dependency to a SHA (not a moving tag), so a pinned deploy is
genuinely self-contained and reproducible — no transitive drift. Dependency
bumps land via a review-required PR (never blind auto-merge).

## Security

- The `token` is passed to `setup-csph`, which masks it (`::add-mask::`) and
  exports it as an environment variable. It is **never** interpolated into a
  `run:` command.
- `registry-password` is masked (`::add-mask::`) by the deploy step before any
  command is echoed or run. Still, always source it from a repository or
  environment secret — never a workflow literal.
- All user inputs are mapped into step `env:` and referenced as quoted `"$VARS"`
  — never spliced into the command string — so a value can't inject shell.
- Deploy by immutable **digest** and enable `provenance`/`sbom` on your build so
  what you deploy is provably what you built.

## Related

- [`computesphere/setup-csph`](https://github.com/computesphere/setup-csph) — the
  install primitive this action builds on. Use it directly for a plain
  `run: csph …` step.
