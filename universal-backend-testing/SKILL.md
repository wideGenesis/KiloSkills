---
name: universal-backend-testing
description: Comprehensive backend testing framework combining TDD, pytest patterns, test pyramid strategy, CI/CD integration, and workflow testing. Covers unit, integration, E2E testing with quality gates and observability.
---

# Universal Backend Testing

Complete testing framework for Python backend applications combining best practices from TDD, pytest, QA strategy, and workflow testing.

## When to Use This Skill

- Setting up testing infrastructure for Python backend projects
- Implementing test-driven development (TDD)
- Designing test strategy following the test pyramid
- Writing tests at any level: unit, integration, E2E
- Setting up CI/CD quality gates
- Testing Temporal workflows and activities
- Managing test coverage and flaky tests

---

## Table of Contents

1. [Overview & Philosophy](#1-overview--philosophy)
2. [Test Pyramid & Strategy](#2-test-pyramid--strategy)
3. [Unit Testing](#3-unit-testing)
4. [Integration Testing](#4-integration-testing)
5. [Advanced Patterns](#5-advanced-patterns)
6. [E2E & Workflow Testing](#6-e2e--workflow-testing)
7. [Coverage & Quality Gates](#7-coverage--quality-gates)
8. [CI/CD Integration](#8-cicd-integration)
9. [Test Organization & Maintenance](#9-test-organization--maintenance)
10. [Troubleshooting & Best Practices](#10-troubleshooting--best-practices)

---

## 1. Overview & Philosophy

### Core Principles

| Principle | Description |
|-----------|-------------|
| **TDD First** | Write tests before code: RED → GREEN → REFACTOR |
| **Test Behavior** | Test what code does, not how it does it |
| **Fast Feedback** | Unit tests run in milliseconds, PR gate ≤10 min |
| **Isolation** | Tests must be independent with no shared state |
| **Determinism** | Same input → same output, every time |

### TDD Cycle

```
┌─────────┐    ┌──────────┐    ┌───────────┐
│  RED    │───→│  GREEN   │───→│ REFACTOR  │
│         │    │          │    │           │
│ Write   │    │ Minimal  │    │ Improve   │
│ failing │    │ code to  │    │ design    │
│ test    │    │ pass     │    │ keep green│
└─────────┘    └──────────┘    └─────┬─────┘
      ↑───────────────────────────────┘
```

```python
# Step 1: RED - Write failing test
def test_calculate_discount():
    assert calculate_discount(100, 10) == 90  # Fails: function doesn't exist

# Step 2: GREEN - Minimal implementation
def calculate_discount(price: float, percent: float) -> float:
    return price * (1 - percent / 100)

# Step 3: REFACTOR - Improve while keeping tests green
def calculate_discount(price: float, percent: float) -> float:
    """Calculate discounted price."""
    if percent < 0 or percent > 100:
        raise ValueError("Discount must be 0-100")
    return price * (1 - percent / 100)
```

---

## 2. Test Pyramid & Strategy

### Test Pyramid Structure

```
            /\
           /E2E \          5-10% - Critical user journeys
          /------\
         /Integr. \       15-25% - API, DB, queues, external services
        /----------\
       / Component  \      20-30% - Service boundaries
      /--------------\
     /     Unit       \    40-60% - Business logic, pure functions
    /------------------\
```

### Test Type Decision Tree

```
Need to test: [Feature Type]
    │
    ├─ Pure business logic/invariants? → Unit tests (mock boundaries)
    │
    ├─ API Endpoint?
    │   ├─ Single service boundary? → Integration tests (real DB/deps)
    │   └─ Cross-service compatibility? → Contract tests (OpenAPI/AsyncAPI)
    │
    ├─ Database operations? → Integration tests with in-memory/test DB
    │
    ├─ Temporal Workflow? → Workflow tests with time-skipping
    │
    ├─ Event-driven/API schema evolution? → Contract + backward-compat tests
    │
    └─ Critical user journey? → E2E tests (1-2 per product area)
```

### Shift-Left vs Shift-Right

| Phase | Activities | Goal |
|-------|------------|------|
| **Shift-Left** (Pre-Merge) | Unit tests, contract validation, static analysis, lint | Catch defects early |
| **Shift-Right** (Post-Deploy) | Synthetic checks, canary analysis, feature flags | Validate production behavior |

---

## 3. Unit Testing

### AAA Pattern

Every unit test follows **Arrange → Act → Assert**:

```python
def test_user_creation():
    # Arrange: Set up test data and preconditions
    user_data = {"name": "Alice", "email": "alice@example.com"}
    service = UserService()
    
    # Act: Execute the code under test
    user = service.create_user(user_data)
    
    # Assert: Verify the results
    assert user.name == "Alice"
    assert user.email == "alice@example.com"
    assert user.id is not None
```

### Basic pytest Structure

```python
import pytest
from unittest.mock import Mock, patch

# Simple test function
def test_addition():
    """Test basic addition."""
    assert 2 + 2 == 4

# Test with exception
def test_divide_by_zero():
    """Test division by zero raises error."""
    with pytest.raises(ZeroDivisionError):
        1 / 0

# Test exception message
def test_invalid_email_raises():
    """Test invalid email raises ValueError with message."""
    with pytest.raises(ValueError, match="Invalid email format"):
        validate_email("not-an-email")

# Test exception attributes
def test_custom_exception():
    """Test exception with custom attributes."""
    with pytest.raises(CustomError) as exc_info:
        raise CustomError("error", code=400)
    assert exc_info.value.code == 400
```

### Fixtures

#### Basic Fixture

```python
@pytest.fixture
def sample_user():
    """Provide sample user data."""
    return {"id": 1, "name": "Test User", "email": "test@example.com"}

def test_user_processing(sample_user):
    """Test using the fixture."""
    assert sample_user["name"] == "Test User"
```

#### Fixture Scopes

```python
# Function scope (default) - runs for each test
@pytest.fixture
def temp_file():
    with open("temp.txt", "w") as f:
        yield f
    os.remove("temp.txt")

# Module scope - runs once per module
@pytest.fixture(scope="module")
def module_db():
    db = Database(":memory:")
    db.create_tables()
    yield db
    db.close()

# Session scope - runs once per test session
@pytest.fixture(scope="session")
def shared_resource():
    resource = ExpensiveResource()
    yield resource
    resource.cleanup()
```

#### Fixture with Setup/Teardown

```python
@pytest.fixture
def database():
    """Fixture with setup and teardown."""
    # Setup
    db = Database(":memory:")
    db.create_tables()
    db.insert_test_data()
    
    yield db  # Provide to test
    
    # Teardown
    db.close()
```

#### Autouse Fixtures

```python
@pytest.fixture(autouse=True)
def reset_config():
    """Automatically runs before every test."""
    Config.reset()
    yield
    Config.cleanup()
```

#### Conftest.py for Shared Fixtures

```python
# tests/conftest.py
import pytest

@pytest.fixture
def client():
    """Shared fixture for all tests."""
    app = create_app(testing=True)
    with app.test_client() as client:
        yield client

@pytest.fixture
def auth_headers(client):
    """Generate auth headers for API testing."""
    response = client.post("/api/login", json={
        "username": "test",
        "password": "test"
    })
    token = response.json["token"]
    return {"Authorization": f"Bearer {token}"}
```

### Parametrization

#### Basic Parametrization

```python
@pytest.mark.parametrize("email,expected", [
    ("user@example.com", True),
    ("test.user@domain.co.uk", True),
    ("invalid.email", False),
    ("@example.com", False),
])
def test_email_validation(email, expected):
    """Test runs 4 times with different inputs."""
    assert is_valid_email(email) == expected
```

#### Multiple Parameters

```python
@pytest.mark.parametrize("a,b,expected", [
    (2, 3, 5),
    (0, 0, 0),
    (-1, 1, 0),
    (100, 200, 300),
])
def test_add(a, b, expected):
    """Test addition with multiple parameter sets."""
    assert add(a, b) == expected
```

#### Parametrize with IDs

```python
@pytest.mark.parametrize("value,expected", [
    pytest.param(1, True, id="positive"),
    pytest.param(0, False, id="zero"),
    pytest.param(-1, False, id="negative"),
])
def test_is_positive(value, expected):
    """Test with custom test IDs for readable output."""
    assert (value > 0) == expected
```

### Markers

```python
# Mark slow tests
@pytest.mark.slow
def test_slow_operation():
    time.sleep(5)

# Mark integration tests
@pytest.mark.integration
def test_api_integration():
    response = requests.get("https://api.example.com")
    assert response.status_code == 200

# Mark unit tests
@pytest.mark.unit
def test_unit_logic():
    assert calculate(2, 3) == 5
```

### Running Tests by Markers

```bash
# Run only fast tests
pytest -m "not slow"

# Run only integration tests
pytest -m integration

# Run integration or slow tests
pytest -m "integration or slow"

# Run tests marked as unit but not slow
pytest -m "unit and not slow"
```

---

## 4. Integration Testing

### Database Testing with SQLAlchemy

```python
import pytest
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String(50))
    email = Column(String(100), unique=True)

@pytest.fixture(scope="function")
def db_session() -> Session:
    """Create in-memory database for testing."""
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    
    SessionLocal = sessionmaker(bind=engine)
    session = SessionLocal()
    
    yield session
    
    session.close()

def test_create_user(db_session):
    """Test creating a user."""
    user = User(name="Test User", email="test@example.com")
    db_session.add(user)
    db_session.commit()
    
    assert user.id is not None
    assert user.name == "Test User"

def test_unique_email_constraint(db_session):
    """Test unique email constraint."""
    from sqlalchemy.exc import IntegrityError
    
    user1 = User(name="User 1", email="same@example.com")
    user2 = User(name="User 2", email="same@example.com")
    
    db_session.add(user1)
    db_session.commit()
    db_session.add(user2)
    
    with pytest.raises(IntegrityError):
        db_session.commit()
```

### API Integration Testing

```python
@pytest.fixture
def client():
    """Create test client."""
    app = create_app(testing=True)
    return app.test_client()

def test_get_user(client):
    """Test GET /api/users/{id}"""
    response = client.get("/api/users/1")
    assert response.status_code == 200
    assert response.json["id"] == 1

def test_create_user(client):
    """Test POST /api/users"""
    response = client.post("/api/users", json={
        "name": "Alice",
        "email": "alice@example.com"
    })
    assert response.status_code == 201
    assert response.json["name"] == "Alice"
```

### Async Testing

```python
import pytest
import asyncio

@pytest.mark.asyncio
async def test_async_function():
    """Test async function."""
    result = await async_add(2, 3)
    assert result == 5

@pytest.fixture
async def async_client():
    """Async fixture providing async test client."""
    app = create_app()
    async with app.test_client() as client:
        yield client

@pytest.mark.asyncio
async def test_api_endpoint(async_client):
    """Test using async fixture."""
    response = await async_client.get("/api/data")
    assert response.status_code == 200
```

### Mocking External Dependencies

```python
from unittest.mock import Mock, patch, MagicMock

@patch("mypackage.external_api_call")
def test_with_mock(api_call_mock):
    """Test with mocked external API."""
    api_call_mock.return_value = {"status": "success"}
    
    result = my_function()
    
    api_call_mock.assert_called_once()
    assert result["status"] == "success"

@patch("mypackage.api_call")
def test_api_error_handling(api_call_mock):
    """Test error handling with mocked exception."""
    api_call_mock.side_effect = ConnectionError("Network error")
    
    with pytest.raises(ConnectionError):
        api_call()

# Mocking context managers
@patch("builtins.open", new_callable=mock_open)
def test_file_reading(mock_file):
    """Test file reading with mocked open."""
    mock_file.return_value.read.return_value = "file content"
    
    result = read_file("test.txt")
    
    mock_file.assert_called_once_with("test.txt", "r")
    assert result == "file content"
```

### Testing Retry Behavior

```python
from unittest.mock import Mock

def test_retries_on_transient_error():
    """Test that service retries on transient failures."""
    client = Mock()
    # Fail twice, then succeed
    client.request.side_effect = [
        ConnectionError("Failed"),
        ConnectionError("Failed"),
        {"status": "ok"},
    ]
    
    service = ServiceWithRetry(client, max_retries=3)
    result = service.fetch()
    
    assert result == {"status": "ok"}
    assert client.request.call_count == 3

def test_gives_up_after_max_retries():
    """Test that service stops retrying after max attempts."""
    client = Mock()
    client.request.side_effect = ConnectionError("Failed")
    
    service = ServiceWithRetry(client, max_retries=3)
    
    with pytest.raises(ConnectionError):
        service.fetch()
    
    assert client.request.call_count == 3
```

---

## 5. Advanced Patterns

### Property-Based Testing (Hypothesis)

```python
from hypothesis import given, strategies as st

@given(st.text())
def test_reverse_twice_is_original(s):
    """Property: reversing twice returns original."""
    assert reverse_string(reverse_string(s)) == s

@given(st.text())
def test_reverse_length(s):
    """Property: reversed string has same length."""
    assert len(reverse_string(s)) == len(s)

@given(st.integers(), st.integers())
def test_addition_commutative(a, b):
    """Property: addition is commutative."""
    assert a + b == b + a

@given(st.lists(st.integers()))
def test_sorted_list_properties(lst):
    """Property: sorted list is ordered."""
    sorted_lst = sorted(lst)
    
    # Same length
    assert len(sorted_lst) == len(lst)
    
    # All elements present
    assert set(sorted_lst) == set(lst)
    
    # Is ordered
    for i in range(len(sorted_lst) - 1):
        assert sorted_lst[i] <= sorted_lst[i + 1]
```

### Time Control with Freezegun

```python
from freezegun import freeze_time
from datetime import datetime, timedelta

@freeze_time("2026-01-15 10:00:00")
def test_token_expiry():
    """Test token expires at correct time."""
    token = create_token(expires_in_seconds=3600)
    assert token.expires_at == datetime(2026, 1, 15, 11, 0, 0)

@freeze_time("2026-01-15 10:00:00")
def test_is_expired_returns_false_before_expiry():
    """Test token is not expired when within validity period."""
    token = create_token(expires_in_seconds=3600)
    assert not token.is_expired()

@freeze_time("2026-01-15 12:00:00")
def test_is_expired_returns_true_after_expiry():
    """Test token is expired after validity period."""
    token = Token(expires_at=datetime(2026, 1, 15, 11, 30, 0))
    assert token.is_expired()

def test_with_time_travel():
    """Test behavior across time using freeze_time context."""
    with freeze_time("2026-01-01") as frozen_time:
        item = create_item()
        assert item.created_at == datetime(2026, 1, 1)
        
        # Move forward in time
        frozen_time.move_to("2026-01-15")
        assert item.age_days == 14
```

### Monkeypatch for Environment

```python
def test_database_url_custom(monkeypatch):
    """Test custom database URL with monkeypatch."""
    monkeypatch.setenv("DATABASE_URL", "postgresql://localhost/test")
    assert get_database_url() == "postgresql://localhost/test"

def test_database_url_not_set(monkeypatch):
    """Test when env var is not set."""
    monkeypatch.delenv("DATABASE_URL", raising=False)
    assert get_database_url() == "sqlite:///:memory:"

def test_monkeypatch_attribute(monkeypatch):
    """Test monkeypatching object attributes."""
    config = Config()
    monkeypatch.setattr(config, "api_key", "test-key")
    assert config.get_api_key() == "test-key"
```

### Temporary Files and Directories

```python
def test_file_operations(tmp_path):
    """Test file operations with temporary directory."""
    # tmp_path is a pathlib.Path object
    test_file = tmp_path / "test_data.txt"
    
    # Save data
    test_file.write_text("Hello, World!")
    
    # Verify file exists
    assert test_file.exists()
    
    # Load and verify data
    data = test_file.read_text()
    assert data == "Hello, World!"
    # tmp_path automatically cleaned up
```

### One Behavior Per Test

```python
# BAD - testing multiple behaviors
def test_user_service():
    user = service.create_user(data)
    assert user.id is not None
    assert user.email == data["email"]
    updated = service.update_user(user.id, {"name": "New"})
    assert updated.name == "New"

# GOOD - focused tests
def test_create_user_assigns_id():
    user = service.create_user(data)
    assert user.id is not None

def test_create_user_stores_email():
    user = service.create_user(data)
    assert user.email == data["email"]

def test_update_user_changes_name():
    user = service.create_user(data)
    updated = service.update_user(user.id, {"name": "New"})
    assert updated.name == "New"
```

---

## 6. E2E & Workflow Testing

### Temporal Workflow Testing

#### Basic Workflow Test with Time-Skipping

```python
import pytest
from temporalio.testing import WorkflowEnvironment
from temporalio.worker import Worker

@pytest.fixture
async def workflow_env():
    """Create workflow environment with time-skipping."""
    env = await WorkflowEnvironment.start_time_skipping()
    yield env
    await env.shutdown()

@pytest.mark.asyncio
async def test_workflow(workflow_env):
    """Test workflow with time-skipping."""
    async with Worker(
        workflow_env.client,
        task_queue="test-queue",
        workflows=[YourWorkflow],
        activities=[your_activity],
    ):
        result = await workflow_env.client.execute_workflow(
            YourWorkflow.run,
            args,
            id="test-wf-id",
            task_queue="test-queue",
        )
        assert result == expected
```

#### Testing Activities

```python
from temporalio.testing import ActivityEnvironment

async def test_activity():
    """Test activity in isolation."""
    env = ActivityEnvironment()
    result = await env.run(your_activity, "test-input")
    assert result == expected_output
```

#### Activity Mocking

```python
@pytest.mark.asyncio
async def test_workflow_with_mocked_activities(workflow_env):
    """Test workflow with mocked activities to isolate workflow logic."""
    
    # Mock activities
    async def mock_send_email(to: str, subject: str) -> str:
        return f"mock-sent-{to}"
    
    async with Worker(
        workflow_env.client,
        task_queue="test-queue",
        workflows=[NotificationWorkflow],
        activities=[mock_send_email],  # Use mock instead of real
    ):
        result = await workflow_env.client.execute_workflow(
            NotificationWorkflow.run,
            {"email": "user@example.com"},
            id="test-notification",
            task_queue="test-queue",
        )
        assert result == "mock-sent-user@example.com"
```

#### Replay Testing for Determinism

```python
def test_workflow_replay():
    """Validate workflow determinism against production history."""
    # Load production workflow history
    history = load_workflow_history("workflow-id")
    
    # Replay workflow against history
    WorkflowReplayer.replay_workflow(
        workflow_class=YourWorkflow,
        history=history
    )
    # If replay succeeds, workflow is deterministic
```

### Page Object Model for E2E

```python
class LoginPage:
    """Page Object for login page."""
    
    def __init__(self, page):
        self.page = page
        self.email_input = '[data-testid="email"]'
        self.password_input = '[data-testid="password"]'
        self.submit_button = '[data-testid="submit"]'
    
    async def login(self, email: str, password: str):
        """Perform login action."""
        await self.page.fill(self.email_input, email)
        await self.page.fill(self.password_input, password)
        await self.page.click(self.submit_button)
    
    async def get_error_message(self) -> str:
        """Get error message if login fails."""
        return await self.page.text_content('[data-testid="error"]')

# Usage in test
async def test_login_flow(page):
    login_page = LoginPage(page)
    await login_page.login("user@example.com", "password")
    # Assert successful login
```

---

## 7. Coverage & Quality Gates

### Coverage Requirements

| Level | Target | Critical Paths |
|-------|--------|----------------|
| **Overall** | ≥80% | - |
| **Critical Paths** | 100% | Payment, auth, data export |
| **Workflows** | ≥80% | Temporal workflow logic |
| **Activities** | ≥80% | Temporal activity logic |

### Running Tests with Coverage

```bash
# Install coverage
pip install pytest-cov

# Run tests with coverage
pytest --cov=myapp tests/

# Generate HTML report
pytest --cov=myapp --cov-report=html tests/

# Fail if coverage below threshold
pytest --cov=myapp --cov-fail-under=80 tests/

# Show missing lines
pytest --cov=myapp --cov-report=term-missing tests/
```

### Coverage Configuration (pyproject.toml)

```toml
[tool.coverage.run]
source = ["myapp"]
omit = ["*/tests/*", "*/migrations/*"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
]
```

### Flaky Test Management

**Definition:** Test fails without product change, passes on rerun

**SLO:** Suite flake rate ≤1% weekly

**Quarantine Policy:**
```
1. Detect flaky test (fails then passes on rerun)
2. Assign owner and create ticket
3. Quarantine with expiry date (e.g., 7 days)
4. Fix or remove before expiry
5. Document root cause
```

```python
# Mark known flaky test
@pytest.mark.xfail(reason="Flaky due to timing issue - BUG-123", strict=False)
def test_flaky_feature():
    # Test code
    pass
```

---

## 8. CI/CD Integration

### GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Install dependencies
        run: |
          pip install -e ".[dev]"
          pip install pytest pytest-cov
      
      - name: Run unit tests
        run: |
          pytest tests/unit -m "not slow" --cov=myapp --cov-report=xml
      
      - name: Run integration tests
        run: |
          pytest tests/integration --cov=myapp --cov-report=xml --cov-append
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
```

### CI Economics

| Budget | Target |
|--------|--------|
| **PR gate** | p50 ≤ 10 min, p95 ≤ 20 min |
| **Mainline health** | ≥ 99% green builds/day |
| **Flake rate** | ≤ 1% weekly |

### Quality Gates

**Merge Gate (Pre-Merge):**
- [ ] All unit tests pass
- [ ] Coverage ≥80%
- [ ] No new flaky tests
- [ ] Lint and type check pass

**Deploy Gate (Pre-Deploy):**
- [ ] Integration tests pass
- [ ] E2E smoke tests pass
- [ ] Contract tests pass
- [ ] Security scan clean

### Contract Testing in CI

```yaml
- name: Validate OpenAPI contracts
  run: |
    # Validate API against OpenAPI spec
    schemathesis run openapi.yaml --base-url=http://localhost:8000
    
- name: Check backward compatibility
  run: |
    # Compare current spec against baseline
    openapi-diff baseline.yaml current.yaml --fail-on-incompatible
```

---

## 9. Test Organization & Maintenance

### Directory Structure

```
project/
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── models.py
│       ├── services.py
│       └── api.py
├── tests/
│   ├── conftest.py              # Shared fixtures
│   ├── __init__.py
│   ├── unit/                    # Unit tests (40-60%)
│   │   ├── __init__.py
│   │   ├── test_models.py
│   │   ├── test_services.py
│   │   └── test_utils.py
│   ├── integration/             # Integration tests (15-25%)
│   │   ├── __init__.py
│   │   ├── test_api.py
│   │   ├── test_database.py
│   │   └── test_external_services.py
│   ├── e2e/                     # End-to-end tests (5-10%)
│   │   ├── __init__.py
│   │   ├── test_user_flow.py
│   │   └── test_payment_flow.py
│   └── workflows/               # Temporal workflow tests
│       ├── __init__.py
│       ├── test_order_workflow.py
│       └── test_notification_workflow.py
├── pytest.ini
├── pyproject.toml
└── .github/
    └── workflows/
        └── test.yml
```

### Test Classes Organization

```python
class TestUserService:
    """Group related tests in a class."""
    
    @pytest.fixture(autouse=True)
    def setup(self):
        """Setup runs before each test in this class."""
        self.service = UserService()
    
    def test_create_user(self):
        """Test user creation."""
        user = self.service.create_user("Alice")
        assert user.name == "Alice"
    
    def test_delete_user(self):
        """Test user deletion."""
        user = User(id=1, name="Bob")
        self.service.delete_user(user)
        assert not self.service.user_exists(1)
```

### Test Naming Convention

Pattern: `test_<unit>_<scenario>_<expected>`

```python
# Good examples
def test_create_user_with_valid_data_returns_user():
    """Clear name describes what is being tested."""
    pass

def test_login_fails_with_invalid_password():
    """Name describes expected behavior."""
    pass

def test_api_returns_404_for_missing_resource():
    """Specific about inputs and expected outcomes."""
    pass

# Bad examples
def test_1():  # Not descriptive
    pass

def test_user():  # Too vague
    pass
```

### Regression Suite Organization

| Suite Type | Duration | Frequency | Coverage |
|------------|----------|-----------|----------|
| **Smoke** | 15-30 min | Daily | Critical paths only |
| **Targeted** | 30-60 min | Per change | Affected areas |
| **Full** | 2-4 hours | Weekly/Release | Comprehensive |
| **Sanity** | 10-15 min | After hotfix | Quick validation |

### Priority Matrix

| Priority | Description | Must Run |
|----------|-------------|----------|
| **P0 (Critical)** | Business-critical, security | Always |
| **P1 (High)** | Major features, common flows | Weekly+ |
| **P2 (Medium)** | Minor features, edge cases | Releases |
| **P3 (Low)** | Cosmetic, rarely used | Optional |

---

## 10. Troubleshooting & Best Practices

### Common Anti-Patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| Testing implementation | Breaks on refactor | Test behavior |
| Shared mutable state | Flaky tests | Isolate test data |
| sleep() in tests | Slow, unreliable | Use proper waits or freezegun |
| Everything E2E | Slow, expensive | Use test pyramid |
| Ignoring flaky tests | False confidence | Fix or quarantine |

### DO's

- ✅ Write tests first (TDD) or alongside code
- ✅ Use descriptive test names that explain behavior
- ✅ Keep tests independent and isolated
- ✅ Use fixtures for setup and teardown
- ✅ Mock external dependencies appropriately
- ✅ Parametrize tests to reduce duplication
- ✅ Test edge cases and error conditions
- ✅ Measure coverage but focus on quality
- ✅ Run tests in CI/CD on every commit
- ✅ Test one behavior per test

### DON'Ts

- ❌ Don't test implementation details
- ❌ Don't use complex conditionals in tests
- ❌ Don't ignore test failures
- ❌ Don't test third-party code
- ❌ Don't share state between tests
- ❌ Don't catch exceptions in tests (use `pytest.raises`)
- ❌ Don't use print statements (use assertions)
- ❌ Don't write tests that are too brittle

### Debugging Failing Tests

```bash
# Run with verbose output
pytest -v

# Run until first failure
pytest -x

# Run and stop on N failures
pytest --maxfail=3

# Run last failed tests
pytest --lf

# Run with debugger on failure
pytest --pdb

# Show local variables in tracebacks
pytest -l
```

### Test Performance Optimization

```python
# Use session-scoped fixtures for expensive resources
@pytest.fixture(scope="session")
def database():
    """Create database once for all tests."""
    db = create_database()
    yield db
    db.cleanup()

# Mark slow tests to skip in fast mode
@pytest.mark.slow
def test_expensive_operation():
    pass

# Run: pytest -m "not slow" for quick feedback
```

### Observability-First Testing

```python
def test_with_correlation_id():
    """Include correlation IDs for traceability."""
    correlation_id = str(uuid.uuid4())
    
    with patch("logging.Logger.info") as mock_log:
        result = process_request(data, correlation_id=correlation_id)
        
        # Verify logs include correlation ID
        mock_log.assert_any_call(
            "Processing request",
            extra={"correlation_id": correlation_id}
        )
```

---

## Quick Reference

| Pattern | Usage |
|---------|-------|
| `pytest.raises()` | Test expected exceptions |
| `@pytest.fixture()` | Create reusable test fixtures |
| `@pytest.mark.parametrize()` | Run tests with multiple inputs |
| `@pytest.mark.slow` | Mark slow tests |
| `pytest -m "not slow"` | Skip slow tests |
| `@patch()` | Mock functions and classes |
| `tmp_path` fixture | Automatic temp directory |
| `pytest --cov` | Generate coverage report |
| `freeze_time` | Control time in tests |
| `monkeypatch` | Modify environment/attributes |
| `@given()` | Property-based testing |

### pytest.ini Template

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    -v
    --strict-markers
    --tb=short
    --cov=myapp
    --cov-report=term-missing
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
    unit: marks tests as unit tests
    e2e: marks end-to-end tests
```

---

**Remember:** Tests are code too. Keep them clean, readable, and maintainable. Good tests catch bugs; great tests prevent them.
