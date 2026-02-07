---
name: python-web-api-standards
description: Basic project standards for production web APIs with Litestar and Vertical Slice Architecture. Use always when working with Python production web APIs.
---

# Python Project Standards

## When to Use This Skill

In any Python APIs development with Litestar and Vertical Slice Architecture, these are fundamental rules that should always be applied.
❌ Do NOT apply to:

- One-off data migration scripts
- Admin/maintenance CLI tools
- Data Science / ML Pipelines
- Libraries / Packages
- Background Jobs / Workers

✅ DO apply to:

- New LiteStar endpoints
- Refactoring existing LiteStar features
- Creating new feature slices

## HOW TO USE THIS SKILL

### Priority Levels

- **[HARD]** — Never violate (breaks project standards)
- **[DEFAULT]** — Follow by default (override with justification)
- **[CONTEXTUAL]** — Depends on task (use your judgment)

### Decision Framework

1. Apply [HARD] constraints unconditionally
2. Follow [DEFAULT] rules unless you have a reason not to
3. Evaluate [CONTEXTUAL] rules based on:
   - Criticality of the feature
   - Performance requirements
   - Maintenance burden
   - Team familiarity

## CONSTRAINTS BY PRIORITY

### [HARD] TECHNOLOGY STACK

**[HARD]** MUST use Python >= 3.13  
**[HARD]** MUST use `uv` package manager (NOT pip, NOT poetry)  
**[HARD]** MUST use LiteStar + Granian + asyncio  
**[HARD]** MUST use `msgspec` for serialization (NEVER Pydantic)  
**[HARD]** MUST use `asyncpg` direct (separate DB skill handles specifics)  
**[HARD]** MUST use Dockerfile with staged and slim build  

### [HARD] DATA FORMATS

**[HARD]** MUST use UUID v7 for IDs (not v4, not integers)  
**[HARD]** MUST use ISO-8601 UTC for timestamps (no Unix timestamps, no naive datetimes)  
**[HARD]** MUST use `msgspec.Struct` with `msgspec.json.encode/decode`  

### [HARD] ARCHITECTURE

**[HARD]** MUST use Vertical Slice Architecture (VSA)

- Group by FEATURE, not by layer
- Each slice = self-contained plugin
- NO monolithic services/repositories

### Code Structure Per Feature Slice

```
src/features/avatar/create_via_wizard/
├── router.py      # HTTP → Command (NO business logic here)
├── schema.py      # msgspec.Struct for external API contract
├── command.py     # Internal DTO (dataclass/msgspec for business logic)
├── handler.py     # Pure business logic (NO LiteStar/HTTP knowledge)
├── queries.py     # SQL queries as CONSTANTS
└── models.py      # SQLAlchemy (if slice-specific)
```

### [HARD] PERFORMANCE - NEVER DO THIS

**[HARD]** NEVER create connection pools in handlers  
**[HARD]** NEVER use synchronous code in async context  
**[HARD]** NEVER use exceptions for control flow  
**[HARD]** NEVER skip connection pooling  
**[DEFAULT]** SHOULD use O(n²) when O(n) exists if nessesary

### [HARD] SECURITY

**[HARD]** NEVER put auth tokens/user_id from token into `schema.py`  
**[HARD]** NEVER log sensitive data (passwords, tokens, PII, full emails)  
**[HARD]** NEVER skip `command.py` (Schema = external, Command = internal + enriched)  

### [HARD] SERIALIZATION

**[HARD]** NEVER use Pydantic in this project  
**[HARD]** NEVER use dataclasses for API responses (use msgspec.Struct)  

### [HARD] CODE PATTERNS - FORBIDDEN

**[HARD]** NEVER use mutable defaults:

```python
# ❌ FORBIDDEN
def add_item(item, items=[]):
    items.append(item)
    return items

# ✅ REQUIRED
def add_item(item, items: list | None = None) -> list:
    if items is None:
        items = []
    items.append(item)
    return items
```

**[HARD]** NEVER use broad exception handling:

```python
# ❌ FORBIDDEN
try:
    risky_operation()
except:  # Catches KeyboardInterrupt!
    pass

# ✅ REQUIRED
try:
    risky_operation()
except ValueError as e:
    logger.error(f"Invalid value: {e}")
    raise
```

**[HARD]** NEVER use global mutable state:

