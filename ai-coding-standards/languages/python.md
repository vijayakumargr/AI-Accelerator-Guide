# Python Instructions

## General Principles
- **ALWAYS follow PEP 8** style guidelines
- **ALWAYS use type hints** for function signatures
- **ALWAYS use virtual environments** for project isolation
- **ALWAYS handle exceptions explicitly** - never bare `except:`
- **NEVER use mutable default arguments** (lists, dicts)
- **NEVER use `import *`** - explicit imports only

---

# Code Style

## Naming Conventions
```python
# Variables and functions: snake_case
user_name = "John"
def calculate_total_price(items: list) -> float:
    pass

# Classes: PascalCase
class UserAccount:
    pass

# Constants: UPPER_SNAKE_CASE
MAX_RETRY_COUNT = 3
DEFAULT_TIMEOUT_SECONDS = 30

# Private: leading underscore
_internal_cache = {}
def _helper_function():
    pass

# "Protected": single underscore (convention)
class Base:
    def _protected_method(self):
        pass
```

## Type Hints
```python
# GOOD: Full type hints
from typing import Optional, List, Dict, Union
from datetime import datetime

def process_user(
    user_id: int,
    options: Optional[Dict[str, str]] = None,
    tags: List[str] | None = None,  # Python 3.10+ syntax
) -> dict[str, Any]:
    """Process user data and return result."""
    pass

# Use TypedDict for structured dicts
from typing import TypedDict

class UserData(TypedDict):
    id: int
    name: str
    email: str
    created_at: datetime
```

## Docstrings
```python
def calculate_discount(
    price: float,
    discount_percent: float,
    min_price: float = 0.0
) -> float:
    """Calculate discounted price with minimum threshold.

    Args:
        price: Original price before discount.
        discount_percent: Discount percentage (0-100).
        min_price: Minimum price after discount.

    Returns:
        The discounted price, not less than min_price.

    Raises:
        ValueError: If price is negative or discount_percent is invalid.

    Example:
        >>> calculate_discount(100.0, 20.0)
        80.0
    """
    if price < 0:
        raise ValueError("Price cannot be negative")
    if not 0 <= discount_percent <= 100:
        raise ValueError("Discount must be between 0 and 100")

    discounted = price * (1 - discount_percent / 100)
    return max(discounted, min_price)
```

---

# Error Handling

## Exception Patterns
```python
# GOOD: Specific exception handling
try:
    result = external_api.fetch_data(user_id)
except requests.Timeout:
    logger.warning("API timeout for user %s", user_id)
    return fallback_data
except requests.HTTPError as e:
    if e.response.status_code == 404:
        return None
    logger.error("API error: %s", e)
    raise
except Exception:
    logger.exception("Unexpected error fetching data")
    raise

# BAD: Bare except or catching Exception without re-raise
try:
    result = external_api.fetch_data(user_id)
except:  # Never do this
    pass
```

## Custom Exceptions
```python
# Define domain-specific exceptions
class DomainError(Exception):
    """Base exception for domain errors."""
    pass

class ValidationError(DomainError):
    """Raised when validation fails."""
    def __init__(self, field: str, message: str):
        self.field = field
        self.message = message
        super().__init__(f"{field}: {message}")

class NotFoundError(DomainError):
    """Raised when a resource is not found."""
    def __init__(self, resource: str, identifier: Any):
        self.resource = resource
        self.identifier = identifier
        super().__init__(f"{resource} not found: {identifier}")
```

---

# Project Structure

## Recommended Layout
```
project/
├── src/
│   └── package_name/
│       ├── __init__.py
│       ├── main.py
│       ├── models/
│       │   ├── __init__.py
│       │   └── user.py
│       ├── services/
│       │   ├── __init__.py
│       │   └── user_service.py
│       └── utils/
│           ├── __init__.py
│           └── helpers.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── unit/
│   └── integration/
├── pyproject.toml
├── requirements.txt
└── README.md
```

