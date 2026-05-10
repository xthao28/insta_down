# Backend — InstaDown FastAPI

**Stack:** Python 3.11 · FastAPI · yt-dlp · httpx · slowapi  
**Pattern:** Clean Architecture (4 layers)  
**Liên quan:** [TDD.md](TDD.md) · [PRD.md](PRD.md)

---

## Mục lục

1. [Clean Architecture Overview](#1-clean-architecture-overview)
2. [Directory Structure](#2-directory-structure)
3. [Layer 1 — Domain](#3-layer-1--domain)
4. [Layer 2 — Application](#4-layer-2--application)
5. [Layer 3 — Infrastructure](#5-layer-3--infrastructure)
6. [Layer 4 — Presentation](#6-layer-4--presentation)
7. [Dependency Injection](#7-dependency-injection)
8. [Error Handling](#8-error-handling)
9. [Setup & Run](#9-setup--run)
10. [Testing Strategy](#10-testing-strategy)

---

## 1. Clean Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  PRESENTATION  (FastAPI routers, schemas, middleware)        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  APPLICATION  (Use Cases, DTOs)                        │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  DOMAIN  (Entities, Value Objects, Ports)        │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                               │
│  INFRASTRUCTURE  (yt-dlp, httpx — implements Domain Ports)   │
│  (nằm ngoài nhưng phụ thuộc vào Domain, không ngược lại)     │
└─────────────────────────────────────────────────────────────┘
```

**Dependency Rule:** import chỉ đi từ ngoài vào trong.
- `Presentation` → `Application` → `Domain`
- `Infrastructure` → `Domain` (implement ports)
- `Domain` không import bất kỳ layer nào khác

**Lợi ích cụ thể cho project này:**
- Swap extractor (yt-dlp → instaloader) chỉ cần thêm class mới implement `IMediaExtractor`, không đụng use case
- Test use case không cần mock HTTP — dùng fake extractor implement interface
- Thêm endpoint mới (e.g., v2) không ảnh hưởng business logic

---

## 2. Directory Structure

```
backend/
├── app/
│   │
│   ├── domain/                          # Layer 1 — không import gì ngoài stdlib
│   │   ├── __init__.py
│   │   ├── entities/
│   │   │   ├── __init__.py
│   │   │   ├── media_info.py            # MediaInfo, MediaItem
│   │   │   └── content_type.py          # ContentType enum
│   │   ├── value_objects/
│   │   │   ├── __init__.py
│   │   │   └── instagram_url.py         # InstagramUrl (validated, normalized)
│   │   ├── ports/
│   │   │   ├── __init__.py
│   │   │   └── media_extractor.py       # IMediaExtractor ABC
│   │   └── exceptions.py                # DomainException hierarchy
│   │
│   ├── application/                     # Layer 2 — chỉ import domain
│   │   ├── __init__.py
│   │   ├── use_cases/
│   │   │   ├── __init__.py
│   │   │   └── resolve_media.py         # ResolveMediaUseCase
│   │   └── dtos/
│   │       ├── __init__.py
│   │       └── media_dto.py             # ResolveRequestDTO, ResolveResultDTO
│   │
│   ├── infrastructure/                  # Layer 3 — implement domain ports
│   │   ├── __init__.py
│   │   ├── extractors/
│   │   │   ├── __init__.py
│   │   │   ├── ytdlp_extractor.py       # IMediaExtractor via yt-dlp
│   │   │   └── og_meta_extractor.py     # IMediaExtractor via httpx + BS4
│   │   └── http/
│   │       ├── __init__.py
│   │       └── client.py                # shared httpx.AsyncClient factory
│   │
│   ├── presentation/                    # Layer 4 — FastAPI
│   │   ├── __init__.py
│   │   ├── api/
│   │   │   ├── __init__.py
│   │   │   └── v1/
│   │   │       ├── __init__.py
│   │   │       ├── router.py            # include tất cả v1 endpoints
│   │   │       └── endpoints/
│   │   │           ├── __init__.py
│   │   │           ├── media.py         # POST /media/resolve
│   │   │           └── health.py        # GET /health
│   │   ├── schemas/
│   │   │   ├── __init__.py
│   │   │   ├── requests.py              # ResolveRequest (Pydantic)
│   │   │   └── responses.py             # MediaResponse, ErrorResponse (Pydantic)
│   │   └── middleware/
│   │       ├── __init__.py
│   │       ├── cors.py                  # CORSMiddleware config
│   │       └── rate_limiter.py          # slowapi setup + limiter instance
│   │
│   ├── main.py                          # app factory, register middleware & routers
│   └── config.py                        # Pydantic BaseSettings (.env)
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                      # fixtures: fake extractor, test client
│   ├── domain/
│   │   └── test_instagram_url.py
│   ├── application/
│   │   └── test_resolve_media.py        # test use case với fake extractor
│   └── presentation/
│       └── test_media_endpoint.py       # integration test qua HTTP
│
├── .env.example
├── .env                                 # gitignored
├── requirements.txt
├── requirements-dev.txt
├── Dockerfile
└── docker-compose.yml
```

---

## 3. Layer 1 — Domain

> Không import bất kỳ package bên ngoài. Không phụ thuộc FastAPI hay yt-dlp.

### `domain/entities/content_type.py`

```python
from enum import StrEnum

class ContentType(StrEnum):
    PHOTO    = "photo"
    VIDEO    = "video"
    REEL     = "reel"
    CAROUSEL = "carousel"
```

### `domain/entities/media_info.py`

```python
from dataclasses import dataclass, field
from typing import Optional
from .content_type import ContentType

@dataclass(frozen=True)
class MediaItem:
    index:            int
    type:             str              # "photo" | "video"
    url:              str              # CDN URL trực tiếp
    thumbnail_url:    str
    width:            Optional[int]   = None
    height:           Optional[int]   = None
    duration_seconds: Optional[float] = None

@dataclass(frozen=True)
class MediaInfo:
    content_type: ContentType
    username:     str
    media_items:  list[MediaItem] = field(default_factory=list)

    def __post_init__(self):
        if not self.media_items:
            raise ValueError("MediaInfo must have at least one media item")
```

### `domain/value_objects/instagram_url.py`

Value Object: bất biến, tự validate, cung cấp normalized URL.

```python
import re
import ipaddress
import socket
from urllib.parse import urlparse

ALLOWED_HOSTS = {"instagram.com", "www.instagram.com", "instagr.am"}

# Nhận diện loại URL
_PATTERNS = {
    "post":    re.compile(r"/p/([A-Za-z0-9_-]+)"),
    "reel":    re.compile(r"/reel(?:s)?/([A-Za-z0-9_-]+)"),
    "tv":      re.compile(r"/tv/([A-Za-z0-9_-]+)"),
    "stories": re.compile(r"/stories/"),
}

_PRIVATE_NETWORKS = [
    ipaddress.ip_network(cidr) for cidr in (
        "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16",
        "127.0.0.0/8", "169.254.0.0/16", "::1/128",
    )
]

class InstagramUrl:
    def __init__(self, raw: str):
        raw = raw.strip()
        if len(raw) > 2048:
            raise InvalidUrlError("URL too long")

        if not raw.startswith(("http://", "https://")):
            raw = "https://" + raw

        parsed = urlparse(raw)

        if parsed.hostname not in ALLOWED_HOSTS:
            raise InvalidUrlError("Not an Instagram URL")

        self._reject_ssrf(parsed.hostname)

        # Normalize: giữ lại scheme + host + path, bỏ query/fragment
        self._value = f"https://www.instagram.com{parsed.path.rstrip('/')}/"
        self._url_type = self._detect_type(self._value)

    @property
    def value(self) -> str:
        return self._value

    @property
    def url_type(self) -> str:
        return self._url_type

    @property
    def is_stories(self) -> bool:
        return self._url_type == "stories"

    def _detect_type(self, url: str) -> str:
        for name, pattern in _PATTERNS.items():
            if pattern.search(url):
                return name
        return "unknown"

    @staticmethod
    def _reject_ssrf(hostname: str) -> None:
        try:
            ip = ipaddress.ip_address(socket.gethostbyname(hostname))
            for net in _PRIVATE_NETWORKS:
                if ip in net:
                    raise InvalidUrlError(f"Rejected private IP: {ip}")
        except socket.gaierror:
            pass  # DNS fail → Instagram sẽ 404, không phải security issue

# Nằm trong domain/exceptions.py nhưng dùng ở đây cho gọn
class InvalidUrlError(ValueError):
    pass
```

### `domain/ports/media_extractor.py`

Interface mà Infrastructure phải implement.

```python
from abc import ABC, abstractmethod
from app.domain.entities.media_info import MediaInfo
from app.domain.value_objects.instagram_url import InstagramUrl

class IMediaExtractor(ABC):
    @abstractmethod
    async def extract(self, url: InstagramUrl) -> MediaInfo:
        """
        Trích xuất thông tin media từ URL Instagram.
        Raise DomainException nếu thất bại.
        """
        ...

    @property
    @abstractmethod
    def name(self) -> str:
        """Tên extractor để logging."""
        ...
```

### `domain/exceptions.py`

```python
class DomainException(Exception):
    """Base class cho tất cả domain exceptions."""
    def __init__(self, code: str, detail: str = ""):
        self.code   = code
        self.detail = detail
        super().__init__(code)

class InvalidUrlError(DomainException):
    def __init__(self, detail: str = ""):
        super().__init__("INVALID_URL", detail)

class ContentPrivateError(DomainException):
    def __init__(self):
        super().__init__("CONTENT_PRIVATE")

class ContentNotFoundError(DomainException):
    def __init__(self):
        super().__init__("CONTENT_NOT_FOUND")

class ContentNotSupportedError(DomainException):
    def __init__(self, detail: str = ""):
        super().__init__("CONTENT_NOT_SUPPORTED", detail)

class ExtractionFailedError(DomainException):
    def __init__(self, detail: str = ""):
        super().__init__("EXTRACTION_FAILED", detail)
```

---

## 4. Layer 2 — Application

> Chỉ import từ `domain`. Không biết yt-dlp hay FastAPI tồn tại.

### `application/dtos/media_dto.py`

DTO trung chuyển dữ liệu giữa Presentation và Use Case.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class ResolveRequestDTO:
    url: str                      # raw string từ user

@dataclass(frozen=True)
class MediaItemDTO:
    index:            int
    type:             str
    url:              str
    thumbnail_url:    str
    width:            int | None
    height:           int | None
    duration_seconds: float | None

@dataclass(frozen=True)
class ResolveResultDTO:
    content_type: str
    username:     str
    media_items:  list[MediaItemDTO]
```

### `application/use_cases/resolve_media.py`

Business logic thuần túy — không biết HTTP, không biết extractor cụ thể.

```python
import logging
from app.domain.ports.media_extractor import IMediaExtractor
from app.domain.value_objects.instagram_url import InstagramUrl
from app.domain.exceptions import (
    DomainException, ContentPrivateError,
    ContentNotFoundError, ContentNotSupportedError, ExtractionFailedError,
)
from app.application.dtos.media_dto import ResolveRequestDTO, ResolveResultDTO, MediaItemDTO

logger = logging.getLogger(__name__)

class ResolveMediaUseCase:
    """
    Nhận URL thô từ user, validate, extract media info qua danh sách
    extractors theo thứ tự ưu tiên (fallback chain).
    """

    def __init__(self, extractors: list[IMediaExtractor]):
        if not extractors:
            raise ValueError("At least one extractor is required")
        self._extractors = extractors

    async def execute(self, request: ResolveRequestDTO) -> ResolveResultDTO:
        # 1. Validate & normalize URL (Domain logic)
        url = InstagramUrl(request.url)   # raise InvalidUrlError nếu sai

        # 2. Từ chối Stories (out of scope v1.0)
        if url.is_stories:
            raise ContentNotSupportedError("Instagram Stories are not supported yet")

        # 3. Chạy extractors theo thứ tự, fallback nếu extractor thất bại
        last_error: DomainException | None = None

        for extractor in self._extractors:
            try:
                logger.debug("Trying extractor: %s", extractor.name)
                media_info = await extractor.extract(url)
                return self._to_dto(media_info)

            except (ContentPrivateError, ContentNotFoundError):
                raise  # Kết quả xác định — không retry

            except DomainException as e:
                logger.warning("Extractor %s failed: %s", extractor.name, e.detail)
                last_error = e

        raise last_error or ExtractionFailedError("All extractors failed")

    @staticmethod
    def _to_dto(media_info) -> ResolveResultDTO:
        return ResolveResultDTO(
            content_type=media_info.content_type.value,
            username=media_info.username,
            media_items=[
                MediaItemDTO(
                    index=item.index,
                    type=item.type,
                    url=item.url,
                    thumbnail_url=item.thumbnail_url,
                    width=item.width,
                    height=item.height,
                    duration_seconds=item.duration_seconds,
                )
                for item in media_info.media_items
            ],
        )
```

---

## 5. Layer 3 — Infrastructure

> Implement `IMediaExtractor`. Được phép import yt-dlp, httpx, etc.

### `infrastructure/extractors/ytdlp_extractor.py`

```python
import asyncio
import logging
import yt_dlp

from app.domain.ports.media_extractor import IMediaExtractor
from app.domain.entities.media_info import MediaInfo, MediaItem
from app.domain.entities.content_type import ContentType
from app.domain.value_objects.instagram_url import InstagramUrl
from app.domain.exceptions import (
    ContentPrivateError, ContentNotFoundError, ExtractionFailedError,
)

logger = logging.getLogger(__name__)

_YDL_OPTS = {
    "quiet":          True,
    "no_warnings":    True,
    "skip_download":  True,
    "socket_timeout": 15,
    "http_headers": {
        "User-Agent": (
            "Mozilla/5.0 (iPhone; CPU iPhone OS 16_0 like Mac OS X) "
            "AppleWebKit/605.1.15 (KHTML, like Gecko) Version/16.0 "
            "Mobile/15E148 Safari/604.1"
        ),
    },
}

class YtDlpExtractor(IMediaExtractor):

    @property
    def name(self) -> str:
        return "yt-dlp"

    async def extract(self, url: InstagramUrl) -> MediaInfo:
        loop = asyncio.get_running_loop()
        # yt-dlp là blocking → chạy trong thread pool
        return await loop.run_in_executor(None, self._extract_sync, url.value)

    def _extract_sync(self, url: str) -> MediaInfo:
        with yt_dlp.YoutubeDL(_YDL_OPTS) as ydl:
            try:
                info = ydl.extract_info(url, download=False)
            except yt_dlp.utils.DownloadError as exc:
                self._raise_domain_error(exc)

        return self._parse(info, url)

    def _parse(self, info: dict, url: str) -> MediaInfo:
        entries = info.get("entries") or [info]
        items = [self._parse_entry(i, e) for i, e in enumerate(entries)]

        content_type = self._detect_content_type(url, items, len(entries))
        return MediaInfo(
            content_type=content_type,
            username=info.get("uploader_id") or info.get("uploader", ""),
            media_items=items,
        )

    @staticmethod
    def _parse_entry(index: int, entry: dict) -> MediaItem:
        is_video = entry.get("vcodec", "none") != "none"

        if is_video:
            formats = [
                f for f in entry.get("formats", [])
                if f.get("ext") == "mp4" and f.get("vcodec", "none") != "none"
            ]
            best_url = (
                max(formats, key=lambda f: f.get("height") or 0)["url"]
                if formats else entry["url"]
            )
        else:
            best_url = entry["url"]

        thumbnails = entry.get("thumbnails") or []
        thumb = (thumbnails[-1]["url"] if thumbnails else None) or entry.get("thumbnail", "")

        return MediaItem(
            index=index,
            type="video" if is_video else "photo",
            url=best_url,
            thumbnail_url=thumb,
            width=entry.get("width"),
            height=entry.get("height"),
            duration_seconds=entry.get("duration"),
        )

    @staticmethod
    def _detect_content_type(url: str, items: list[MediaItem], count: int) -> ContentType:
        if count > 1:
            return ContentType.CAROUSEL
        if "/reel" in url:
            return ContentType.REEL
        if items[0].type == "video":
            return ContentType.VIDEO
        return ContentType.PHOTO

    @staticmethod
    def _raise_domain_error(exc: yt_dlp.utils.DownloadError) -> None:
        msg = str(exc).lower()
        if "private" in msg or "login" in msg or "authentication" in msg:
            raise ContentPrivateError()
        if "not found" in msg or "does not exist" in msg or "404" in msg:
            raise ContentNotFoundError()
        raise ExtractionFailedError(detail=str(exc)[:200])
```

### `infrastructure/extractors/og_meta_extractor.py`

```python
import httpx
import logging
from bs4 import BeautifulSoup

from app.domain.ports.media_extractor import IMediaExtractor
from app.domain.entities.media_info import MediaInfo, MediaItem
from app.domain.entities.content_type import ContentType
from app.domain.value_objects.instagram_url import InstagramUrl
from app.domain.exceptions import (
    ContentPrivateError, ContentNotFoundError, ExtractionFailedError,
)

logger = logging.getLogger(__name__)

_HEADERS = {
    # facebookexternalhit để Instagram trả OG tags đầy đủ
    "User-Agent": "facebookexternalhit/1.1 (+http://www.facebook.com/externalhit_uatext.php)",
    "Accept-Language": "en-US,en;q=0.9",
}

class OgMetaExtractor(IMediaExtractor):

    @property
    def name(self) -> str:
        return "og-meta"

    async def extract(self, url: InstagramUrl) -> MediaInfo:
        async with httpx.AsyncClient(timeout=15, follow_redirects=True) as client:
            try:
                resp = await client.get(url.value, headers=_HEADERS)
            except httpx.RequestError as exc:
                raise ExtractionFailedError(detail=str(exc))

        if resp.status_code == 404:
            raise ContentNotFoundError()
        if resp.status_code in (401, 403):
            raise ContentPrivateError()
        if resp.status_code != 200:
            raise ExtractionFailedError(detail=f"HTTP {resp.status_code}")

        return self._parse_html(resp.text, url)

    @staticmethod
    def _parse_html(html: str, url: InstagramUrl) -> MediaInfo:
        soup  = BeautifulSoup(html, "lxml")
        video = soup.find("meta", property="og:video")
        image = soup.find("meta", property="og:image")
        title = soup.find("meta", property="og:title")

        if not video and not image:
            raise ExtractionFailedError(detail="No OG media tags found")

        media_tag = video or image
        media_url = media_tag["content"]          # type: ignore[index]
        thumb_url = image["content"] if image else media_url  # type: ignore[index]
        is_video  = video is not None

        username = ""
        if title:
            raw = title.get("content", "")        # type: ignore[union-attr]
            username = raw.split(" on Instagram")[0].strip()

        content_type = (
            ContentType.REEL if "/reel" in url.value
            else ContentType.VIDEO if is_video
            else ContentType.PHOTO
        )

        return MediaInfo(
            content_type=content_type,
            username=username,
            media_items=[
                MediaItem(
                    index=0,
                    type="video" if is_video else "photo",
                    url=media_url,
                    thumbnail_url=thumb_url,
                    width=None,
                    height=None,
                    duration_seconds=None,
                )
            ],
        )
```

---

## 6. Layer 4 — Presentation

### `presentation/schemas/requests.py`

```python
from pydantic import BaseModel, field_validator
import re

class ResolveRequest(BaseModel):
    url: str

    @field_validator("url")
    @classmethod
    def validate_url(cls, v: str) -> str:
        v = v.strip()
        if len(v) > 2048:
            raise ValueError("URL too long")
        if not re.search(r"instagram\.com|instagr\.am", v):
            raise ValueError("Not an Instagram URL")
        return v
```

### `presentation/schemas/responses.py`

```python
from pydantic import BaseModel
from typing import Optional

class MediaItemSchema(BaseModel):
    index:            int
    type:             str
    url:              str
    thumbnail_url:    str
    width:            Optional[int]   = None
    height:           Optional[int]   = None
    duration_seconds: Optional[float] = None

class MediaDataSchema(BaseModel):
    content_type: str
    username:     str
    media_items:  list[MediaItemSchema]

class SuccessResponse(BaseModel):
    success: bool = True
    data:    MediaDataSchema

class ErrorResponse(BaseModel):
    success: bool  = False
    error:   str
    message: str
    detail:  Optional[str] = None
```

### `presentation/api/v1/endpoints/media.py`

```python
from fastapi import APIRouter, Request, Depends
from app.presentation.schemas.requests  import ResolveRequest
from app.presentation.schemas.responses import SuccessResponse, MediaDataSchema, MediaItemSchema
from app.application.use_cases.resolve_media import ResolveMediaUseCase
from app.application.dtos.media_dto import ResolveRequestDTO
from app.presentation.middleware.rate_limiter import limiter
from app.dependencies import get_resolve_use_case   # xem Dependency Injection

router = APIRouter(prefix="/media", tags=["media"])

@router.post("/resolve", response_model=SuccessResponse)
@limiter.limit("30/minute")
@limiter.limit("200/hour")
async def resolve_media(
    request: Request,
    body:    ResolveRequest,
    use_case: ResolveMediaUseCase = Depends(get_resolve_use_case),
) -> SuccessResponse:
    result = await use_case.execute(ResolveRequestDTO(url=body.url))

    return SuccessResponse(
        data=MediaDataSchema(
            content_type=result.content_type,
            username=result.username,
            media_items=[
                MediaItemSchema(**item.__dict__)
                for item in result.media_items
            ],
        )
    )
```

### `presentation/api/v1/endpoints/health.py`

```python
from fastapi import APIRouter
from app.config import settings

router = APIRouter(tags=["health"])

@router.get("/health")
async def health():
    return {"status": "ok", "version": settings.APP_VERSION}
```

### `presentation/middleware/rate_limiter.py`

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
```

### `presentation/api/v1/router.py`

```python
from fastapi import APIRouter
from app.presentation.api.v1.endpoints import media, health

router = APIRouter(prefix="/api/v1")
router.include_router(health.router)
router.include_router(media.router)
```

---

## 7. Dependency Injection

### `app/dependencies.py`

Wiring Infrastructure → Application: đây là nơi duy nhất các layer "gặp nhau".

```python
from functools import lru_cache
from app.infrastructure.extractors.ytdlp_extractor   import YtDlpExtractor
from app.infrastructure.extractors.og_meta_extractor import OgMetaExtractor
from app.application.use_cases.resolve_media import ResolveMediaUseCase

@lru_cache(maxsize=1)
def _get_extractors():
    return [YtDlpExtractor(), OgMetaExtractor()]

def get_resolve_use_case() -> ResolveMediaUseCase:
    return ResolveMediaUseCase(extractors=_get_extractors())
```

> `lru_cache` để tạo extractors một lần duy nhất. Nếu sau này cần scoped lifetime (per-request), đổi thành FastAPI `Depends` với `yield`.

### `app/main.py`

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from slowapi.errors import RateLimitExceeded
from slowapi import _rate_limit_exceeded_handler

from app.config import settings
from app.presentation.api.v1.router import router as v1_router
from app.presentation.middleware.rate_limiter import limiter
from app.presentation.middleware.cors import add_cors
from app.domain.exceptions import (
    DomainException, InvalidUrlError, ContentPrivateError,
    ContentNotFoundError, ContentNotSupportedError, ExtractionFailedError,
)
from app.presentation.schemas.responses import ErrorResponse

def create_app() -> FastAPI:
    app = FastAPI(
        title="InstaDown API",
        version=settings.APP_VERSION,
        docs_url="/docs" if settings.DEBUG else None,    # ẩn Swagger ở prod
        redoc_url=None,
    )

    # Middleware
    app.state.limiter = limiter
    app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
    add_cors(app)

    # Routers
    app.include_router(v1_router)

    # Exception handlers
    _register_exception_handlers(app)

    return app

def _register_exception_handlers(app: FastAPI) -> None:
    _ERROR_MAP: dict[type[DomainException], tuple[int, str]] = {
        InvalidUrlError:          (400, "The provided URL is not a valid Instagram URL."),
        ContentNotSupportedError: (400, "This content type is not supported yet."),
        ContentPrivateError:      (403, "This content is private or requires login."),
        ContentNotFoundError:     (404, "Content not found. It may have been deleted."),
        ExtractionFailedError:    (503, "Could not extract media. Please try again later."),
    }

    @app.exception_handler(DomainException)
    async def domain_exception_handler(request: Request, exc: DomainException) -> JSONResponse:
        status, message = _ERROR_MAP.get(type(exc), (500, "An unexpected error occurred."))
        return JSONResponse(
            status_code=status,
            content=ErrorResponse(
                error=exc.code,
                message=message,
                detail=exc.detail if settings.DEBUG else None,
            ).model_dump(),
        )

    @app.exception_handler(Exception)
    async def generic_handler(request: Request, exc: Exception) -> JSONResponse:
        return JSONResponse(
            status_code=500,
            content=ErrorResponse(
                error="INTERNAL_ERROR",
                message="An internal error occurred.",
            ).model_dump(),
        )

app = create_app()
```

### `app/config.py`

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

    APP_VERSION:     str  = "1.0.0"
    APP_ENV:         str  = "development"
    DEBUG:           bool = True

    HOST:            str  = "0.0.0.0"
    PORT:            int  = 8000
    WORKERS:         int  = 1

    ALLOWED_ORIGINS: list[str] = ["*"]

    RATE_PER_MINUTE: int  = 30
    RATE_PER_HOUR:   int  = 200

settings = Settings()
```

---

## 8. Error Handling

**Luồng xử lý lỗi end-to-end:**

```
User request
    │
    ▼ Pydantic validation (presentation/schemas)
    │ → HTTP 422 ValidationError tự động (FastAPI built-in)
    │
    ▼ InstagramUrl(raw_url)  (domain/value_objects)
    │ → raise InvalidUrlError  (DomainException)
    │
    ▼ ResolveMediaUseCase.execute()  (application)
    │ → raise ContentNotSupportedError  (stories)
    │ → raise ContentPrivateError / ContentNotFoundError  (từ extractor)
    │ → raise ExtractionFailedError  (tất cả extractor thất bại)
    │
    ▼ domain_exception_handler  (presentation/main.py)
    │ → map DomainException → HTTP status + ErrorResponse JSON
    │
    ▼ Client nhận:
       { "success": false, "error": "...", "message": "..." }
```

**Nguyên tắc:**
- `domain/exceptions.py` là nguồn sự thật duy nhất về các loại lỗi
- Presentation chỉ map, không tạo ra lỗi mới
- `detail` chỉ trả về client khi `DEBUG=True`
- `ExtractionFailedError` là lỗi "bất khả kháng" — log đầy đủ, trả về 503 generic

---

## 9. Setup & Run

### `.env.example`

```bash
APP_ENV=development
APP_VERSION=1.0.0
DEBUG=true

HOST=0.0.0.0
PORT=8000
WORKERS=1

ALLOWED_ORIGINS=["*"]
RATE_PER_MINUTE=30
RATE_PER_HOUR=200
```

### `requirements.txt`

```
fastapi==0.115.0
uvicorn[standard]==0.30.6
pydantic==2.9.2
pydantic-settings==2.5.2
httpx==0.27.2
beautifulsoup4==4.12.3
lxml==5.3.0
yt-dlp==2024.10.7
slowapi==0.1.9
```

### `requirements-dev.txt`

```
-r requirements.txt
pytest==8.3.3
pytest-asyncio==0.24.0
httpx
```

### Chạy local

```bash
# 1. Tạo virtualenv
python -m venv .venv && source .venv/bin/activate

# 2. Cài dependencies
pip install -r requirements-dev.txt

# 3. Copy env
cp .env.example .env

# 4. Chạy
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Swagger UI: http://localhost:8000/docs  (chỉ khi DEBUG=true)
```

### `Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y ffmpeg && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### `docker-compose.yml`

```yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    restart: unless-stopped
```

---

## 10. Testing Strategy

**Nguyên tắc:** Test theo layer, không test implementation detail.

### Fake extractor cho unit tests

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app
from app.domain.ports.media_extractor import IMediaExtractor
from app.domain.entities.media_info import MediaInfo, MediaItem
from app.domain.entities.content_type import ContentType
from app.domain.value_objects.instagram_url import InstagramUrl

class FakeExtractor(IMediaExtractor):
    """Extractor giả để test, không cần mạng."""

    def __init__(self, result: MediaInfo | Exception):
        self._result = result

    @property
    def name(self) -> str:
        return "fake"

    async def extract(self, url: InstagramUrl) -> MediaInfo:
        if isinstance(self._result, Exception):
            raise self._result
        return self._result

SAMPLE_PHOTO = MediaInfo(
    content_type=ContentType.PHOTO,
    username="test_user",
    media_items=[MediaItem(0, "photo", "https://cdn.ig.com/img.jpg", "https://cdn.ig.com/thumb.jpg", 1080, 1080)],
)

@pytest.fixture
def fake_photo_extractor():
    return FakeExtractor(result=SAMPLE_PHOTO)

@pytest.fixture
async def test_client():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        yield client
```

### Test Use Case (không cần HTTP)

```python
# tests/application/test_resolve_media.py
import pytest
from app.application.use_cases.resolve_media import ResolveMediaUseCase
from app.application.dtos.media_dto import ResolveRequestDTO
from app.domain.exceptions import ContentPrivateError, ContentNotSupportedError

@pytest.mark.asyncio
async def test_resolve_photo_success(fake_photo_extractor):
    use_case = ResolveMediaUseCase(extractors=[fake_photo_extractor])
    result = await use_case.execute(ResolveRequestDTO(url="https://www.instagram.com/p/ABC/"))
    assert result.content_type == "photo"
    assert result.username == "test_user"
    assert len(result.media_items) == 1

@pytest.mark.asyncio
async def test_fallback_to_second_extractor(fake_photo_extractor):
    from tests.conftest import FakeExtractor
    from app.domain.exceptions import ExtractionFailedError
    failing = FakeExtractor(result=ExtractionFailedError("first failed"))
    use_case = ResolveMediaUseCase(extractors=[failing, fake_photo_extractor])
    result = await use_case.execute(ResolveRequestDTO(url="https://www.instagram.com/p/ABC/"))
    assert result.content_type == "photo"

@pytest.mark.asyncio
async def test_stories_raises_not_supported():
    from tests.conftest import FakeExtractor, SAMPLE_PHOTO
    use_case = ResolveMediaUseCase(extractors=[FakeExtractor(SAMPLE_PHOTO)])
    with pytest.raises(ContentNotSupportedError):
        await use_case.execute(ResolveRequestDTO(url="https://www.instagram.com/stories/user/123/"))

@pytest.mark.asyncio
async def test_private_content_does_not_fallback():
    from tests.conftest import FakeExtractor, SAMPLE_PHOTO
    private  = FakeExtractor(result=ContentPrivateError())
    fallback = FakeExtractor(result=SAMPLE_PHOTO)
    use_case = ResolveMediaUseCase(extractors=[private, fallback])
    with pytest.raises(ContentPrivateError):
        await use_case.execute(ResolveRequestDTO(url="https://www.instagram.com/p/ABC/"))
```

### Test Domain Value Object

```python
# tests/domain/test_instagram_url.py
import pytest
from app.domain.value_objects.instagram_url import InstagramUrl
from app.domain.exceptions import InvalidUrlError

def test_normalize_removes_query_params():
    url = InstagramUrl("https://www.instagram.com/p/ABC123/?utm_source=ig_web")
    assert "?" not in url.value

def test_detect_reel_type():
    url = InstagramUrl("https://www.instagram.com/reel/XYZ789/")
    assert url.url_type == "reel"

def test_detect_stories():
    url = InstagramUrl("https://www.instagram.com/stories/user/123/")
    assert url.is_stories is True

def test_reject_non_instagram_domain():
    with pytest.raises(InvalidUrlError):
        InstagramUrl("https://example.com/p/ABC/")

def test_add_scheme_if_missing():
    url = InstagramUrl("www.instagram.com/p/ABC/")
    assert url.value.startswith("https://")
```

### Test Endpoint (integration)

```python
# tests/presentation/test_media_endpoint.py
import pytest
from app.dependencies import get_resolve_use_case
from app.application.use_cases.resolve_media import ResolveMediaUseCase
from app.main import app

@pytest.mark.asyncio
async def test_resolve_returns_200(test_client, fake_photo_extractor):
    # Override dependency
    app.dependency_overrides[get_resolve_use_case] = (
        lambda: ResolveMediaUseCase(extractors=[fake_photo_extractor])
    )
    resp = await test_client.post("/api/v1/media/resolve", json={"url": "https://www.instagram.com/p/ABC/"})
    assert resp.status_code == 200
    assert resp.json()["data"]["content_type"] == "photo"
    app.dependency_overrides.clear()

@pytest.mark.asyncio
async def test_invalid_url_returns_400(test_client):
    resp = await test_client.post("/api/v1/media/resolve", json={"url": "https://example.com/p/ABC/"})
    assert resp.status_code == 400
    assert resp.json()["error"] == "INVALID_URL"

@pytest.mark.asyncio
async def test_health_endpoint(test_client):
    resp = await test_client.get("/api/v1/health")
    assert resp.status_code == 200
    assert resp.json()["status"] == "ok"
```

### Chạy tests

```bash
pytest tests/ -v --asyncio-mode=auto

# Chỉ test một layer
pytest tests/domain/ -v
pytest tests/application/ -v

# Coverage report
pytest tests/ --cov=app --cov-report=term-missing
```