```python
# ❌ FORBIDDEN - Breaks with multiple instances
_avatar_cache = {}

async def create_avatar(cmd: CreateCommand):
    _avatar_cache[cmd.id] = ...

# ✅ REQUIRED - Stateless handlers
async def create_avatar(cmd: CreateCommand, pool: asyncpg.Pool):
    # No instance state, can scale horizontally
    ...
```

### [DEFAULT] TYPE SAFETY

**[DEFAULT]** PREFER full type coverage on all functions/methods:

```python
# ✅ PREFERRED
def handle_create(cmd: CreateCommand, pool: asyncpg.Pool) -> AvatarResponse:
    """Create avatar via wizard flow."""
    ...

# ❌ AVOID
def handle_create(cmd, pool):
    ...
```

### [DEFAULT] DOCSTRINGS

**[DEFAULT]** SHOULD use Google Style docstrings for non-obvious logic:

```python
# ✅ GOOD - Complex logic explained
def calculate_reputation_score(user: User, posts: list[Post]) -> int:
    """
    Calculate user reputation using engagement metrics.
    
    Algorithm:
    - Posts with >100 likes: weight 2x
    - Comments: weight 0.5x
    - Decay factor: 7 days
    
    Args:
        user: User entity with profile data
        posts: List of user's posts
    
    Returns:
        Reputation score (0-1000 range)
    
    Raises:
        ValueError: If posts list is empty
        DatabaseError: If connection fails
    """
    ...

# ✅ GOOD - Obvious function, no docstring needed
def is_even(n: int) -> bool:
    return n % 2 == 0

# ❌ BAD - Useless docstring
def add(a: int, b: int) -> int:
    """Add two numbers."""  # Obvious from code!
    return a + b
```

**Rule:** Docstring explains WHY and complex HOW, NOT WHAT line-by-line.

### [DEFAULT] MEMORY OPTIMIZATION

**[DEFAULT]** PREFER `__slots__` in regular classes:

```python
# ✅ PREFERRED
class Avatar:
    """Avatar entity."""
    __slots__ = ('id', 'name', 'color', 'created_at')
    
    def __init__(self, id: UUID, name: str, color: str, created_at: datetime):
        self.id = id
        self.name = name
        self.color = color
        self.created_at = created_at

# ❌ AVOID - Wasting memory with __dict__
class Avatar:
    def __init__(self, id: UUID, name: str):
        self.id = id
        self.name = name
```

**NOTE:** `msgspec.Struct` uses `slots=True` by default.

### [HARD] STARTUP LIFECYCLE

**[HARD]** MUST initialize pools at startup in `lifespan`:

```python
from contextlib import asynccontextmanager
from litestar import Litestar

@asynccontextmanager
async def app_lifespan(app: Litestar):
    # === STARTUP ===
    db_pool = await asyncpg.create_pool(
        dsn=settings.DATABASE_URL,
        min_size=10,
        max_size=20,
        command_timeout=10.0,
        server_settings={'jit': 'off'},
    )
    app.state.db_pool = db_pool
    
    http_client = aiohttp.ClientSession(
        timeout=aiohttp.ClientTimeout(total=5, connect=2),
        connector=aiohttp.TCPConnector(limit=100, keepalive_timeout=30)
    )
    app.state.http = http_client
    
    background_tasks = set()
    app.state.tasks = background_tasks
    
    yield
    
    # === SHUTDOWN ===
    for task in background_tasks:
        task.cancel()
    await asyncio.gather(*background_tasks, return_exceptions=True)
    await db_pool.close()
    await http_client.close()

app = Litestar(route_handlers=[...], lifespan=[app_lifespan])
```

### [DEFAULT] DEPENDENCY INJECTION

**[DEFAULT]** SHOULD use LiteStar built-in DI (lightweight only):

```python
from litestar.datastructures import State
from litestar.params import Dependency

async def get_db_pool(state: State) -> asyncpg.Pool:
    """Inject database pool from app state."""
    return state.db_pool

# schema.py
class CreateAvatarRequest(Struct):
    name: str
    color: str

# command.py
class CreateAvatarCommand:
    __slots__ = ("name", "color", "user_id")

    def __init__(self, name: str, color: str, user_id: UUID):
        self.name = name
        self.color = color
        self.user_id = user_id


@post("/avatars")
async def create_avatar(
    data: CreateAvatarRequest,
    pool: asyncpg.Pool = Dependency(get_db_pool),
) -> AvatarResponse:
    cmd = CreateAvatarCommand(
        name=data.name,
        color=data.color,
        user_id=current_user.id,
    )

    return await handle_create_avatar(cmd, pool)
```

