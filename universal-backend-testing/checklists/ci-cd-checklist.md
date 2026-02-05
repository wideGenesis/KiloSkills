# CI/CD Integration Checklist

## Workflow Configuration

- [ ] Workflow triggers on push and PR
- [ ] Python version matrix configured
- [ ] Dependency caching enabled
- [ ] Secrets properly secured

## Test Stages

### Stage 1: Fast Feedback (PR Gate)

- [ ] Lint and type checks pass
- [ ] Unit tests run (`-m "not slow"`)
- [ ] Coverage ≥ 80%
- [ ] Duration: p50 ≤ 10 min, p95 ≤ 20 min

### Stage 2: Integration (Pre-Merge)

- [ ] Integration tests pass
- [ ] Database migration tests pass
- [ ] Contract tests pass
- [ ] Security scan clean

### Stage 3: Full Suite (Nightly/Release)

- [ ] All tests including slow
- [ ] E2E tests pass
- [ ] Performance regression checked
- [ ] Coverage report generated

## Quality Gates

### Merge Gate
- [ ] All unit tests pass
- [ ] Coverage ≥ 80%
- [ ] No new flaky tests
- [ ] Static analysis clean

### Deploy Gate
- [ ] Integration tests pass
- [ ] E2E smoke tests pass
- [ ] Contract tests pass
- [ ] Security scan clean

## Coverage Reporting

- [ ] Coverage uploaded to external service (Codecov/Code Climate)
- [ ] PR coverage diff commented
- [ ] Coverage badges in README
- [ ] Coverage history tracked

## Flaky Test Management

- [ ] Flake rate tracked weekly
- [ ] SLO: ≤ 1% weekly flake rate
- [ ] Quarantine process documented
- [ ] Flaky tests have owners and tickets

## Artifacts & Debugging

- [ ] Test reports saved as artifacts
- [ ] Coverage reports archived
- [ ] Logs collected on failure
- [ ] Screenshots saved for E2E failures

## Example Workflow

```yaml
name: Tests

on: [push, pull_request]

jobs:
  # Stage 1: Fast Feedback
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -e ".[dev]"
      
      - name: Lint
        run: ruff check src tests
      
      - name: Type check
        run: mypy src
      
      - name: Run unit tests
        run: |
          pytest tests/unit -m "not slow" \
            --cov=myapp \
            --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  # Stage 2: Integration
  integration-tests:
    needs: unit-tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
    steps:
      - uses: actions/checkout@v4
      
      - name: Run integration tests
        run: pytest tests/integration
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost/test

  # Stage 3: E2E (on main only)
  e2e-tests:
    needs: integration-tests
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      
      - name: Run E2E tests
        run: pytest tests/e2e
```

## Metrics to Track

| Metric | Target | Alert If |
|--------|--------|----------|
| Build Success Rate | ≥ 99% | < 95% |
| PR Gate Duration | ≤ 10 min | > 20 min |
| Flake Rate | ≤ 1% | > 2% |
| Coverage | ≥ 80% | < 75% |

## Quick Check

- [ ] All PRs run tests automatically
- [ ] Failed builds block merge
- [ ] Coverage reports accessible
- [ ] Failed test logs easy to find
