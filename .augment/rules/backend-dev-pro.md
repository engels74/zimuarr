---
type: "agent_requested"
description: "Litestar / Granian / msgspec / SQLAlchemy async — Python 3.14 backend coding guidelines"
---
# Litestar / Granian / msgspec / SQLAlchemy Async — Backend Reference

This is a fully-typed, async-first, Rust-accelerated Python backend built for throughput and correctness. **Litestar** is the application framework, **msgspec** is the serialization/validation core (not Pydantic), **Granian** is the Rust HTTP server (not uvicorn/gunicorn), **SQLAlchemy 2.0 async over asyncpg** is the persistence layer (usually via Advanced Alchemy's repository/service layer), and the toolchain is the Astral trio plus DetachHead's checker — **uv**, **ruff**, and **basedpyright**. Optimize for: async everywhere (never block the event loop), msgspec `Struct`s and SQLAlchemy DTOs instead of hand-written serializers, eager relationship loading (lazy loading raises under async), long-lived connection pools and HTTP clients, precise type annotations that a checker verifies, and the modern stdlib idioms (`pathlib`, `zoneinfo`, `tomllib`, structured concurrency).

The single biggest way an agent writes wrong-but-plausible code here is by importing habits from FastAPI / Pydantic / Flask / Django, or from old Python. Get these reflexes right and most of the rest follows:

- Use `msgspec.Struct`, **not** `pydantic.BaseModel`, for request/response and internal models.
- Run the app with `litestar run` + `GranianPlugin()`, **not** `uvicorn`/`gunicorn`.
- Inject dependencies with `Provide(...)` in a `dependencies={}` mapping, **not** FastAPI's `Depends()`.
- Query with `select()` + `session.execute()`/`session.scalars()`, **not** the legacy `session.query(...)`.
- Return the struct/model and let the framework encode it — never call `.dict()` / `.model_dump()`.
- Never access an unloaded relationship under async (`MissingGreenlet`); **eager-load**.
- Never instantiate `httpx.AsyncClient()` per request; keep one long-lived client on `app.state`.
- Use built-in generics + `X | None`; never `typing.List` / `Optional`.
- Use `uv`/`ruff`/`basedpyright`; never `pip`/`poetry`/`black`/`isort`/`flake8`.

This is a capability reference. Every code block is copy-ready on the stack below. Version tags like `(Python 3.12)` mark the floor a feature needs; assume the floor is met.

---

## Stack snapshot

- **Research basis:** current official docs, release notes, specifications, changelogs, and primary repositories (mid-2026).

| Component | Current stable | Floor | Notes |
|---|---|---|---|
| Python | 3.14.x (3.14.6) | 3.14 | Deferred annotations (PEP 649/749) on by default; free-threading officially supported (PEP 779) but opt-in; JIT experimental |
| Litestar | 2.24.0 | 2.x | Production-ready, rigorously typed, msgspec-based ASGI framework; 3.0 not yet stable |
| Granian | 2.7.x | 2.x | Rust HTTP server (BSD-3); ASGI/RSGI/WSGI; HTTP/1.1 + HTTP/2; free-threaded (cp314t) wheels |
| litestar-granian | 0.15.x | 0.15 | First-party `GranianPlugin` wiring Granian into the `litestar` CLI (Python ≥3.10) |
| msgspec | 0.21.x | 0.19+ | JSON/MessagePack/YAML/TOML; `Struct` type; cp314 + free-threaded wheels |
| SQLAlchemy | 2.0.x (2.0.51) | 2.0 | Target 2.0; 2.1 still beta |
| asyncpg | 0.31.x | — | PostgreSQL 9.5–18; cp314 wheels |
| advanced-alchemy | 1.11.x | — | Repository/service layer + Litestar plugin |
| Alembic | 1.18.x | — | Async + `pyproject` templates |
| structlog | 26.x | — | Structured logging |
| httpx | 0.28.x | — | Async HTTP client; 1.0 still pre-release |
| uv | 0.11.x | — | Packaging / project management (PEP 735 groups) |
| ruff | 0.15.x | — | Lint + format; 2026 style guide since 0.15.0 |
| basedpyright | 1.39.x | — | Type checker; `recommended` mode is its default |
| mypy | 2.x | — | Mature alternative checker (2.0, May 2026: experimental parallel `--num-workers N`) |
| pytest | 9.x | — | + `pytest-asyncio`; `hypothesis` for property tests |

---

## Python 3.14 language baseline

Assume the 3.14 floor. The features that change how you write code:

### Deferred annotations (PEP 649 / PEP 749)

Annotations on functions, classes, and modules are **no longer evaluated at definition time** — they are stored in a compiler-generated `__annotate__` function and computed lazily on first access. Forward references "just work"; you do **not** write `from __future__ import annotations` (that directive still works but is redundant and deprecated).

```python
# Forward references need no quotes and no __future__ import.
class Node:
    def __init__(self, parent: Node | None, children: list[Node]) -> None:
        self.parent = parent
        self.children = children
```

Two runtime-resolution caveats that bite on this stack:

- **Litestar builds signature models at startup by resolving annotations at runtime.** The injected types (a handler's `data` parameter, its return type, DI types) must be importable in module scope. Do **not** hide handler/DI types behind `if TYPE_CHECKING:` — the framework needs them at runtime and will raise `NameError`.
- **SQLAlchemy `Mapped[...]` and Advanced Alchemy base modules introspect annotations at class-creation time.** Do **not** put `from __future__ import annotations` in model modules. Handlers/services/tests may use it freely, but it's redundant on 3.14.

To introspect annotations at runtime, use the `annotationlib` module rather than reading `__annotations__` directly — it handles forward references via `Format.VALUE` / `Format.FORWARDREF` / `Format.STRING`:

```python
import annotationlib

def process(items: list[int], limit: int = 10) -> bool: ...

hints = annotationlib.get_annotations(process, format=annotationlib.Format.FORWARDREF)
```

### Template strings — t-strings (PEP 750)

A `t"..."` literal looks like an f-string but evaluates to a `string.templatelib.Template`, **not** a `str`. It exposes the static string parts and interpolated values separately, so a library can sanitize before rendering — making injection-safe interpolation structurally possible.

```python
from string.templatelib import Template, Interpolation
import html

def html_escape(template: Template) -> str:
    parts: list[str] = []
    for item in template:
        if isinstance(item, Interpolation):
            parts.append(html.escape(str(item.value)))
        else:  # static string segment
            parts.append(item)
    return "".join(parts)

user = "<script>alert('xss')</script>"
safe = html_escape(t"<p>Hello, {user}</p>")
# "<p>Hello, &lt;script&gt;alert('xss')&lt;/script&gt;</p>"
```

Use t-strings when authoring a SQL/HTML/shell/logging boundary; use f-strings for ordinary formatting. A `Template` is **not** a `str` — do not return a t-string where a `str` is expected, and there is no backport (`t"..."` is a `SyntaxError` on 3.13 and earlier).

### f-strings for everything else

f-strings are the one true string-formatting mechanism. Never use `%` or `str.format()` in new code (the one exception is lazy logging arguments — see [Logging](#logging-structlog)).

```python
value, name, width = 42.12345, "widget", 8
print(f"{name}: {value:.2f}")   # widget: 42.12
print(f"{value=:.1f}")          # value=42.1   (self-documenting)
print(f"{name:>{width}}")       # right-align in a computed width
```

### `except` without parentheses (PEP 758)

Drop the parentheses around a tuple of exception types — but only when there is no `as` clause.

```python
try:
    risky()
except ValueError, KeyError:            # no parentheses needed
    handle()
except (OSError, TimeoutError) as exc:  # parentheses REQUIRED with `as`
    log(exc)
```

### Free-threading, subinterpreters, JIT

- **Free-threading (PEP 779)** is officially supported but **opt-in** via a separate `python3.14t` binary — not the default, and single-threaded code pays a ~5–10% penalty. Granian ships free-threaded wheels but its README warns that no-GIL support is "still experimental and highly discouraged in production," and that Granian "will refuse to start" if the GIL gets re-enabled on a free-threaded build. This async I/O-bound stack scales via the event loop and Granian workers, not threads — **do not** design around no-GIL in production.
- **Subinterpreters (PEP 734)** land in the stdlib as `concurrent.interpreters` — a niche CPU-offload tool, not part of the request path.
- **JIT** remains experimental and off by default. Do not rely on it.

Error messages continue to sharpen (misspelled-keyword suggestions), the REPL has syntax highlighting and import autocompletion, and several stdlib CLIs emit color — all default behavior, nothing to configure.

---

## The type system

Modern Python is gradually but seriously typed. Write annotations everywhere and run a checker in CI.

### Built-in generics and unions — no `typing` imports for the basics

```python
# RIGHT
def totals(rows: list[dict[str, int]]) -> tuple[int, int]: ...
def find(key: str) -> int | None: ...          # not Optional[int]

# WRONG — do not import these:
# from typing import List, Dict, Tuple, Optional, Union
```

Use `list`, `dict`, `tuple`, `set`, `frozenset` directly as generics and `X | Y` / `X | None` for unions and optionals.

### The `type` statement and PEP 695 generics

Declare type parameters inline with brackets. Do not create `TypeVar` objects by hand.

```python
from collections.abc import Callable

type UserId = int
type Vector = list[float]
type Result[T] = T | Exception
type Handler[T, R] = Callable[[T], R]

def first[T](items: list[T]) -> T | None:       # generic function
    return items[0] if items else None

class Stack[T]:                                  # generic class, no Generic[T] base
    def __init__(self) -> None:
        self._items: list[T] = []
    def push(self, item: T) -> None:
        self._items.append(item)

def largest[T: (int, float)](values: list[T]) -> T:   # constrained
    return max(values)

class Repository[ModelT: "Entity"]:                    # upper bound
    def __init__(self, model: type[ModelT]) -> None:
        self.model = model
```

### `Self`, `override`, and structural typing

```python
from typing import Self, override, Protocol, runtime_checkable

class QueryBuilder:
    def __init__(self) -> None:
        self._conds: list[str] = []
    def where(self, cond: str) -> Self:        # returns the concrete subclass type
        self._conds.append(cond)
        return self

class Base:
    def handle(self) -> None: ...

class Worker(Base):
    @override                                  # checker errors if this doesn't override
    def handle(self) -> None: ...

@runtime_checkable
class Readable(Protocol):                       # prefer Protocol over ABCs for "has these methods"
    def read(self, size: int = -1, /) -> bytes: ...
```

basedpyright's `recommended` mode flags missing `@override`. Use `TypeIs`/`TypeGuard` for narrowing.

### TypedDict, Literal, Final, Annotated, typed `**kwargs`

```python
from typing import (
    TypedDict, Required, NotRequired, ReadOnly, Literal, Final, Annotated, Unpack,
)

class UserPayload(TypedDict):
    id: ReadOnly[int]                # checker forbids reassignment (PEP 705)
    name: Required[str]
    nickname: NotRequired[str]       # key may be absent

Mode = Literal["r", "w", "a"]        # only these exact values
MAX_RETRIES: Final = 3               # reassignment is a type error
Port = Annotated[int, "TCP port 1-65535"]   # metadata for pydantic/FastAPI/msgspec/etc.

class RequestOpts(TypedDict):
    timeout: float
    retries: NotRequired[int]

def fetch(url: str, **opts: Unpack[RequestOpts]) -> bytes: ...   # typed **kwargs (PEP 692)
```

### Checker-assisted development

```python
from typing import assert_type

x = first([1, 2, 3])
assert_type(x, int | None)   # static assertion; checker errors on mismatch
# reveal_type(x)             # checker prints inferred type; remove before commit
```

---

## Data modeling: msgspec first

Litestar's serialization core **is** msgspec. Define request/response and internal models as `msgspec.Struct` by default. `Struct` is a C-level slotted type that is dramatically faster than the alternatives — per msgspec's benchmarks, "In benchmarks msgspec decodes and validates JSON faster than orjson can decode it alone," Struct operations run "roughly 5x to 60x faster than the alternatives," and versus Pydantic v2 the author measured "roughly 6.4X faster for decode and 1.6X faster for encode."

```python
import msgspec
from typing import Annotated
from datetime import datetime
from uuid import UUID

# Reusable constrained type aliases via Annotated + Meta
PositiveInt = Annotated[int, msgspec.Meta(gt=0)]
Email = Annotated[str, msgspec.Meta(pattern=r"[^@]+@[^@]+\.[^@]+", max_length=254)]
Slug = Annotated[str, msgspec.Meta(pattern=r"^[a-z0-9-]+$", max_length=100)]

class UserCreate(msgspec.Struct, kw_only=True, forbid_unknown_fields=True):
    email: Email
    display_name: Annotated[str, msgspec.Meta(min_length=1, max_length=64)]
    password: Annotated[str, msgspec.Meta(min_length=12)]
    age: Annotated[int, msgspec.Meta(ge=0, le=150)] | None = None

class UserRead(msgspec.Struct, kw_only=True, omit_defaults=True):
    id: UUID
    email: str
    display_name: str
    created_at: datetime
```

### Two behavioral rules that differ from Pydantic

- **The constructor does not validate.** `User(email=123)` builds a wrong instance happily — mypy/basedpyright is expected to catch that *statically* (the compensation for no runtime constructor validation). Validation runs only at the **decode boundary** (`msgspec.json.decode(data, type=User)`), which is exactly where untrusted input enters. Litestar wires that boundary for you. Constructing a struct directly does **not** run `Meta` constraints — for declarative "always validated" objects, decode through Litestar or `msgspec.convert`.
- **Validation is strict, not coercive.** `{"age": "30"}` decoded into `age: int` raises `msgspec.ValidationError` — msgspec will not silently coerce `"30"` → `30`. This is a feature; do not "fix" it by loosening types. If you truly need coercion, pass `strict=False` on a decoder deliberately.

### Struct config flags (set as class kwargs)

| Flag | Effect |
|---|---|
| `kw_only=True` | Keyword-only init; strongly recommended so field-order changes aren't breaking. **Does not inherit** — set on each struct. |
| `frozen=True` | Pseudo-immutable + hashable; use for config/value objects. Update with `msgspec.structs.replace(obj, field=...)`, never mutate. |
| `forbid_unknown_fields=True` | Reject unexpected keys on decode; use for strict inbound request contracts. |
| `omit_defaults=True` | Drop default-valued fields on encode; smaller payloads, faster. |
| `rename="camel"` | Emit/accept `camelCase` on the wire, keep `snake_case` in Python. Also `"kebab"`, `"pascal"`, `"upper"`, `"lower"`, or a mapping. |
| `tag=` / `tag_field=` | Discriminator for tagged unions (below). |

Constraints live in `Annotated[..., msgspec.Meta(...)]` (not Pydantic's `Field`): `gt/ge/lt/le`, `multiple_of`, `min_length/max_length`, `pattern` (unanchored `re.search`), `tz` for datetimes.

### Tagged unions (discriminated unions)

Set an explicit `tag=` on **every** member; msgspec adds a discriminator field (default `"type"`) and dispatches on decode. Omitting tags forces msgspec to try each variant in order — slow and ambiguous when fields overlap.

```python
class Cat(msgspec.Struct, tag="cat"):
    meows: int

class Dog(msgspec.Struct, tag="dog"):
    barks: int

Animal = Cat | Dog
msgspec.json.decode(b'{"type":"cat","meows":3}', type=Animal)  # -> Cat(meows=3)
```

### Encoding/decoding — reuse `Encoder`/`Decoder` on hot paths

```python
import msgspec

# One-shot
payload = msgspec.json.encode(user)                 # -> bytes
user = msgspec.json.decode(payload, type=UserRead)  # validates

# Reused (faster): build once, call many
_decoder = msgspec.json.Decoder(list[UserRead])
_encoder = msgspec.json.Encoder()
users = _decoder.decode(raw_bytes)

# MessagePack for internal/binary channels
blob = msgspec.msgpack.encode(user)
```

### Other tools worth knowing

- `msgspec.field(default_factory=list)` / `field(name="userName")` — per-field defaults/renames.
- `msgspec.UNSET` / `UnsetType` — distinguish "field absent" from "explicit null" for PATCH semantics; unset fields are omitted on encode.
- `msgspec.convert(obj, Type, from_attributes=True)` — coerce ORM rows / attribute objects into structs without a JSON round-trip (the right primitive for "SQLAlchemy row → response struct").
- `msgspec.Raw` — defer decoding of a field (hold raw bytes, decode later).
- `msgspec.structs.replace/asdict/astuple` — immutable update and conversion helpers.
- `enc_hook` / `dec_hook` — support custom types.

Do **not** reach for `.model_dump()`, `.dict()`, `BaseModel`, `Field(...)`, `@validator`, or `ConfigDict` — those are Pydantic and do not exist here.

```python
from msgspec import UNSET, UnsetType

class UserPatch(msgspec.Struct, kw_only=True):
    display_name: str | UnsetType = UNSET
    email: str | UnsetType = UNSET
```

### Decision table — which data type?

| Use | When |
|---|---|
| `msgspec.Struct` | Default for all request/response models and internal data |
| `@dataclass(slots=True, frozen=True, kw_only=True)` | Interop with stdlib/other libs; still works with Litestar DTOs; immutable value objects |
| `TypedDict` | Loosely-structured dict-shaped data you don't want a class for |
| `pydantic.BaseModel` | **Only** when integrating an existing Pydantic codebase |
| Plain `dict[...]` return | Trivial handlers; loses schema richness |

If you do use dataclasses, `field(default_factory=list)` is mandatory for mutable defaults — a bare `tags: list[str] = []` is the classic shared-mutable-default bug (dataclasses raise `ValueError` if you try).

---

## Application structure & routing (Litestar 2.x)

A Litestar app is a collection of route handlers, plugins, and layered config. Unlike FastAPI (a thin layer over Starlette), Litestar has no Starlette dependency and uses msgspec as its native data layer. Compose the app from module-level handlers and `Controller` classes registered on a single `Litestar` object; prefer an **application-factory** so tests and the CLI build fresh instances.

```python
# app/server.py
from litestar import Litestar
from litestar_granian import GranianPlugin

from app.controllers.users import UserController
from app.lifespan import db_lifespan
from app.config import openapi_config

def create_app() -> Litestar:
    return Litestar(
        route_handlers=[UserController],
        plugins=[GranianPlugin()],
        lifespan=[db_lifespan],
        openapi_config=openapi_config,
    )

app = create_app()
```

Controllers group related routes under a common path and share dependencies, guards, and DTOs:

```python
# app/controllers/users.py
from uuid import UUID
from litestar import Controller, get, post, patch, delete

from app.models import User, UserCreate, UserUpdate
from app.services import UserService

class UserController(Controller):
    path = "/users"
    tags = ["users"]

    @get()
    async def list_users(self, users: UserService) -> list[User]:
        return await users.list()

    @get("/{user_id:uuid}")
    async def get_user(self, user_id: UUID, users: UserService) -> User:
        return await users.get(user_id)

    @post()
    async def create_user(self, data: UserCreate, users: UserService) -> User:
        return await users.create(data)

    @patch("/{user_id:uuid}")
    async def update_user(self, user_id: UUID, data: UserUpdate, users: UserService) -> User:
        return await users.update(user_id, data)

    @delete("/{user_id:uuid}")
    async def delete_user(self, user_id: UUID, users: UserService) -> None:
        await users.delete(user_id)
```

Details agents get wrong:

- **Path parameters are typed inline with converters:** `{user_id:uuid}`, `{item_id:int}`, `{name:str}`, `{date:date}`, `{price:float}`, `{path_val:path}`. The converter drives both parsing and the OpenAPI schema, and must match the handler parameter type. This is **not** FastAPI's `{user_id}` + separate annotation.
- **Request bodies bind to the reserved `data` parameter** — you do not add `Body(...)`.
- `@delete` handlers returning nothing default to `204 No Content`.
- **The framework enforces typing** — an untyped return value raises at startup. Annotations are how the framework parses, validates, and documents your code.
- **`Router`** groups controllers/handlers under a path prefix, and routers nest. Guards/dependencies declared on a router apply to everything beneath it.

```python
from litestar import Router
api_v1 = Router(path="/api/v1", route_handlers=[UserController, OrderController])
app = Litestar(route_handlers=[api_v1], plugins=[GranianPlugin()])
```

Litestar is **layered**: `dependencies`, `guards`, `middleware`, `before_request`, `after_response`, `exception_handlers`, and more can be set at app, router, controller, or handler level. The most specific layer wins — **except guards, which accumulate**.

---

## DTOs: reshaping models at the edge

DTOs transform between wire format and your domain objects — excluding secret fields, renaming, partial updates — without hand-written conversion code. `dto=` governs the **inbound** parse; `return_dto=` governs the **outbound** serialize. Litestar ships `MsgspecDTO`, `DataclassDTO`, `PydanticDTO`, and — most important here — `SQLAlchemyDTO` (from `advanced_alchemy.extensions.litestar`, re-exported at `litestar.plugins.sqlalchemy`), which lets you return ORM models directly from handlers.

```python
from datetime import datetime
from typing import Annotated
from sqlalchemy.orm import Mapped, mapped_column
from litestar import post
from litestar.dto import DTOConfig, DTOData, dto_field
from litestar.plugins.sqlalchemy import SQLAlchemyDTO
from advanced_alchemy.base import UUIDAuditBase

class User(UUIDAuditBase):
    __tablename__ = "user_account"
    name: Mapped[str]
    password: Mapped[str] = mapped_column(info=dto_field("private"))    # never serialized
    created_at: Mapped[datetime] = mapped_column(info=dto_field("read-only"))

config = DTOConfig(exclude={"password"}, rename_fields={"name": "userName"}, max_nested_depth=1)
UserWriteDTO = SQLAlchemyDTO[Annotated[User, config]]

@post("/users", dto=UserWriteDTO, return_dto=SQLAlchemyDTO[User])
async def create_user(data: User) -> User:
    return await save(data)
```

- `DTOConfig` supports `include`/`exclude` (mutually exclusive; take dot-paths like `"address.street"`), `rename_fields`, `rename_strategy` (`"camel"` etc.), `max_nested_depth`, and `partial=True` (all fields optional — for PATCH).
- Mark columns `dto_field("private")` (never read/write) or `dto_field("read-only")` (serialized out, ignored on input) directly on the model.
- For **partial updates**, type the handler `data` param as `DTOData[SomeDTO]` and call `.create_instance()` / `.update_instance(obj)` / `.as_builtins()` — this gives you only the fields the client actually sent.
- **If a DTO isn't adding value** (you're just returning a struct/model as-is), **don't use one** — return it directly and let msgspec encode. DTOs earn their keep when input and output shapes diverge.

---

## Dependency injection

DI is a named mapping of `Provide(callable)` entries, resolved by matching the dependency **name** to a handler parameter name — pytest-inspired, not FastAPI's `Depends()`.

```python
from litestar import Litestar, get
from litestar.di import Provide
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession

async def provide_db_session(state) -> AsyncGenerator[AsyncSession, None]:
    async with state.sessionmaker() as session:
        yield session  # teardown runs after the response

async def provide_user_service(db_session: AsyncSession) -> UserService:
    return UserService(session=db_session)   # dependencies nest by name

@get("/users/{user_id:uuid}")
async def get_user(user_id: UUID, user_service: UserService) -> UserRead:
    return await user_service.get(user_id)

app = Litestar(
    route_handlers=[get_user],
    dependencies={
        "db_session": Provide(provide_db_session),
        "user_service": Provide(provide_user_service),
        "settings": Provide(get_settings, use_cache=True),
    },
)
```

- **Scopes are layered:** declare `dependencies={}` on app, `Router`, `Controller`, or handler. A dependency is visible only at and below where it's declared; lower layers **override** higher ones by name (unlike guards, which accumulate).
- **Dependencies nest** — a provider can declare its own dependencies by name.
- **Generator dependencies** provide setup/teardown — `yield` the resource, clean up after. This is the idiomatic DB-session/connection pattern.
- `Provide(fn, use_cache=True)` memoizes the return across the request/connection (no kwargs-aware LRU — use it for expensive, argument-independent singletons like config or a shared client).
- `sync_to_thread=` applies to sync providers (see [Concurrency](#concurrency-sync-vs-async-and-where-work-runs)). Passing the callable directly (`dependencies={"x": some_callable}`) is allowed when you don't need `Provide`'s options.

---

## SQLAlchemy 2.0 async + asyncpg

Use the 2.0 style exclusively: `select()` statements executed through an `AsyncSession`. The legacy `session.query(...)` API is 1.x and must not be used.

### Engine & session

One engine per process; short-lived sessions per unit of work. **`expire_on_commit=False` is effectively mandatory under async** — otherwise attributes expire after commit and re-accessing them triggers a lazy load that raises.

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = create_async_engine(
    "postgresql+asyncpg://app:pwd@localhost:5432/app",
    pool_size=10, max_overflow=20, pool_pre_ping=True, pool_recycle=1800,
)
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
```

The `postgresql+asyncpg://` URL selects asyncpg, which MagicStack benchmarks as, on average, "5x faster than psycopg3" — the correct default. psycopg3 async (`postgresql+psycopg://`) is the adjacent alternative but not the default here. Behind **pgbouncer in transaction/statement pooling mode**, or on serverless, disable SQLAlchemy's pool with `poolclass=NullPool` and disable asyncpg's prepared-statement cache to avoid cross-connection statement collisions.

### Declarative models

`Mapped[...]` + `mapped_column()`; relationships typed `Mapped[list["Child"]]` / `Mapped["Parent"]`. Include `AsyncAttrs` on the base as an escape hatch for awaited lazy loads. `Mapped[str]` reads as `str` on an instance, so basedpyright needs no casts for attribute access.

```python
from datetime import datetime
from sqlalchemy import ForeignKey, func, String
from sqlalchemy.ext.asyncio import AsyncAttrs
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(AsyncAttrs, DeclarativeBase):
    pass

class Author(Base):
    __tablename__ = "author"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    books: Mapped[list["Book"]] = relationship(back_populates="author")

class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))
    author: Mapped[Author] = relationship(back_populates="books")
```

### Querying

```python
from sqlalchemy import select
from sqlalchemy.orm import selectinload

async with async_session() as session:
    stmt = select(Author).where(Author.name == "Ada").options(selectinload(Author.books))
    author = (await session.scalars(stmt)).one_or_none()
```

Use `.scalar_one()` (exactly one, else raises), `.scalar_one_or_none()`, or `session.scalars(...).all()`.

### The async lazy-loading rule (the #1 gotcha)

Accessing an unloaded relationship or expired attribute outside an awaitable context raises `MissingGreenlet`. You must eager-load.

| Strategy | Emits | Use when |
|---|---|---|
| `selectinload` | Second `IN` query per relationship | Collections (one-to-many, many-to-many); avoids row multiplication |
| `joinedload` | Single `LEFT OUTER JOIN` | Many-to-one / one-to-one scalar relationships |
| `subqueryload` | Correlated subquery | Legacy; prefer `selectinload` |
| `lazy="raise"` | Raises on access | Default guard to catch accidental N+1 at dev time |

Escape hatch for a single lazy attribute: `await obj.awaitable_attrs.books` (requires `AsyncAttrs`). To run sync ORM code inside async: `await session.run_sync(fn)`.

### Transactions, bulk/upsert, PostgreSQL types

```python
# Transaction: automatic commit/rollback boundary
async with async_session() as session, session.begin():
    session.add(Author(name="Grace"))

# Bulk insert + upsert with RETURNING
from sqlalchemy.dialects.postgresql import insert
stmt = (
    insert(Author).values([{"name": "A"}, {"name": "B"}])
    .on_conflict_do_update(index_elements=["name"], set_={"name": insert.excluded.name})
    .returning(Author.id)
)
ids = (await session.scalars(stmt)).all()

# PostgreSQL-specific column types
from sqlalchemy.dialects.postgresql import JSONB, ARRAY
from sqlalchemy import Integer, String

class Event(Base):
    __tablename__ = "event"
    id: Mapped[int] = mapped_column(primary_key=True)
    payload: Mapped[dict] = mapped_column(JSONB)
    tags: Mapped[list[str]] = mapped_column(ARRAY(String))
    scores: Mapped[list[int]] = mapped_column(ARRAY(Integer))
```

---

## Advanced Alchemy: repositories, services & the plugin

Advanced Alchemy is the Litestar team's official SQLAlchemy companion and the idiomatic persistence layer here — audit-ready base classes, generic async repositories/services, and the `SQLAlchemyPlugin` that wires session DI and transaction handling into Litestar. Install via `litestar[sqlalchemy]` or `advanced-alchemy`.

**Base classes** (by PK strategy): `UUIDBase`/`UUIDAuditBase`, `UUIDv7Base`/`UUIDv7AuditBase`, `BigIntBase`/`BigIntAuditBase`, `NanoIDBase`. Audit variants add `created_at`/`updated_at`. Default to `UUIDAuditBase`. **Model modules must not use `from __future__ import annotations`** (Mapped introspection).

```python
from datetime import date
from uuid import UUID
from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship
from advanced_alchemy.base import UUIDAuditBase

class Author(UUIDAuditBase):
    __tablename__ = "author"
    name: Mapped[str]
    dob: Mapped[date | None]
    books: Mapped[list["Book"]] = relationship(back_populates="author", lazy="selectin")

class Book(UUIDAuditBase):
    __tablename__ = "book"
    title: Mapped[str]
    author_id: Mapped[UUID] = mapped_column(ForeignKey("author.id"))
    author: Mapped[Author] = relationship(back_populates="books", lazy="joined", innerjoin=True)
```

**Repository** — generic CRUD, pagination, filtering, dialect-aware bulk ops with auto-detected RETURNING: `get`, `get_one`, `get_one_or_none`, `list`, `list_and_count`, `add`, `add_many`, `update`, `upsert`, `delete`, `delete_many`.

```python
from advanced_alchemy.repository import SQLAlchemyAsyncRepository
from advanced_alchemy.service import SQLAlchemyAsyncRepositoryService

class AuthorRepository(SQLAlchemyAsyncRepository[Author]):
    model_type = Author

class AuthorService(SQLAlchemyAsyncRepositoryService[Author]):
    repository_type = AuthorRepository
```

**Plugin wiring** provides the `db_session` dependency and configures the engine:

```python
from litestar import Litestar
from advanced_alchemy.extensions.litestar import (
    SQLAlchemyPlugin, SQLAlchemyAsyncConfig, AsyncSessionConfig,
)

alchemy = SQLAlchemyPlugin(
    config=SQLAlchemyAsyncConfig(
        connection_string="postgresql+asyncpg://app:pwd@localhost/app",
        session_config=AsyncSessionConfig(expire_on_commit=False),
        create_all=False,  # use Alembic for schema in production, not create_all
    )
)
app = Litestar(route_handlers=[...], plugins=[alchemy])
```

Then inject `db_session: AsyncSession` (or a provided service) into handlers. Advanced Alchemy also ships a UTC `DateTimeUTC` type, a database-agnostic JSON type (JSONB on PostgreSQL), `EncryptedString`, and a `FileObject` type with fsspec/obstore backends.

---

## Concurrency: sync vs async, and where work runs

Choose the model by workload: **async** for I/O-bound concurrency (the default here), **threads** for blocking I/O, **subinterpreters/processes** for CPU-bound isolation.

### The correctness rules

- **`async def` handlers/providers must never call blocking I/O.** Granian runs each worker on an asyncio event loop; a blocking DB/HTTP/file call starves it and stalls the whole worker. Use async drivers (`asyncpg`, async SQLAlchemy, `httpx.AsyncClient`).
- **Synchronous handlers/providers require an explicit `sync_to_thread` decision.** A blocking sync function must be `sync_to_thread=True` (Litestar runs it in a thread pool). A genuinely non-blocking sync function (pure CPU, fast, no I/O) should be `sync_to_thread=False`. Omitting it for a sync callable raises a warning — deliberately; make the choice explicit.

```python
from litestar import get

@get("/compute", sync_to_thread=False)   # fast, CPU-only, non-blocking
def compute() -> int:
    return sum(range(1000))

@get("/report", sync_to_thread=True)      # blocking library call -> thread pool
def report() -> bytes:
    return legacy_blocking_render()
```

### asyncio with structured concurrency

Use `asyncio.TaskGroup`, not bare `gather`. A TaskGroup awaits all tasks on block exit and, if any fails, cancels the siblings and raises an `ExceptionGroup`.

```python
import asyncio

async def fetch_all(client, urls: list[str]) -> list[bytes]:
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fetch(client, u)) for u in urls]
    return [t.result() for t in tasks]     # reached only if all succeeded

async def with_deadline() -> None:
    try:
        async with asyncio.timeout(5.0):    # cancels on timeout
            await slow_operation()
    except TimeoutError:
        log("timed out")
```

Handle grouped failures with `except*`:

```python
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(worker_a())
        tg.create_task(worker_b())
except* ValueError as eg:
    for exc in eg.exceptions:
        logger.warning("validation failure", error=str(exc))
except* ConnectionError as eg:
    log(f"{len(eg.exceptions)} connection failures")
```

Never `await` sequentially when you meant concurrency, never swallow `CancelledError`, and hold a reference to fire-and-forget tasks (the loop keeps only weak references).

### CPU-bound offload

For CPU-bound work, offload rather than block the loop: `InterpreterPoolExecutor` (PEP 734 subinterpreters — isolated state, "isolation of processes, efficiency of threads") or `ProcessPoolExecutor` (highest isolation, IPC cost). Free-threading could change this, but it's experimental — scaling here is by Granian **workers** (processes), not threads.

| Workload | Use |
|---|---|
| Many network/disk I/O ops | `asyncio` + `TaskGroup` (the default path) |
| Blocking I/O in a sync handler | `sync_to_thread=True` (Litestar thread pool) |
| CPU-bound, want isolation | `InterpreterPoolExecutor` |
| CPU-bound, heavy/legacy libs | `ProcessPoolExecutor` |

---

## Serving with Granian via litestar-granian

Granian is a Rust HTTP server ("built on top of the Hyper crate") running WSGI, ASGI, and its own RSGI protocol, with native HTTP/1.1 + HTTP/2 — the modern replacement for the uvicorn/gunicorn combination. It's a single binary with native HTTP/2, lower memory, and higher throughput, avoiding "the usual Gunicorn + uvicorn + http-tools dependency composition."

Add `GranianPlugin()` and the standard `litestar run` command launches Granian instead of Uvicorn — the plugin owns the server lifecycle (lifespan, signal handling, dev-reload). **Do not** run the bare `granian` CLI against a Litestar app when using the plugin, and never hand-roll a `uvicorn`/`gunicorn` invocation.

```python
from litestar import Litestar
from litestar_granian import GranianPlugin
app = Litestar(route_handlers=[...], plugins=[GranianPlugin()])
```

```bash
litestar run --reload                                   # dev, autoreload on
litestar run --host 0.0.0.0 --port 8000 --workers 8     # production
```

`GranianPlugin()` takes no config object; tuning is driven through `litestar` CLI flags and Granian's `GRANIAN_*` env vars. Key options:

| Option | Values / default | Guidance |
|---|---|---|
| `--interface` | `asgi\|asginl\|rsgi\|wsgi` (Granian default `rsgi`) | **When configuring Granian manually, use `asgi` — it is always correct for a Litestar app.** The plugin path manages this for you. Litestar has RSGI support (which avoids per-request ASGI scope-dict allocations); only pin `rsgi` once you've confirmed it works end-to-end for your Litestar version. |
| `--http` | `auto\|1\|2` (default `auto`) | Leave `auto` unless you must pin. Pure HTTP/2 breaks HTTP/1.1 clients. |
| `--workers` | int ≥1 (default 1) | Match CPU-core count; in k8s/Docker prefer **1 worker per container** and scale replicas. |
| `--runtime-mode` | `auto\|mt\|st` (default `auto`) | `st` = N single-threaded Rust runtimes; `mt` = one multi-threaded. Leave `auto` unless benchmarking. |
| `--runtime-threads` | int ≥1 (default 1) | Increase for many concurrent websockets or heavy HTTP/2. |
| `--loop` | `auto\|asyncio\|rloop\|uvloop\|winloop` (default `auto`) | `uvloop` is the common fast choice on Linux/macOS. |
| `--ssl-certificate` / `--ssl-keyfile` | file paths | Terminate TLS at Granian or a load balancer; keyfile is PKCS#8 only. |

Do **not** size Granian with Gunicorn/Uvicorn rules of thumb — its Rust runtime + backpressure architecture differs. Granian applies backpressure at the per-worker accept loop; behind a reverse proxy with many keep-alive connections, ensure backpressure exceeds expected concurrency. As DeployHQ's 2026 guide notes, "Granian's Rust-based I/O shows the largest gains in connection-heavy scenarios with minimal application logic. Once your app does real work — database queries, template rendering, JSON serialization — the I/O layer stops being the bottleneck." The cardinal rule stands: **never do blocking I/O inside an `async def` handler.**

---

## Auth: JWT backends and guards

Authentication (who you are) uses a JWT backend; authorization (what you may do) uses guards. Keep them separate.

```python
import secrets
from dataclasses import dataclass
from litestar import Litestar, Request, get, post, Response
from litestar.connection import ASGIConnection
from litestar.datastructures import State
from litestar.security.jwt import JWTAuth, Token

@dataclass
class User:
    id: str
    is_admin: bool = False

async def retrieve_user_handler(token: Token, connection: ASGIConnection) -> User | None:
    return await load_user(token.sub)

jwt_auth = JWTAuth[User](
    token_secret=secrets.token_hex(),      # load from environment in production
    retrieve_user_handler=retrieve_user_handler,
    exclude=["/login", "/schema"],         # public paths
)

@post("/login")
async def login(data: User) -> Response[User]:
    return jwt_auth.login(identifier=str(data.id), response_body=data)

@get("/me")
async def me(request: Request[User, Token, State]) -> User:
    return request.user   # populated by the middleware

app = Litestar(route_handlers=[login, me], on_app_init=[jwt_auth.on_app_init])
```

Register via `on_app_init=[jwt_auth.on_app_init]` — it injects the middleware and OpenAPI security scheme. `request.user`/`request.auth` are set from the token. Use `JWTCookieAuth` for HttpOnly-cookie transport, or `OAuth2PasswordBearerAuth` for the OAuth2 password flow; subclass `AbstractAuthenticationMiddleware` for lower-level needs.

**Guards** are cumulative callables `(connection, handler) -> None` that authorize and raise on failure. They **stack** across layers (app → router → controller → handler) — unlike dependencies, they combine rather than override.

```python
from litestar.connection import ASGIConnection
from litestar.handlers.base import BaseRouteHandler
from litestar.exceptions import NotAuthorizedException

def admin_guard(connection: ASGIConnection, _: BaseRouteHandler) -> None:
    if not getattr(connection.user, "is_admin", False):
        raise NotAuthorizedException()

@post("/admin/users", guards=[admin_guard])
async def create_user(data: User) -> User: ...
```

Built-in `CORSConfig`, `CSRFConfig`, `RateLimitConfig`, `CompressionConfig`, and `AllowedHostsConfig` cover the standard cross-cutting concerns and are passed to `Litestar(...)`.

---

## Middleware, lifespan, state & stores

**Lifespan** — manage engines/pools/HTTP clients with an async context manager; it enters on startup and exits on shutdown (multiple managers exit in inverse order). Store handles on `app.state`.

```python
from contextlib import asynccontextmanager
from collections.abc import AsyncGenerator
import httpx
from litestar import Litestar

@asynccontextmanager
async def lifespan(app: Litestar) -> AsyncGenerator[None, None]:
    app.state.http = httpx.AsyncClient(timeout=10.0)
    try:
        yield
    finally:
        await app.state.http.aclose()

app = Litestar(route_handlers=[...], lifespan=[lifespan])
```

Inject app state via a `state: State` parameter. Use it sparingly for connections/config, not as a mutable request cache. There are also `on_startup`/`on_shutdown` and `before_request`/`after_request`/`after_response` hooks at every layer.

**Middleware** — Litestar ships CORS, CSRF, compression, rate-limit, session, and logging middleware configured via config objects. Prefer these and guards over hand-rolled ASGI middleware; when you must, use the ASGI factory pattern or `AbstractMiddleware` and register with `DefineMiddleware` for arguments.

```python
from litestar.middleware import AbstractMiddleware
from litestar.types import Receive, Scope, Send

class TimingMiddleware(AbstractMiddleware):
    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        await self.app(scope, receive, send)
```

**Stores** (`litestar.stores`) are the async key/value abstraction backing sessions, response caching, and rate limiting — `MemoryStore`, `FileStore`, `RedisStore`. Register named stores and point features at them; cache a handler's response with `@get(..., cache=10)` (seconds) or `cache=True`.

```python
from litestar.stores.redis import RedisStore
from litestar.config.response_cache import ResponseCacheConfig

app = Litestar(
    stores={"redis": RedisStore.with_client(url="redis://localhost:6379")},
    response_cache_config=ResponseCacheConfig(store="redis"),
)
```

`RedisStore` namespaces keys under `LITESTAR` and its `delete_all` is namespace-scoped and safe — never issue a global `FLUSHALL`.

---

## Responses, errors & background tasks

Return a value and the framework encodes it with the right content type from your annotation. Reach for explicit `Response`/`Stream`/`File`/`Redirect`/`ServerSentEvent`/`Template` only when you need headers, cookies, status, or streaming.

```python
from litestar import get, Response
from litestar.background_tasks import BackgroundTask
from litestar.response import Stream, ServerSentEvent

@get("/greet")
async def greet(name: str) -> Response[dict[str, str]]:
    return Response({"hello": name}, headers={"X-Trace": "1"},
                    background=BackgroundTask(log_greeting, name))

@get("/download")
async def download() -> Stream:
    async def gen():
        yield b"chunk-1"
        yield b"chunk-2"
    return Stream(gen(), media_type="application/octet-stream")

@get("/events")
async def events() -> ServerSentEvent:
    async def publisher():
        yield "tick"
    return ServerSentEvent(publisher())
```

**Errors** — raise `HTTPException` or subclasses (`NotFoundException`, `NotAuthorizedException`, `PermissionDeniedException`, `ValidationException`). They serialize to a structured JSON body (`status_code`, `detail`, `extra`). Customize with an `exception_handlers={}` mapping (by status code or exception class) at any layer.

```python
from litestar import Request, Response
from litestar.exceptions import NotFoundException

@get("/users/{user_id:uuid}")
async def get_user(user_id: UUID, users: UserService) -> User:
    if (user := await users.find(user_id)) is None:
        raise NotFoundException(detail=f"user {user_id} not found")
    return user

def handle_domain_error(request: Request, exc: DomainError) -> Response:
    return Response({"detail": str(exc)}, status_code=422)

app = Litestar(route_handlers=[...], exception_handlers={DomainError: handle_domain_error})
```

Background tasks run after the response is sent; use `BackgroundTasks([...], run_in_task_group=True)` to run several concurrently.

---

## OpenAPI

Litestar auto-generates an OpenAPI 3.1.0 schema from handler signatures and msgspec/SQLAlchemy types. The JSON is served at `/schema/openapi.json` and UIs under `/schema/*`. Scalar is the project's preferred UI (the single default in the upcoming 3.0).

```python
from litestar import Litestar
from litestar.openapi import OpenAPIConfig
from litestar.openapi.plugins import ScalarRenderPlugin, SwaggerRenderPlugin

app = Litestar(
    route_handlers=[...],
    openapi_config=OpenAPIConfig(
        title="My API", version="1.0.0",
        render_plugins=[ScalarRenderPlugin(), SwaggerRenderPlugin()],
    ),
)
```

Available renderers include `ScalarRenderPlugin`, `SwaggerRenderPlugin`, `RedocRenderPlugin`, `RapidocRenderPlugin`, `StoplightRenderPlugin`. Refine per-parameter schema with `Parameter(...)` and request bodies with `Body(...)`.

---

## Parameters & request data

Query, header, and cookie parameters use `Parameter`; request bodies use `Body`; DI-only values use `Dependency`. A bare typed parameter matching no path parameter is treated as a query parameter.

```python
from typing import Annotated
from litestar import get, post
from litestar.params import Parameter, Body
from litestar.enums import RequestEncodingType

@get("/search")
async def search(
    q: str,
    limit: Annotated[int, Parameter(ge=1, le=100)] = 20,
    x_request_id: Annotated[str | None, Parameter(header="X-Request-ID")] = None,
) -> list[Result]: ...

@post("/upload")
async def upload(
    data: Annotated[dict, Body(media_type=RequestEncodingType.MULTI_PART)],
) -> None: ...
```

---

## Standard-library idioms

### Filesystem: `pathlib`, never `os.path`

```python
from pathlib import Path

config = Path("~/.config/app").expanduser()
config.mkdir(parents=True, exist_ok=True)
data = (config / "settings.toml").read_text(encoding="utf-8")
for py in Path("src").rglob("*.py"):
    print(py.relative_to("src"))
# Python 3.14 adds Path.copy() and Path.move().
```

### Dates and times: `zoneinfo`, never `pytz`

```python
from datetime import datetime, UTC
from zoneinfo import ZoneInfo

now = datetime.now(UTC)     # never datetime.utcnow() — deprecated, returns naive
meeting = datetime(2026, 7, 2, 14, 0, tzinfo=ZoneInfo("America/New_York"))
in_tokyo = meeting.astimezone(ZoneInfo("Asia/Tokyo"))
```

### Reading TOML: `tomllib` (read-only; use `tomli-w` to write)

```python
import tomllib
from pathlib import Path

with Path("pyproject.toml").open("rb") as f:   # must be binary mode
    config = tomllib.load(f)
```

### Enums, functools, contextlib

```python
from enum import StrEnum, auto
from functools import cache, cached_property, singledispatch
from contextlib import contextmanager, suppress, ExitStack
from pathlib import Path

class Color(StrEnum):
    RED = auto()      # "red" — members are real strings
    GREEN = auto()

@cache
def fib(n: int) -> int:
    return n if n < 2 else fib(n - 1) + fib(n - 2)

class Dataset:
    def __init__(self, rows: list[dict]) -> None:
        self.rows = rows
    @cached_property                    # computed once per instance
    def stats(self) -> dict[str, float]:
        return expensive_analysis(self.rows)

with suppress(FileNotFoundError):
    Path("maybe.txt").unlink()

with ExitStack() as stack:              # manage a dynamic number of contexts
    files = [stack.enter_context(open(p)) for p in paths]
```

---

## Error handling

Catch the narrowest exception you can, never `except:` bare, use `raise ... from e` to preserve chains, and use `try/finally` or context managers for cleanup.

```python
# Exception groups (PEP 654): raise/handle multiple at once.
def validate(data: dict) -> None:
    errors: list[Exception] = []
    if "name" not in data:
        errors.append(ValueError("missing name"))
    if "age" not in data:
        errors.append(ValueError("missing age"))
    if errors:
        raise ExceptionGroup("validation failed", errors)

try:
    validate({})
except* ValueError as eg:
    for e in eg.exceptions:
        logger.warning("invalid", error=str(e))

# Add context to a propagating exception without catching it.
try:
    parse(raw)
except ValueError as e:
    e.add_note(f"while parsing {source!r}")
    raise
```

---

## Logging: structlog

Use Litestar's `StructlogPlugin`, which wires request-scoped structured logging and gives every handler a `request.logger`. Configure JSON rendering in production and console rendering in dev; bind request context via contextvars (async-safe, flow through the event loop).

```python
from litestar import Litestar, Request, get
from litestar.plugins.structlog import StructlogPlugin

@get("/")
async def index(request: Request) -> None:
    request.logger.info("handled", path="/", user="anon")

app = Litestar(route_handlers=[index], plugins=[StructlogPlugin()])
```

For control over the processor chain, pass a `StructlogConfig` wrapping a `StructLoggingConfig` (`processors`, `wrapper_class`, `logger_factory`, `cache_logger_on_first_use`, `log_exceptions`). A standalone JSON config uses the stdlib bridge so third-party logs share the format:

```python
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.JSONRenderer(),   # ConsoleRenderer() in dev
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)
```

Bind per-request context with `structlog.contextvars.bind_contextvars(request_id=...)`. Prefer `WriteLogger`/stdlib output over `PrintLogger` under Granian so lines aren't interleaved. Configure logging once at the entry point — never in library modules. Log structured key-values, not `%`-style messages; the one place lazy `%`-args are correct is a *stdlib* logging call (`logger.info("x=%s", x)`), where the message is only formatted if the level is enabled — never an f-string there. Never use `print()` for diagnostics in production.

---

## httpx: outbound HTTP

Use `httpx.AsyncClient` as a **long-lived object** created at startup and closed at shutdown (see [Lifespan](#middleware-lifespan-state--stores)) — never instantiate one per request, which throws away connection pooling and TLS reuse. `requests` is sync/legacy and must not be used here.

```python
import httpx

client = httpx.AsyncClient(
    base_url="https://api.example.com",
    timeout=httpx.Timeout(connect=3.0, read=5.0, write=5.0, pool=3.0),
    limits=httpx.Limits(max_connections=100, max_keepalive_connections=20, keepalive_expiry=30.0),
    http2=True,
)
resp = await client.get("/users")
resp.raise_for_status()
data = resp.json()
```

Stream large responses with `async with client.stream("GET", url) as resp: async for chunk in resp.aiter_bytes(): ...`. Retries go through a custom `AsyncHTTPTransport(retries=...)`. For in-process testing against your Litestar app without a socket, use `httpx.ASGITransport(app=app)`.

---

## Testing

Litestar ships httpx-based clients: `TestClient` (sync), `AsyncTestClient` (async, async-native in v3), and `create_test_client` / `create_async_test_client` helpers. Build the app **fresh per test** via the application factory so tests stay isolated; Litestar deliberately has no global `dependency_overrides` — override by passing `dependencies={}` to the client.

```python
from litestar.status_codes import HTTP_200_OK
from litestar.testing import create_test_client
from app.controllers.users import UserController

def test_list_users() -> None:
    with create_test_client(route_handlers=[UserController]) as client:
        resp = client.get("/users")
        assert resp.status_code == HTTP_200_OK
```

```python
import pytest
from litestar.testing import AsyncTestClient

@pytest.mark.asyncio
async def test_users() -> None:
    async with AsyncTestClient(app=app) as client:
        resp = await client.get("/api/v1/users")
        assert resp.status_code == 200
```

Use `websocket_connect` for WebSocket handlers, and `tmp_path` (a `pathlib.Path`) / `monkeypatch` (auto-reverting) fixtures. Prefer `parametrize` over loops so each case reports separately. Add `hypothesis` property tests where a function has an invariant:

```python
from hypothesis import given, strategies as st

@given(st.text())
def test_roundtrip(s: str):
    assert decode(encode(s)) == s
```

---

## Alembic migrations (async)

Initialize with the async template so `env.py` uses an async engine (Alembic 1.16+ also offers a `pyproject` template). Set naming conventions on your `MetaData` so autogenerated constraints have stable, reversible names.

```bash
alembic init -t async migrations
```

```python
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)
```

In `migrations/env.py`, import your models so their metadata is populated, set `target_metadata = Base.metadata`, and read the async URL (the async template already runs migrations through `connection.run_sync(do_run_migrations)`). Typical loop:

```bash
alembic revision --autogenerate -m "add author table"
alembic upgrade head
```

Always review autogenerated scripts — Alembic detects table/column adds/removes and nullability but not every change (some type changes). Use Alembic for production schema; `create_all=False`. Advanced Alchemy ships its own Alembic CLI integration that plays with its base models if you prefer that path.

---

## Project layout & packaging

Use the `src/` layout with `pyproject.toml` as the single source of truth. The layout prevents accidentally importing your package from the working directory instead of the installed copy. Ship `py.typed` so downstream checkers use your annotations.

```
myapp/
├── pyproject.toml
├── uv.lock
├── .python-version
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── py.typed
│       ├── server.py
│       ├── models.py
│       ├── controllers/
│       └── services/
├── migrations/
└── tests/
```

```toml
[project]
name = "myapp"
version = "0.1.0"
requires-python = ">=3.14"
dependencies = [
    "litestar>=2.24",
    "granian>=2.7",
    "litestar-granian>=0.15",
    "msgspec>=0.21",
    "sqlalchemy[asyncio]>=2.0.51",
    "asyncpg>=0.31",
    "advanced-alchemy>=1.11",
    "alembic>=1.18",
    "structlog>=26",
    "httpx>=0.28",
]

[dependency-groups]
dev = [
    "ruff>=0.15",
    "basedpyright>=1.39",
    "pytest>=9",
    "pytest-asyncio>=1",
    "hypothesis>=6",
]

[project.scripts]
myapp = "myapp.server:app"

[build-system]
requires = ["uv_build>=0.11,<0.12"]
build-backend = "uv_build"
```

`uv_build` is the idiomatic default for a pure-Python `uv` project; use `hatchling` when you need build hooks/plugins or VCS-tag-driven versioning, `maturin`/`scikit-build-core` for native extensions. Use `[dependency-groups]` (PEP 735) for dev tooling rather than `[project.optional-dependencies]` (which is for real runtime extras).

---

## Tooling

### uv — project & dependency management

`uv` replaces pip, pip-tools, pipx, poetry, pdm, pyenv, virtualenv, and twine with one fast Rust binary. Do **not** hand-maintain `requirements.txt` or use `pip install` in a project.

```bash
uv init myapp
uv python pin 3.14
uv add litestar granian litestar-granian msgspec \
       "sqlalchemy[asyncio]" asyncpg advanced-alchemy alembic structlog httpx
uv add --dev ruff basedpyright pytest pytest-asyncio hypothesis
uv sync                        # make the env match the lockfile exactly
uv run litestar run --reload   # run in the project env (auto-syncs first)
uv lock --upgrade-package httpx
uvx ruff check .               # run a tool ephemerally, no project install
```

`uv.lock` is cross-platform and managed by uv — commit it, never edit it by hand. `requires-python` doubles as ruff's inferred target version.

### ruff — lint + format

`ruff` is a single Rust tool replacing black, isort, flake8, pyupgrade, and pydocstyle. `ruff format` is the black-compatible formatter; `ruff check --fix` lints and autofixes. v0.15.0 (2026-02-03) introduced the 2026 style guide, including unparenthesized `except A, B, C` (PEP 758) for target-version 3.14+.

```bash
uv run ruff check --fix .
uv run ruff format .
```

```toml
[tool.ruff]
line-length = 88
target-version = "py314"

[tool.ruff.lint]
select = ["E", "W", "F", "I", "UP", "B", "C4", "SIM", "ASYNC", "PTH", "RUF"]
ignore = ["E501"]              # let the formatter own line length

[tool.ruff.lint.isort]
known-first-party = ["myapp"]

[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]       # re-exports

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

`ASYNC` catches blocking calls in async functions (directly relevant to "don't block the event loop"), `UP` modernizes syntax, `B` catches bugbears, `SIM` simplifies, `PTH` pushes `os.path` → `pathlib`. Wire the official `ruff-pre-commit` hooks (`ruff-check --fix`, then `ruff-format`).

### basedpyright (default) / mypy (alternative)

**basedpyright** is the stricter pyright fork and the stack's type checker. Its default `typeCheckingMode` is `recommended` and it adds rules pyright lacks — `reportAny`, `reportExplicitAny`, `reportIgnoreCommentWithoutRule`, `reportUnreachable`. It ships a bundled Node runtime via the PyPI wheel (no npm install).

```toml
[tool.basedpyright]
pythonVersion = "3.14"
typeCheckingMode = "recommended"
include = ["src", "tests"]
reportMissingTypeStubs = false
```

```bash
uv run basedpyright
```

Because `recommended` enables `reportIgnoreCommentWithoutRule`, every suppression must name its rule: `# pyright: ignore[reportAny]`, not a bare `# type: ignore`. This mode pairs well with the stack: SQLAlchemy `Mapped[str]` resolves to `str` on instances, msgspec `Struct`s are fully statically checked (the compensation for no runtime constructor validation), and Litestar handler signatures are real types. For an existing codebase, `basedpyright --writebaseline` records current diagnostics so you enforce the ceiling without fixing everything at once.

**mypy** is the mature alternative — run it in `strict = true` for new projects if you prefer it as the CI gate (2.0, May 2026, adds experimental parallel checking via `--num-workers N`):

```toml
[tool.mypy]
python_version = "3.14"
strict = true
warn_unreachable = true

[[tool.mypy.overrides]]
module = ["tests.*"]
disallow_untyped_defs = false
```

Emerging checkers — Astral's `ty` (beta) and Meta's `pyrefly` (1.0.0) — are dramatically faster but not yet ready as a sole gate; keep basedpyright (or mypy/pyright) as the primary CI check.

### pytest config

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
addopts = "-ra --strict-markers --strict-config"
markers = [
    "slow: marks tests as slow",
    "integration: requires external services",
]
```

```bash
uv run ruff check --fix . && uv run ruff format .
uv run basedpyright
uv run pytest
```

---

## Anti-patterns to avoid

| Wrong (adjacent-ecosystem / legacy habit) | Right (this stack) |
|---|---|
| `class UserIn(pydantic.BaseModel)` for models | `class UserIn(msgspec.Struct, kw_only=True)` |
| `Field(gt=0)` / `@validator` / `ConfigDict` | `Annotated[int, msgspec.Meta(gt=0)]` |
| `return user.model_dump()` / `.dict()` | `return user` — let msgspec encode |
| Untagged `Union` of structs for polymorphism | `tag=`/`tag_field=` on **every** union member |
| Expecting `"30"` to coerce to `int` | msgspec is strict; fix the input or opt into `strict=False` |
| Constructing a struct and expecting `Meta` to validate | validate on decode / `msgspec.convert` |
| Mutating a `frozen=True` struct | `msgspec.structs.replace(obj, field=…)` |
| `uvicorn app:app` / `gunicorn -k uvicorn.workers…` | `litestar run` with `GranianPlugin()` |
| `--interface rsgi` before confirming support | `--interface asgi` (or let the plugin manage it) |
| `Depends(get_db)` in the signature | `dependencies={"db": Provide(get_db)}` + `db` param |
| `session.query(User).filter_by(...)` | `select(User).where(...)` + `session.scalars()` |
| Accessing `author.books` unloaded under async | `selectinload(...)` / `await author.awaitable_attrs.books` |
| `expire_on_commit=True` (default) under async | `expire_on_commit=False` |
| `httpx.AsyncClient()` per request | one long-lived client on `app.state`, closed in lifespan |
| `import requests` | `httpx.AsyncClient` |
| Blocking I/O inside `async def` | async driver/client, or sync handler with `sync_to_thread=True` |
| Sync handler with no `sync_to_thread` | `sync_to_thread=True` (blocking) or `False` (pure CPU) |
| `create_all` for production schema | Alembic migrations (`upgrade head`) |
| Global `FLUSHALL`-style cache clears | namespaced `RedisStore` + scoped `delete_all` |
| Handler/DI types behind `if TYPE_CHECKING:` | keep injected/handler types importable at runtime |
| `from __future__ import annotations` in ORM model modules | omit it (Mapped introspection); redundant elsewhere on 3.14 |
| Hand-writing model→response conversion | a DTO with `exclude`/`rename`, or `msgspec.convert(..., from_attributes=True)` |
| `from typing import List, Dict, Optional, Union` | built-in `list`/`dict`, `X \| None`, `X \| Y` |
| `TypeVar('T')` + `Generic[T]` | `def f[T](...)` / `class C[T]:` (PEP 695) |
| `import os.path; os.path.join(a, b)` | `from pathlib import Path; Path(a) / b` |
| `import pytz` / `datetime.utcnow()` | `zoneinfo.ZoneInfo` / `datetime.now(UTC)` |
| `"%s" % x` / `"{}".format(x)` | f-strings (`f"{x}"`) |
| `# type:` comment hints | inline annotation `x: list[int]` |
| Bare `asyncio.gather(...)` for related tasks | `async with asyncio.TaskGroup() as tg:` |
| `except Exception:` swallowing | catch the specific type; `raise ... from e` |
| `pip install` / `poetry add` + `requirements.txt` | `uv add` / `uv sync` + `uv.lock` |
| `black . && isort . && flake8` | `ruff format` + `ruff check --fix` |
| `mypy` / plain `pyright` as the only choice | `basedpyright` (`recommended`); mypy strict as alternative |
| bare `# type: ignore` | `# pyright: ignore[reportX]` |
| `print()` for diagnostics | structlog `request.logger` / structured events |
| f-string inside a stdlib logging call | lazy `logger.info("x=%s", x)` |
| Mutable default argument / class attribute | `x: list \| None = None` / `field(default_factory=...)` |

---

## Version & compatibility

| Feature | Introduced / stabilized |
|---|---|
| Built-in generics (`list[int]`) | Python 3.9 |
| `X \| Y` unions, `match`/`case` (PEP 634) | Python 3.10 |
| `TaskGroup`, `asyncio.timeout`, exception groups + `except*`, `StrEnum`, `tomllib`, `Self`, `add_note` | Python 3.11 |
| PEP 695 generics + `type` statement, `@override`, `Unpack` `**kwargs`, per-interpreter GIL (PEP 684) | Python 3.12 |
| `ReadOnly` TypedDict items (PEP 705) | Python 3.13 |
| Deferred annotations (PEP 649/749), `annotationlib`, t-strings (PEP 750), `except` without parens (PEP 758), `concurrent.interpreters` (PEP 734), free-threaded build supported (PEP 779, opt-in `3.14t`), experimental JIT | Python 3.14 |

