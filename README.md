# Centralized reusable GitHub Actions workflows

Reusable CI/CD workflows for RideTrip-tour microservices. Place a small calling workflow in each service under `.github/workflows/`; implementation lives here.

## Available workflows

| Workflow | Purpose |
| --- | --- |
| `python-ruff.yml` | Ruff lint and format checks |
| `python-mypy.yml` | Mypy type checking |
| `python-test.yml` | Pytest + coverage and XML artifact |
| `docker-build-push.yml` | Docker Hub build/push; exports `image-ref` |
| `swarm-alembic-check.yml` | Runs `alembic check` against the current database before migrations |\n| `swarm-migrate.yml` | Docker Swarm replicated migration job |
| `swarm-deploy.yml` | Swarm service update |
| `swarm-healthcheck.yml` | HTTP health probe inside overlay network |
| `telegram-notify.yml` | Telegram success/failure notification |

## Example: auth-service / develop

Store the following in `RideTrip-tour/auth-service/.github/workflows/ci-dev.yml` (this repository only provides the example; it does not change auth-service). For initial integration testing, use the branch ref `@feat/swarm-deployment-reusable-workflows`; **replace it with a release tag or immutable commit SHA after merging and releasing the templates**.

```yaml
name: CI Dev

on:
  push:
    branches: [develop]
  pull_request:
    branches: [develop]

permissions:
  contents: read

jobs:
  ruff:
    uses: RideTrip-tour/ci-templates/.github/workflows/python-ruff.yml@feat/swarm-deployment-reusable-workflows

  tests:
    uses: RideTrip-tour/ci-templates/.github/workflows/python-test.yml@feat/swarm-deployment-reusable-workflows
    with:
      coverage-source: app
      coverage-min: 90

  # Enable if mypy is configured and installed in requirements.test.txt:
  # mypy:
  #   uses: RideTrip-tour/ci-templates/.github/workflows/python-mypy.yml@feat/swarm-deployment-reusable-workflows
  #   with:
  #     mypy-target: app

  build:
    needs: [ruff, tests]
    if: github.event_name == 'push'
    uses: RideTrip-tour/ci-templates/.github/workflows/docker-build-push.yml@feat/swarm-deployment-reusable-workflows
    with:
      namespace: ${{ vars.DOCKERHUB_NAMESPACE || 'ride2trip' }}
      service-name: ${{ vars.SERVICE_NAME || 'auth-service' }}
      tag-prefix: dev
    secrets: inherit

  migrate:
    needs: build
    uses: RideTrip-tour/ci-templates/.github/workflows/swarm-migrate.yml@feat/swarm-deployment-reusable-workflows
    with:
      service-name: ${{ vars.SERVICE_NAME || 'auth-service' }}
      image-ref: ${{ needs.build.outputs.image-ref }}
      data-network: ${{ vars.DATA_NETWORK || 'data-network' }}
      db-secret-prefix: DB_AUTH_SERVICE
      db-env-prefix: DB_AUTH_SERVICE
      environment: DEV
    secrets: inherit

  deploy:
    needs: [build, migrate]
    uses: RideTrip-tour/ci-templates/.github/workflows/swarm-deploy.yml@feat/swarm-deployment-reusable-workflows
    with:
      image-ref: ${{ needs.build.outputs.image-ref }}
      swarm-service: ${{ vars.AUTH_SWARM_SERVICE || 'apps_auth-service' }}
      environment: DEV
    secrets: inherit

  healthcheck:
    needs: deploy
    uses: RideTrip-tour/ci-templates/.github/workflows/swarm-healthcheck.yml@feat/swarm-deployment-reusable-workflows
    with:
      service-name: ${{ vars.SERVICE_NAME || 'auth-service' }}
      app-name: ${{ vars.APP_NAME || 'auth' }}
      data-network: ${{ vars.DATA_NETWORK || 'data-network' }}
      environment: DEV
    secrets: inherit

  notify:
    needs: [ruff, tests, build, alembic_check, migrate, deploy, healthcheck]
    if: ${{ always() && (github.event_name == 'push' || contains(needs.*.result, 'failure')) }}
    uses: RideTrip-tour/ci-templates/.github/workflows/telegram-notify.yml@feat/swarm-deployment-reusable-workflows
    with:
      service-name: ${{ vars.SERVICE_NAME || 'auth-service' }}
      environment: DEV
      swarm-service: ${{ vars.AUTH_SWARM_SERVICE || 'apps_auth-service' }}
      results-json: ${{ toJSON(needs) }}
      image-ref: ${{ needs.build.outputs.image-ref }}
    secrets: inherit
```

**Required secrets** (repository/org secrets or explicitly passed to called workflows):
- `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`
- `SWARM_SSH_PRIVATE_KEY`, `SWARM_HOST`, `SWARM_USER`
- `SWARM_KNOWN_HOSTS`: trusted, verified SSH host key line(s) for the Swarm manager. Do **not** blindly trust an unverified output of `ssh-keyscan`.
- `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`; optional `TELEGRAM_DEPLOY_THREAD_ID`.

**Environment**: Swarm jobs select GitHub environment `DEV`. Configure any environment-scoped secrets/protection rules there. `secrets: inherit` passes available caller secrets to the reusable workflow; environment secrets need to be available to the called job's selected environment.

**Swarm prerequisites**: manager reachable over SSH, Docker CLI available remotely, overlay network (default `data-network`), service (default `apps_auth-service`), Docker registry access, replicated-job mode support, and existing Docker Swarm DB secrets `DB_AUTH_SERVICE_{HOST,PORT,NAME,USER,PASS}`. The Alembic check runs `alembic check` directly in the service image with the DB secrets mounted. Set `db-secret-prefix` to the prefix used in Swarm secret names and `db-env-prefix` to the environment-variable prefix expected by that service; these can differ between microservices. The migration container runs its migration command through its entrypoint when `APP_MODE=migrate` is set. The probe checks `http://<service-name>:8000/<app-name>/health` from the overlay network.

**Notes**:
- Test and Ruff run in parallel; Docker build runs only after both pass and only on push, not PR. The Alembic check must pass before migrations; migrations must pass before deployment.
- `docker-build-push.yml` publishes both `dev-<sha7>` and `dev`; deployment uses the SHA-specific tag returned as a workflow output. For stronger immutability consider deploying by digest.
- The success notification is sent after checks, build, migration, rollout and healthcheck. Failures in stages are reported based on `needs`; any skipped stages are not labelled as failed.
- The original auth-service workflow is **not changed** by this PR. The example above must be applied in auth-service separately after reviewing it.
- For production, replace mutable workflow refs and, preferably, pin third-party actions by full commit SHA. Consider serialization using `concurrency` for deployments to a shared Swarm service.
