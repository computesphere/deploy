# computesphere/deploy

Deploy an app to [ComputeSphere](https://computesphere.com) from a
`computesphere.yaml` manifest (or a prebuilt image) and **wait for it to reach
Running** — the exit code reflects whether the deploy actually became healthy,
so a red job means a bad deploy.

It installs the `csph` CLI (via [`computesphere/setup-csph`](https://github.com/computesphere/setup-csph)),
runs `csph deploy`, and surfaces the deployment id, URL, and status as action
outputs.

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

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `token` | **yes** | | ComputeSphere API token. A project-scoped `csph_` token is recommended. Masked and exported as `COMPUTESPHERE_API_TOKEN` by the install step — never interpolated into a shell command. |
| `version` | no | `latest` | csph version to install. Pin a concrete version (e.g. `0.12.3`) for reproducible pipelines. |
| `file` | no | `computesphere.yaml` | Manifest to deploy. Ignored when `image` is set. |
| `image` | no | | A prebuilt image to deploy as a one-service web app instead of a manifest. Prefer an immutable digest. |
| `name` | no | | Service name when deploying with `image` (defaults to a name derived from the image). |
| `port` | no | `8080` | Container port for the `image` service. |
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

## Versioning & pinning

Released with a moving major tag. Use `@v1` for automatic patch/minor updates:

```yaml
- uses: computesphere/deploy@v1
```

Or pin to a full-length commit SHA for a hardened supply chain:

```yaml
- uses: computesphere/deploy@<40-char-sha>  # v1.0.0
```

## Security

- The `token` is passed to `setup-csph`, which masks it (`::add-mask::`) and
  exports it as an environment variable. It is **never** interpolated into a
  `run:` command.
- All user inputs are mapped into step `env:` and referenced as quoted `"$VARS"`
  — never spliced into the command string — so a value can't inject shell.
- Deploy by immutable **digest** and enable `provenance`/`sbom` on your build so
  what you deploy is provably what you built.

## Related

- [`computesphere/setup-csph`](https://github.com/computesphere/setup-csph) — the
  install primitive this action builds on. Use it directly for a plain
  `run: csph …` step.