## pyproject.toml
```toml
[project]
name = "package-name"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "requests>=2.28.0",
    "pydantic>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0.0",
    "pytest-cov>=4.0.0",
    "ruff>=0.1.0",
    "mypy>=1.0.0",
]

[tool.ruff]
line-length = 88
select = ["E", "F", "I", "N", "W", "UP"]

[tool.mypy]
python_version = "3.11"
strict = true

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --cov=src"
```

---

# Testing

## Test Structure
```python
import pytest
from package_name.services.user_service import UserService
from package_name.models.user import User

class TestUserService:
    """Tests for UserService."""

    @pytest.fixture
    def user_service(self, mock_repository):
        """Create UserService with mocked dependencies."""
        return UserService(repository=mock_repository)

    def test_create_user_with_valid_data_succeeds(self, user_service):
        # Arrange
        user_data = {"name": "John", "email": "john@example.com"}

        # Act
        user = user_service.create(user_data)

        # Assert
        assert user.id is not None
        assert user.name == "John"
        assert user.email == "john@example.com"

    def test_create_user_with_duplicate_email_raises_error(self, user_service):
        # Arrange
        user_data = {"name": "John", "email": "existing@example.com"}

        # Act & Assert
        with pytest.raises(ValidationError) as exc_info:
            user_service.create(user_data)

        assert "email" in str(exc_info.value)
```

## Fixtures
```python
# conftest.py
import pytest
from unittest.mock import Mock

@pytest.fixture
def mock_repository():
    """Create a mock repository."""
    repo = Mock()
    repo.find_by_email.return_value = None
    return repo

@pytest.fixture
def sample_user():
    """Create a sample user for testing."""
    return User(id=1, name="Test User", email="test@example.com")
```

---

# Common Patterns

## Context Managers
```python
from contextlib import contextmanager
from typing import Generator

@contextmanager
def database_transaction(connection) -> Generator[None, None, None]:
    """Context manager for database transactions."""
    try:
        yield
        connection.commit()
    except Exception:
        connection.rollback()
        raise
    finally:
        connection.close()

# Usage
with database_transaction(conn):
    cursor.execute("INSERT INTO users ...")
```

## Dataclasses / Pydantic
```python
# Pydantic for validation (preferred for APIs)
from pydantic import BaseModel, EmailStr, Field

class UserCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    email: EmailStr
    age: int = Field(..., ge=0, le=150)

    class Config:
        str_strip_whitespace = True

# Dataclasses for internal data structures
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class User:
    id: int
    name: str
    email: str
    created_at: datetime = field(default_factory=datetime.utcnow)
```

---

# Security

## Input Validation
```python
# ALWAYS validate and sanitize input
from pydantic import BaseModel, validator
import re

class UserInput(BaseModel):
    username: str

    @validator("username")
    def validate_username(cls, v):
        if not re.match(r"^[a-zA-Z0-9_]+$", v):
            raise ValueError("Username must be alphanumeric")
        return v
```

## SQL Injection Prevention
```python
# GOOD: Parameterized queries
cursor.execute(
    "SELECT * FROM users WHERE id = %s AND status = %s",
    (user_id, status)
)

# GOOD: ORM usage
User.query.filter_by(id=user_id, status=status).first()

# BAD: String formatting (SQL injection vulnerable)
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")  # NEVER
```

## Secrets Handling
```python
# GOOD: Environment variables
import os

DATABASE_URL = os.environ["DATABASE_URL"]
API_KEY = os.environ.get("API_KEY")  # Optional

# GOOD: Using python-dotenv for local development
from dotenv import load_dotenv
load_dotenv()

# BAD: Hardcoded secrets
API_KEY = "sk-1234567890"  # NEVER
```

---

# Anti-Patterns to Avoid

## Common Mistakes
```python
# BAD: Mutable default argument
def add_item(item, items=[]):  # Bug: shared list
    items.append(item)
    return items

# GOOD: Use None as default
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items

# BAD: Using type() for type checking
if type(obj) == list:
    pass

# GOOD: Using isinstance
if isinstance(obj, list):
    pass

# BAD: Catching too broad
except Exception:
    pass  # Silently swallowing errors

# GOOD: Specific exceptions, re-raise if needed
except ValueError as e:
    logger.error("Validation failed: %s", e)
    raise
```
