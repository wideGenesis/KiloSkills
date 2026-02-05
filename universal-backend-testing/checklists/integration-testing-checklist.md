# Integration Testing Checklist

## Pre-Test Setup

- [ ] Test database configured (sqlite :memory: or test container)
- [ ] External services mocked or use test endpoints
- [ ] Migration scripts ready for test schema
- [ ] Test data fixtures prepared

## Database Testing

- [ ] In-memory database for speed (`sqlite:///:memory:`)
- [ ] Transactions rollback after each test
- [ ] Unique constraints tested
- [ ] Foreign key constraints verified
- [ ] Index usage validated

## API Testing

- [ ] All endpoints have at least one test
- [ ] HTTP status codes verified
- [ ] Response schema validated
- [ ] Error responses tested (400, 401, 404, 500)
- [ ] Request/response headers checked when relevant

## Async Testing

- [ ] Async fixtures use `@pytest.fixture` + `async`
- [ ] Tests marked with `@pytest.mark.asyncio`
- [ ] Async mocks use `assert_awaited_once()`
- [ ] Event loops properly managed

## External Dependencies

- [ ] Third-party APIs mocked
- [ ] Service boundaries tested
- [ ] Timeout scenarios covered
- [ ] Circuit breaker behavior validated

## Retry & Error Handling

- [ ] Transient failures trigger retry
- [ ] Max retry attempts respected
- [ ] Permanent errors don't retry
- [ ] Backoff strategy verified

## Test Data

- [ ] Test data isolated (no shared datasets)
- [ ] Data cleanup after tests
- [ ] Factory pattern used for complex data
- [ ] Edge case data prepared

## Example Verification

```python
# ✅ Database Test
@pytest.fixture
def db_session():
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    session = sessionmaker(bind=engine)()
    yield session
    session.close()

def test_create_user(db_session):
    user = User(name="Test", email="test@example.com")
    db_session.add(user)
    db_session.commit()
    
    assert user.id is not None
    assert user.created_at is not None

# ✅ API Test
def test_create_user_endpoint(client):
    response = client.post("/api/users", json={
        "name": "Alice",
        "email": "alice@example.com"
    })
    
    assert response.status_code == 201
    assert response.json["name"] == "Alice"
```

## Quick Check

- [ ] Tests run in < 5 seconds each
- [ ] Database properly cleaned between tests
- [ ] No real external service calls
- [ ] Coverage tracks integration paths
