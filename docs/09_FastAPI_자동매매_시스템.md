# 9장. FastAPI 기반 자동매매 시스템 구축

> 지금까지 배운 모든 내용을 종합하여, TradingView → FastAPI → 거래소 API로 이어지는
> **완전한 자동매매 시스템**을 구축합니다.

---

## 9.1 전체 시스템 아키텍처

### 시스템 흐름도

```
┌────────────────────────────────────────────────────────────────┐
│                       자동매매 시스템 전체 흐름                    │
└────────────────────────────────────────────────────────────────┘

┌─────────────────┐
│  TradingView     │
│                  │
│  ┌────────────┐  │
│  │ Pine Script │  │
│  │  + Alert    │  │
│  └─────┬──────┘  │
└────────┼────────┘
         │ HTTPS POST (JSON)
         ▼
┌─────────────────────────────────────┐
│            FastAPI 서버               │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ /webhook (수신)                │ │
│  └────────────┬───────────────────┘ │
│               │                      │
│               ▼                      │
│  ┌────────────────────────────────┐ │
│  │ 1. Secret Token 검증            │ │
│  │ 2. JSON 스키마 파싱 (Pydantic)   │ │
│  │ 3. 중복 신호 체크                │ │
│  │ 4. DB 로그 저장 (SQLAlchemy)     │ │
│  └────────────┬───────────────────┘ │
│               │                      │
│               ▼                      │
│  ┌────────────────────────────────┐ │
│  │ 신호 분기 (signal service)       │ │
│  └──┬─────────────────────┬───────┘ │
│     │                     │          │
│     │ exchange=UPBIT      │ exchange=KIS
│     ▼                     ▼          │
│  ┌──────────┐       ┌──────────┐    │
│  │ upbit_    │       │ kis_      │    │
│  │ broker    │       │ broker    │    │
│  └────┬─────┘       └─────┬────┘    │
└───────┼─────────────────────┼───────┘
        │                     │
        ▼                     ▼
┌────────────┐       ┌──────────────┐
│ Upbit API   │       │ KIS Dev API  │
└─────────────┘       └──────────────┘
        │                     │
        └──────────┬──────────┘
                   ▼
          ┌───────────────┐
          │ Telegram Bot   │
          │ (매매 알림)     │
          └───────────────┘
                   │
                   ▼
          ┌───────────────┐
          │  SQLite DB    │
          │ (주문 기록)     │
          └───────────────┘
```

### 기술 스택 요약

| 구성 요소 | 기술 |
|----------|------|
| **웹 프레임워크** | FastAPI + Uvicorn |
| **데이터 검증** | Pydantic v2 |
| **환경변수** | pydantic-settings |
| **DB ORM** | SQLAlchemy 2.0 |
| **DB** | SQLite (입문) / PostgreSQL (심화) |
| **스케줄링** | APScheduler |
| **암호화폐 API** | pyupbit |
| **주식 API** | KIS Developers (requests) |
| **알림** | python-telegram-bot |

---

## 9.2 프로젝트 폴더 구조

```
tradingview-quant-bot/
├── app/
│   ├── __init__.py
│   ├── main.py                  # FastAPI 진입점
│   │
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py            # pydantic-settings 환경변수
│   │   └── security.py          # Token 검증
│   │
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── webhook.py           # TradingView Webhook 수신
│   │   ├── orders.py            # 수동 주문 API
│   │   └── status.py            # 시스템 상태 조회
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── signal.py            # 신호 파싱 및 검증
│   │   ├── upbit_broker.py      # 업비트 매매 실행
│   │   ├── kis_broker.py        # KIS 주식 매매 실행
│   │   └── notify.py            # 텔레그램 알림
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   └── order.py             # SQLAlchemy 모델
│   │
│   ├── schemas/
│   │   ├── __init__.py
│   │   └── alert.py             # Pydantic 스키마
│   │
│   └── database.py              # DB 연결 설정
│
├── .env                         # 환경변수 (Git 제외)
├── .env.example                 # 환경변수 템플릿
├── .gitignore
├── pyproject.toml
└── requirements.txt
```

---

## 9.3 환경변수 템플릿 (`.env.example`)

```bash
# ============================================================
# FastAPI 서버 설정
# ============================================================
APP_NAME="TradingView Quant Bot"
APP_ENV=development  # development / production
DEBUG=true

# ============================================================
# Webhook 보안
# ============================================================
WEBHOOK_SECRET=CHANGE-ME-to-strong-random-string

# ============================================================
# 데이터베이스
# ============================================================
DATABASE_URL=sqlite:///./quant_bot.db

# ============================================================
# 업비트 API
# ============================================================
UPBIT_ACCESS_KEY=your-upbit-access-key
UPBIT_SECRET_KEY=your-upbit-secret-key

# ============================================================
# 한국투자증권 KIS API
# ============================================================
KIS_MODE=paper  # paper(모의) / real(실전)
KIS_APP_KEY=your-kis-app-key
KIS_APP_SECRET=your-kis-app-secret
KIS_ACCOUNT_NO=12345678-01

# ============================================================
# 텔레그램 알림
# ============================================================
TELEGRAM_BOT_TOKEN=your-telegram-bot-token
TELEGRAM_CHAT_ID=your-telegram-chat-id

# ============================================================
# 리스크 관리
# ============================================================
MAX_POSITION_PCT=20.0  # 종목당 최대 포지션 (%)
DAILY_LOSS_LIMIT_PCT=5.0  # 일일 최대 손실 (%)
```

---

## 9.4 `app/core/config.py` - 환경변수 관리

