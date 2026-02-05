# Unit Testing Checklist

## Pre-Test Setup

- [ ] Test file follows naming convention: `test_<module>.py`
- [ ] Test class follows naming convention: `Test<ClassName>` (if using classes)
- [ ] Test function follows naming convention: `test_<scenario>_<expected>`
- [ ] Conftest.py exists for shared fixtures

## Test Structure (AAA Pattern)

For each test:
- [ ] **Arrange**: Test data and preconditions clearly defined
- [ ] **Act**: Single action under test
- [ ] **Assert**: Verify one specific outcome

## Code Quality

- [ ] Each test verifies exactly one behavior
- [ ] Test name describes what is being tested
- [ ] No complex conditionals in tests
- [ ] No print statements (use assertions)
- [ ] Tests are independent (no shared state)

## Coverage

- [ ] All public methods have tests
- [ ] Edge cases covered (empty, None, boundaries)
- [ ] Error paths tested with `pytest.raises()`
- [ ] Critical paths have 100% coverage

## Fixtures

- [ ] Reusable fixtures defined in conftest.py
- [ ] Fixture scope optimized (function/module/session)
- [ ] Setup/teardown properly implemented
- [ ] No expensive operations in function-scoped fixtures

## Mocking

- [ ] External dependencies mocked appropriately
- [ ] Mock assertions verify expected calls
- [ ] Side effects tested for retry/error scenarios
- [ ] `autospec=True` used where API compliance matters

## Parametrization

- [ ] Multiple similar inputs use `@pytest.mark.parametrize`
- [ ] Test IDs provided for readability
- [ ] Edge cases included in parameter sets

## Example Verification

```python
# ✅ Good
def test_calculate_discount_with_valid_input():
    # Arrange
    price = 100
    discount = 10
    
    # Act
    result = calculate_discount(price, discount)
    
    # Assert
    assert result == 90

# ❌ Bad
def test_discount():
    result = calculate_discount(100, 10)
    assert result == 90
    result2 = calculate_discount(200, 20)
    assert result2 == 160
```

## Quick Check

- [ ] Tests run in < 100ms each
- [ ] `pytest -m "not slow"` passes quickly
- [ ] Coverage ≥ 80% for tested module
