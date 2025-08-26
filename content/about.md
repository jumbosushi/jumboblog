+++
title = "About"
[menus]
  [menus.main]
    name = "about"
weight = 2
+++

Index of blog posts:
```python
print("hi")
print("hi") print("hi") print("hi") print("hi") print("hi") print("hi") print("hi") print("hi") print("hi") print("hi") print("hi") print("hi") print("hi") print("hi") print("hi") print("hi")
```

```python
#!/usr/bin/env python3
"""
Complex Python code to test syntax highlighting capabilities.
This module demonstrates various Python features for syntax highlighting testing.
"""

import asyncio
import functools
import re
from abc import ABC, abstractmethod
from collections import defaultdict, namedtuple
from contextlib import contextmanager
from dataclasses import dataclass, field
from enum import Enum, auto
from pathlib import Path
from typing import Dict, List, Optional, Union, Generic, TypeVar, Protocol
from typing_extensions import Literal

# Constants and type aliases
API_VERSION: str = "v2.1.0"
MAX_RETRIES: int = 3
DEFAULT_TIMEOUT: float = 30.0

T = TypeVar('T')
K = TypeVar('K')
V = TypeVar('V')

# Enum definition
class Status(Enum):
    PENDING = auto()
    PROCESSING = auto()
    COMPLETED = auto()
    FAILED = auto()

# Protocol definition
class Serializable(Protocol):
    def serialize(self) -> Dict[str, Union[str, int, float, bool]]:
        ...

# Dataclass with complex annotations
@dataclass(frozen=True, slots=True)
class Config:
    host: str = "localhost"
    port: int = 8080
    ssl_enabled: bool = False
    timeout: float = DEFAULT_TIMEOUT
    headers: Dict[str, str] = field(default_factory=dict)
    allowed_methods: List[Literal["GET", "POST", "PUT", "DELETE"]] = field(
        default_factory=lambda: ["GET", "POST"]
    )

# Named tuple
ConnectionInfo = namedtuple('ConnectionInfo', ['host', 'port', 'secure'])

# Abstract base class
class BaseProcessor(ABC, Generic[T]):
    """Abstract base class for data processors."""

    def __init__(self, config: Config):
        self._config = config
        self._cache: Dict[str, T] = {}
        self._stats = defaultdict(int)

    @abstractmethod
    async def process(self, data: T) -> Optional[T]:
        """Process the given data asynchronously."""
        pass

    @property
    def cache_size(self) -> int:
        return len(self._cache)

    def clear_cache(self) -> None:
        """Clear the internal cache."""
        self._cache.clear()
        self._stats.clear()

        yield pattern.format(i)
```

Checking what `inline code` looks like

This site uses the excellent [Hugo ʕ•ᴥ•ʔ Bear](https://github.com/janraasch/hugo-bearblog) theme