```python
"""
pydantic-settings 기반 환경변수 관리
"""
from __future__ import annotations

from functools import lru_cache
from typing import Literal

from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """애플리케이션 설정"""

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )

    # --- 앱 ---
    app_name: str = Field(default="TradingView Quant Bot")
    app_env: Literal["development", "production"] = "development"
    debug: bool = True

    # --- Webhook 보안 ---
    webhook_secret: str = Field(default="change-me")

    # --- DB ---
    database_url: str = Field(default="sqlite:///./quant_bot.db")

    # --- 업비트 ---
    upbit_access_key: str = ""
    upbit_secret_key: str = ""

    # --- KIS ---
    kis_mode: Literal["paper", "real"] = "paper"
    kis_app_key: str = ""
    kis_app_secret: str = ""
    kis_account_no: str = ""

    # --- 텔레그램 ---
    telegram_bot_token: str = ""
    telegram_chat_id: str = ""

    # --- 리스크 ---
    max_position_pct: float = 20.0
    daily_loss_limit_pct: float = 5.0

    @property
    def kis_base_url(self) -> str:
        if self.kis_mode == "real":
            return "https://openapi.koreainvestment.com:9443"
        return "https://openapivts.koreainvestment.com:29443"

    @property
    def kis_cano(self) -> str:
        return self.kis_account_no.split("-")[0] if "-" in self.kis_account_no else self.kis_account_no[:8]

    @property
    def kis_acnt_prdt_cd(self) -> str:
        return self.kis_account_no.split("-")[1] if "-" in self.kis_account_no else self.kis_account_no[8:]


@lru_cache
def get_settings() -> Settings:
    """설정 싱글톤"""
    return Settings()
```

---

## 9.5 `app/core/security.py` - Token 검증

```python
"""
Webhook Secret Token 검증
"""
from __future__ import annotations

import hmac

from app.core.config import get_settings


def verify_webhook_secret(received_secret: str) -> bool:
    """
    타이밍 공격 방지를 위해 hmac.compare_digest 사용

    Args:
        received_secret: 요청에 포함된 토큰

    Returns:
        검증 성공 여부
    """
    expected = get_settings().webhook_secret
    return hmac.compare_digest(received_secret, expected)
```

---

## 9.6 `app/schemas/alert.py` - Pydantic 스키마

```python
"""
TradingView Alert 및 API 응답 스키마
"""
from __future__ import annotations

from datetime import datetime
from typing import Literal, Optional

from pydantic import BaseModel, ConfigDict, Field


class TradingViewAlert(BaseModel):
    """TradingView Webhook 요청 스키마"""

    model_config = ConfigDict(
        json_schema_extra={
            "example": {
                "secret": "my-secret",
                "strategy": "golden_cross",
                "symbol": "KRW-BTC",
                "exchange": "UPBIT",
                "action": "BUY",
                "price": 60000000,
                "quantity_pct": 20,
            }
        }
    )

    secret: str = Field(..., description="보안 토큰")
    strategy: str = Field(..., min_length=1, max_length=100)
    symbol: str = Field(..., min_length=1, max_length=50)
    exchange: Literal["UPBIT", "KIS"] = Field(...)
    action: Literal["BUY", "SELL", "CLOSE"] = Field(...)
    price: float = Field(..., gt=0)
    quantity_pct: float = Field(default=10.0, ge=0.0, le=100.0)
    timestamp: Optional[str] = None


class AlertResponse(BaseModel):
    """Webhook 응답 스키마"""

    status: Literal["success", "ignored", "error"]
    message: str
    order_id: Optional[int] = None
    received_at: datetime
```

---

## 9.7 `app/models/order.py` - SQLAlchemy 모델

```python
"""
주문 및 Alert 로그 데이터베이스 모델
"""
from __future__ import annotations

from datetime import datetime

from sqlalchemy import Boolean, DateTime, Float, Integer, String, Text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class OrderLog(Base):
    """주문 로그 (Alert 수신 + 실행 결과)"""

    __tablename__ = "order_logs"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)

    # Alert 정보
    strategy: Mapped[str] = mapped_column(String(100), index=True)
    symbol: Mapped[str] = mapped_column(String(50), index=True)
    exchange: Mapped[str] = mapped_column(String(20), index=True)
    action: Mapped[str] = mapped_column(String(10))
    signal_price: Mapped[float] = mapped_column(Float)
    quantity_pct: Mapped[float] = mapped_column(Float, default=10.0)

    # 실행 결과
    executed: Mapped[bool] = mapped_column(Boolean, default=False)
    executed_price: Mapped[float | None] = mapped_column(Float, nullable=True)
    executed_quantity: Mapped[float | None] = mapped_column(Float, nullable=True)
    order_uuid: Mapped[str | None] = mapped_column(String(100), nullable=True)

    # 에러 메시지
    error_message: Mapped[str | None] = mapped_column(Text, nullable=True)

    # 시간
    received_at: Mapped[datetime] = mapped_column(
        DateTime, default=datetime.utcnow, index=True
    )
    executed_at: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)
```

---

## 9.8 `app/database.py` - DB 연결 설정

```python
"""
SQLAlchemy 데이터베이스 연결
"""
from __future__ import annotations

from collections.abc import Generator

from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

from app.core.config import get_settings
from app.models.order import Base

settings = get_settings()

# SQLite는 multithreading 옵션 필요
connect_args: dict = (
    {"check_same_thread": False}
    if settings.database_url.startswith("sqlite")
    else {}
)

engine = create_engine(
    settings.database_url,
    connect_args=connect_args,
    echo=settings.debug,
)

SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False)


def init_db() -> None:
    """앱 시작 시 테이블 생성"""
    Base.metadata.create_all(bind=engine)


def get_db() -> Generator[Session, None, None]:
    """FastAPI 의존성 주입용 DB 세션"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

---

> 📖 **다음**: [9장 Part 2에서는 브로커 서비스, 알림, 메인 앱, 라우터를 구현합니다]
