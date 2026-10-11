# Centralized reusable GitHub Actions workflows

Python quality checks for microservices in the RideTrip-tour organization.

Reusable workflows are located under `.github/workflows/` and are invoked from the consuming repository's `.github/workflows/ci.yml`. The consuming repository is checked out automatically.

## Example

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  ruff:
    uses: RideTrip-tour/ci-templates/.github/workflows/python-ruff.yml@main

  mypy:
    uses: RideTrip-tour/ci-templates/.github/workflows/python-mypy.yml@main
    with:
      mypy-target: app

  tests:
    uses: RideTrip-tour/ci-templates/.github/workflows/python-test.yml@main
    with:
      coverage-source: app
      coverage-min: 90
```

**For production use, replace `@main` with an immutable release tag or commit SHA** after merging and releasing these templates.

All workflows default to Python 3.12, pip 25.0.1 and `requirements.test.txt`. The dependencies file must install the applicable tools (`ruff`, `mypy`, `pytest`, `pytest-cov`) as well as project requirements. Jobs run independently in parallel unless the caller sets `needs`.

The test workflow runs pytest and coverage together; it uploads `coverage.xml` as an artifact for 14 days when available. Its minimum coverage threshold is configurable with `coverage-min`. Other settings can be overridden via the workflow's inputs.