### [HARD] ASYNC PATTERNS

**[HARD]** MUST use context managers for resource management:

```python
# ✅ REQUIRED
class DatabasePool:
    async def __aenter__(self):
        self.pool = await asyncpg.create_pool(...)
        return self
    
    async def __aexit__(self, *args):
        await self.pool.close()
```

**[HARD]** MUST make all IO operations async:

```python
# ✅ REQUIRED
async def fetch_user(pool: asyncpg.Pool, user_id: UUID) -> User | None:
    ...
```

### [DEFAULT] DATABASE PATTERNS

**[DEFAULT]** SHOULD extract SQL queries to separate `queries.py`:

**[DEFAULT]** PREFER direct SQL → msgspec mapping (no dict intermediate):

```python
from msgspec import Struct
from .queries import GET_AVATAR_BY_ID

class Avatar(Struct):
    id: UUID
    name: str
    color: str
    user_id: UUID
    created_at: datetime

# ✅ PREFERRED - Zero-copy mapping
async def get_avatar(pool: asyncpg.Pool, avatar_id: UUID) -> Avatar | None:
    row = await pool.fetchrow(GET_AVATAR_BY_ID, avatar_id)
    if not row:
        return None
    return Avatar(*row)

# ❌ AVOID - Unnecessary dict conversion
async def get_avatar(pool: asyncpg.Pool, avatar_id: UUID) -> Avatar | None:
    row = await pool.fetchrow(GET_AVATAR_BY_ID, avatar_id)
    data = dict(row)  # Extra allocation
    return Avatar(**data)
```

### [HARD] SECURITY & MIDDLEWARE

**[HARD]** MUST apply appropriate middleware to every endpoint:

- [HARD] `@middleware` - Request/response processing
- [HARD] `@rate_limiter` - DoS protection
- [HARD] `@auth` - Authentication (if not public)
- [DEFAULT] `@csrf` - CSRF tokens (for state-changing operations) if nessesary

### [DEFAULT] INPUT VALIDATION

**[DEFAULT]** SHOULD validate at API boundary:

```python
from msgspec import Struct, field

# ✅ PREFERRED
class CreateAvatarRequest(Struct):
    name: str = field(min_length=1, max_length=50)
    color: str = field(pattern=r'^#[0-9A-Fa-f]{6}$')

# ❌ AVOID - Trusting user input
class CreateAvatarRequest(Struct):
    name: str  # Can be empty or 10MB string!
```

### [DEFAULT] OBSERVABILITY

**[DEFAULT]** SHOULD use structured logging:

```python
import structlog

logger = structlog.get_logger()

# ✅ PREFERRED - Structured logs
logger.info(
    "avatar_created",
    user_id=str(cmd.user_id),
    avatar_id=str(avatar.id),
    duration_ms=elapsed_time,
    feature="avatar.create_wizard"
    # NO sensitive data (passwords, tokens, PII)
)

# ❌ AVOID - Unstructured strings
logger.info(f"Created avatar {avatar.id} for user {user.id}")
```

**[CONTEXTUAL]** CONSIDER telemetry for critical paths:

```python
# For user-facing or critical operations
with tracer.start_as_current_span("db.create_avatar"):
    result = await pool.fetch(CREATE_QUERY, ...)
```

### [CONTEXTUAL] ERROR HANDLING & RESILIENCE

**[CONTEXTUAL]** CONSIDER retry mechanism for network calls:

```python
from tenacity import retry, stop_after_attempt, wait_exponential

# Use for idempotent external API calls
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
async def fetch_external_api(client: aiohttp.ClientSession, url: str):
    async with client.get(url) as response:
        return await response.json()
```

**[CONTEXTUAL]** CONSIDER circuit breaker for critical external services:

```python
from aiobreaker import CircuitBreaker

# Use for payment gateways, critical third-party APIs
payment_breaker = CircuitBreaker(fail_max=5, timeout_duration=60)

@payment_breaker
async def charge_payment(amount: Decimal, card_token: str):
    # If 5 consecutive failures → open circuit for 60 sec
    ...
```

**[CONTEXTUAL]** CONSIDER graceful degradation for non-critical services:

```python
# Use when feature can work without external data
async def get_user_stats(user_id: UUID) -> UserStats:
    try:
        external_data = await fetch_analytics_api(user_id)
    except ExternalServiceError as e:
        logger.warning(f"Analytics unavailable: {e}")
        external_data = None  # Continue with defaults
    
    return UserStats(user_id=user_id, analytics=external_data)
```

