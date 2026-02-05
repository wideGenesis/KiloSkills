# E2E & Workflow Testing Checklist

## Pre-Test Setup

- [ ] Full application stack running (or Docker Compose)
- [ ] Test database seeded with realistic data
- [ ] External service sandboxes configured
- [ ] Browser/HTTP client configured

## Temporal Workflow Testing

- [ ] `WorkflowEnvironment` with time-skipping configured
- [ ] Activity mocking for external dependencies
- [ ] Replay tests for determinism validation
- [ ] Coverage ≥ 80% for workflow logic
- [ ] Coverage ≥ 80% for activity logic

### Time-Skipping Tests

- [ ] `WorkflowEnvironment.start_time_skipping()` used
- [ ] Long workflows complete in seconds
- [ ] Timer/sleep behavior validated
- [ ] Workflow state transitions verified

### Activity Isolation

- [ ] Activities tested with `ActivityEnvironment`
- [ ] External calls mocked in activities
- [ ] Activity failures trigger proper compensation
- [ ] Retry policies tested

### Replay Testing

- [ ] Production history exported
- [ ] `WorkflowReplayer` validates changes
- [ ] Determinism verified before deployment
- [ ] Version compatibility checked

## Page Object Model (E2E)

- [ ] Page objects defined for each major page
- [ ] Selectors use data-testid attributes
- [ ] Actions encapsulated in page methods
- [ ] Assertions in tests, not page objects

## Critical User Journeys

- [ ] Login flow tested
- [ ] Core business workflows covered
- [ ] Payment/order flows validated
- [ ] Error recovery paths tested

## Test Data Management

- [ ] Test users with known state
- [ ] Data setup/teardown per test
- [ ] No dependencies on previous test state
- [ ] Parallel execution support

## Observability

- [ ] Correlation IDs tracked
- [ ] Logs captured on failure
- [ ] Screenshots saved for UI failures
- [ ] Traces validated for distributed flows

## Example Verification

```python
# ✅ Temporal Workflow Test
@pytest.mark.asyncio
async def test_order_workflow(workflow_env):
    async with Worker(
        workflow_env.client,
        task_queue="orders",
        workflows=[OrderWorkflow],
        activities=[process_payment_mock, send_email_mock],
    ):
        result = await workflow_env.client.execute_workflow(
            OrderWorkflow.run,
            {"product_id": "123", "quantity": 2},
            id="order-test-1",
            task_queue="orders",
        )
        
        assert result.status == "completed"
        assert result.total == 200.00

# ✅ Page Object Pattern
class CheckoutPage:
    def __init__(self, page):
        self.page = page
        self.card_input = '[data-testid="card-number"]'
        self.pay_button = '[data-testid="pay-button"]'
    
    async def enter_payment(self, card_number):
        await self.page.fill(self.card_input, card_number)
    
    async def complete_payment(self):
        await self.page.click(self.pay_button)

async def test_checkout_flow(page):
    checkout = CheckoutPage(page)
    await checkout.enter_payment("4242424242424242")
    await checkout.complete_payment()
    
    assert await page.is_visible('[data-testid="success"]')
```

## Quick Check

- [ ] E2E tests < 10% of total test suite
- [ ] Tests run in < 30 seconds each
- [ ] Critical paths prioritized
- [ ] No test interdependencies