### [DEFAULT] BACKPRESSURE HANDLING

**[DEFAULT]** SHOULD use bounded queues for task processing:

```python
import asyncio

# ✅ PREFERRED - Bounded queue prevents OOM
task_queue = asyncio.Queue(maxsize=1000)

async def producer():
    await task_queue.put(item)  # Blocks if queue is full

# ❌ AVOID - Unbounded queue
task_queue = asyncio.Queue()  # Can consume all memory
```

### [DEFAULT] IMMUTABLE DATA STRUCTURES

**[DEFAULT]** PREFER immutable types for constants:

```python
# ✅ PREFERRED
ALLOWED_COLORS: frozenset[str] = frozenset({'red', 'blue', 'green'})
DEFAULT_PERMISSIONS: tuple[str, ...] = ('read', 'write')

# ❌ AVOID - Mutable constants
ALLOWED_COLORS = {'red', 'blue', 'green'}  # Can be modified!
```

### [CONTEXTUAL] CQRS LITE - SEPARATE READ/WRITE MODELS

**[CONTEXTUAL]** CONSIDER separate models when query complexity differs:

```python
# Use when read queries need denormalization or aggregation
class AvatarWriteModel(Struct):
    """For INSERT/UPDATE - normalized"""
    id: UUID
    name: str
    user_id: UUID

class AvatarListReadModel(Struct):
    """For GET /avatars - denormalized, optimized"""
    id: UUID
    name: str
    user_name: str  # Pre-joined!
    created_at: datetime
    likes_count: int  # Pre-aggregated!
```

### [DEFAULT] ARCHITECTURE PRINCIPLES

**[HARD]** MUST follow VSA (Vertical Slice Architecture):

- Group by feature, not by layer
- Each slice is self-contained

**[DEFAULT]** PREFER one handler = one action (Single Responsibility)

**[DEFAULT]** SHOULD avoid God Services with 20+ methods

**[DEFAULT]** SHOULD inject only needed dependencies

### [CONTEXTUAL] PERFORMANCE OPTIMIZATION

**[CONTEXTUAL]** CONSIDER caching for expensive computations:

```python
from functools import lru_cache

# Use for pure functions with repeated calls
@lru_cache(maxsize=128)
def calculate_expensive_metric(data: tuple) -> float:
    ...
```

**[CONTEXTUAL]** CONSIDER generators for large datasets:

```python
# Use when processing large result sets
async def stream_users(pool: asyncpg.Pool):
    async with pool.acquire() as conn:
        async for row in conn.cursor(GET_ALL_USERS):
            yield User(*row)
```

**[DEFAULT]** PREFER built-in functions over manual loops:

```python
# ✅ PREFERRED
total = sum(item.price for item in items)
names = [user.name for user in users]

# ❌ AVOID
total = 0
for item in items:
    total += item.price
```

## PERFORMANCE CHECKLIST

- [HARD] Stateless patterns (no shared mutable state)
- [HARD] Use `asyncpg.Pool` (not one-off connections)
- [DEFAULT] Use `msgspec` with `slots=True` (default)
- [CONTEXTUAL] Cache expensive computations (`functools.lru_cache`)
- [CONTEXTUAL] Use generators for large datasets
- [DEFAULT] Direct SQL → msgspec mapping (no dict intermediate)
- [DEFAULT] Built-in functions over manual loops
- [HARD] Aggressive timeouts on all IO operations
- [DEFAULT] Bounded queues/buffers for backpressure

## SOLID PRINCIPLES

**[DEFAULT]** Single Responsibility: One handler = one action  
**[DEFAULT]** Open/Closed: Extend via new slices, don't modify existing  
**[DEFAULT]** Dependency Inversion: Inject `asyncpg.Pool`, not concrete DB class  
**[DEFAULT]** DRY: Extract repeated code immediately  

## WORKFLOW

1. [HARD] Check if task is IN SCOPE (see top of document)
2. [DEFAULT] Read existing project structure FIRST
3. [DEFAULT] If no structure exists, propose VSA layout
4. [DEFAULT] Follow existing patterns (don't introduce new styles)
5. [DEFAULT] For DB operations → use separate DB skill
6. [HARD] Always type-hint everything
7. [DEFAULT] Log all network/DB/critical operations
8. [DEFAULT] All SQL queries in `queries.py` as CONSTANTS
9. [DEFAULT] Use structured logging (structlog)

## RELATED SKILLS

- `python-database-standards` - For asyncpg queries, migrations, pooling
