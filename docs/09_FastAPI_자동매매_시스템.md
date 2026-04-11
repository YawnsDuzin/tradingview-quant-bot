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

## 9.9 `app/services/upbit_broker.py` - 업비트 브로커 서비스

```python
"""
업비트 매매 실행 서비스 (async 래퍼)
"""
from __future__ import annotations

import asyncio
import logging
from typing import Any

import pyupbit

from app.core.config import get_settings

logger = logging.getLogger(__name__)


class UpbitBroker:
    """업비트 비동기 브로커"""

    MIN_ORDER_KRW = 5000

    def __init__(self) -> None:
        settings = get_settings()
        if not settings.upbit_access_key or not settings.upbit_secret_key:
            raise ValueError("UPBIT API 키가 설정되지 않았습니다.")
        self.client = pyupbit.Upbit(
            settings.upbit_access_key,
            settings.upbit_secret_key,
        )

    async def _run(self, func, *args, **kwargs):
        """동기 함수를 비동기로 실행 (pyupbit은 동기 라이브러리)"""
        loop = asyncio.get_running_loop()
        return await loop.run_in_executor(None, lambda: func(*args, **kwargs))

    async def get_krw_balance(self) -> float:
        """원화 잔고 조회"""
        balance = await self._run(self.client.get_balance, "KRW")
        return float(balance) if balance else 0.0

    async def get_coin_balance(self, symbol: str) -> float:
        """코인 보유 수량 조회 (symbol: KRW-BTC)"""
        coin = symbol.split("-")[1] if "-" in symbol else symbol
        balance = await self._run(self.client.get_balance, coin)
        return float(balance) if balance else 0.0

    async def get_current_price(self, symbol: str) -> float:
        """현재가 조회"""
        price = await self._run(pyupbit.get_current_price, symbol)
        return float(price) if price else 0.0

    async def buy_market_pct(
        self,
        symbol: str,
        pct: float,
    ) -> dict[str, Any] | None:
        """
        원화 잔고의 일정 비율로 시장가 매수

        Args:
            symbol: "KRW-BTC" 형식
            pct: 매수 비율 (0.0 ~ 1.0)
        """
        krw = await self.get_krw_balance()
        amount = krw * pct

        if amount < self.MIN_ORDER_KRW:
            logger.warning(f"매수 금액 부족: {amount:,.0f} KRW")
            return None

        try:
            result = await self._run(
                self.client.buy_market_order,
                symbol,
                amount * 0.9995,  # 수수료 반영
            )
            logger.info(f"[UPBIT BUY] {symbol} @ {amount:,.0f} KRW")
            return result
        except Exception as e:
            logger.error(f"[UPBIT BUY] 실패: {e}")
            return None

    async def sell_market_all(self, symbol: str) -> dict[str, Any] | None:
        """보유 전량 시장가 매도"""
        volume = await self.get_coin_balance(symbol)
        if volume <= 0:
            logger.warning(f"[UPBIT SELL] {symbol} 보유 수량 없음")
            return None

        current_price = await self.get_current_price(symbol)
        if volume * current_price < self.MIN_ORDER_KRW:
            logger.warning(f"[UPBIT SELL] 금액 부족")
            return None

        try:
            result = await self._run(
                self.client.sell_market_order,
                symbol,
                volume,
            )
            logger.info(f"[UPBIT SELL] {symbol} x {volume}")
            return result
        except Exception as e:
            logger.error(f"[UPBIT SELL] 실패: {e}")
            return None
```

---

## 9.10 `app/services/kis_broker.py` - KIS 브로커 서비스

```python
"""
한국투자증권 KIS API 매매 서비스
"""
from __future__ import annotations

import json
import logging
from datetime import datetime, timedelta
from typing import Any, Literal

import httpx

from app.core.config import Settings, get_settings

logger = logging.getLogger(__name__)

OrderSide = Literal["BUY", "SELL"]


class KISBroker:
    """한국투자증권 KIS API 비동기 브로커"""

    def __init__(self) -> None:
        self.settings: Settings = get_settings()
        self._access_token: str | None = None
        self._token_expires_at: datetime | None = None
        self._client = httpx.AsyncClient(timeout=10.0)

    async def close(self) -> None:
        """HTTP 클라이언트 정리"""
        await self._client.aclose()

    async def _get_access_token(self) -> str:
        """Access Token 발급/재사용"""
        if (
            self._access_token
            and self._token_expires_at
            and self._token_expires_at > datetime.now() + timedelta(minutes=5)
        ):
            return self._access_token

        url = f"{self.settings.kis_base_url}/oauth2/tokenP"
        headers = {"content-type": "application/json"}
        body = {
            "grant_type": "client_credentials",
            "appkey": self.settings.kis_app_key,
            "appsecret": self.settings.kis_app_secret,
        }

        resp = await self._client.post(url, headers=headers, content=json.dumps(body))
        resp.raise_for_status()
        data = resp.json()

        self._access_token = data["access_token"]
        self._token_expires_at = datetime.now() + timedelta(
            seconds=data.get("expires_in", 86400)
        )
        return self._access_token  # type: ignore

    async def _auth_headers(self, tr_id: str) -> dict[str, str]:
        """공통 인증 헤더"""
        token = await self._get_access_token()
        return {
            "content-type": "application/json; charset=utf-8",
            "authorization": f"Bearer {token}",
            "appkey": self.settings.kis_app_key,
            "appsecret": self.settings.kis_app_secret,
            "tr_id": tr_id,
        }

    async def get_price(self, stock_code: str) -> int:
        """국내 주식 현재가 조회"""
        url = f"{self.settings.kis_base_url}/uapi/domestic-stock/v1/quotations/inquire-price"
        headers = await self._auth_headers("FHKST01010100")
        params = {
            "FID_COND_MRKT_DIV_CODE": "J",
            "FID_INPUT_ISCD": stock_code,
        }
        resp = await self._client.get(url, headers=headers, params=params)
        resp.raise_for_status()
        data = resp.json()
        if data["rt_cd"] != "0":
            raise RuntimeError(f"시세 조회 실패: {data['msg1']}")
        return int(data["output"]["stck_prpr"])

    async def get_cash_balance(self) -> int:
        """예수금 조회"""
        url = f"{self.settings.kis_base_url}/uapi/domestic-stock/v1/trading/inquire-balance"
        prefix = "V" if self.settings.kis_mode == "paper" else "T"
        headers = await self._auth_headers(f"{prefix}TTC8434R")
        params = {
            "CANO": self.settings.kis_cano,
            "ACNT_PRDT_CD": self.settings.kis_acnt_prdt_cd,
            "AFHR_FLPR_YN": "N",
            "OFL_YN": "",
            "INQR_DVSN": "02",
            "UNPR_DVSN": "01",
            "FUND_STTL_ICLD_YN": "N",
            "FNCG_AMT_AUTO_RDPT_YN": "N",
            "PRCS_DVSN": "01",
            "CTX_AREA_FK100": "",
            "CTX_AREA_NK100": "",
        }
        resp = await self._client.get(url, headers=headers, params=params)
        resp.raise_for_status()
        data = resp.json()
        if data["rt_cd"] != "0":
            raise RuntimeError(f"잔고 조회 실패: {data['msg1']}")
        summary = data["output2"][0] if data["output2"] else {}
        return int(summary.get("dnca_tot_amt", 0))

    async def place_order(
        self,
        stock_code: str,
        side: OrderSide,
        quantity: int,
        price: int = 0,
        market: bool = True,
    ) -> dict[str, Any]:
        """주식 주문 실행"""
        url = f"{self.settings.kis_base_url}/uapi/domestic-stock/v1/trading/order-cash"

        prefix = "V" if self.settings.kis_mode == "paper" else "T"
        tr_id = f"{prefix}TTC0802U" if side == "BUY" else f"{prefix}TTC0801U"

        headers = await self._auth_headers(tr_id)
        body = {
            "CANO": self.settings.kis_cano,
            "ACNT_PRDT_CD": self.settings.kis_acnt_prdt_cd,
            "PDNO": stock_code,
            "ORD_DVSN": "01" if market else "00",
            "ORD_QTY": str(quantity),
            "ORD_UNPR": "0" if market else str(price),
        }

        resp = await self._client.post(url, headers=headers, content=json.dumps(body))
        resp.raise_for_status()
        data = resp.json()

        if data["rt_cd"] != "0":
            raise RuntimeError(f"주문 실패: {data['msg1']}")

        return {
            "order_no": data["output"]["ODNO"],
            "order_time": data["output"]["ORD_TMD"],
            "message": data["msg1"],
        }

    async def buy_by_krw(
        self,
        stock_code: str,
        krw_amount: int,
    ) -> dict[str, Any] | None:
        """원화 금액 기준 시장가 매수 (수량 자동 계산)"""
        price = await self.get_price(stock_code)
        quantity = krw_amount // price
        if quantity <= 0:
            logger.warning(f"매수 수량 0: price={price}, krw={krw_amount}")
            return None
        return await self.place_order(stock_code, "BUY", int(quantity), market=True)
```

---

## 9.11 `app/services/notify.py` - 텔레그램 알림

```python
"""
텔레그램 알림 서비스
"""
from __future__ import annotations

import logging

import httpx

from app.core.config import get_settings

logger = logging.getLogger(__name__)


async def send_telegram_message(text: str, parse_mode: str = "HTML") -> bool:
    """
    텔레그램 메시지 전송

    Args:
        text: 메시지 본문 (HTML 태그 사용 가능)
        parse_mode: "HTML" 또는 "Markdown"

    Returns:
        전송 성공 여부
    """
    settings = get_settings()
    if not settings.telegram_bot_token or not settings.telegram_chat_id:
        logger.warning("텔레그램 설정이 없어 알림을 건너뜁니다.")
        return False

    url = f"https://api.telegram.org/bot{settings.telegram_bot_token}/sendMessage"
    payload = {
        "chat_id": settings.telegram_chat_id,
        "text": text,
        "parse_mode": parse_mode,
    }

    try:
        async with httpx.AsyncClient(timeout=10) as client:
            resp = await client.post(url, json=payload)
            resp.raise_for_status()
            return True
    except Exception as e:
        logger.error(f"텔레그램 전송 실패: {e}")
        return False


async def notify_trade(
    action: str,
    symbol: str,
    price: float,
    quantity: float | None = None,
    strategy: str = "",
) -> None:
    """매매 체결 알림"""
    emoji = "🟢" if action == "BUY" else "🔴"
    qty_str = f"\n수량: {quantity}" if quantity else ""
    text = (
        f"{emoji} <b>{action}</b> 체결\n"
        f"종목: {symbol}\n"
        f"가격: {price:,.0f}{qty_str}\n"
        f"전략: {strategy}"
    )
    await send_telegram_message(text)


async def notify_error(title: str, detail: str) -> None:
    """에러 알림"""
    text = f"⚠️ <b>{title}</b>\n\n<code>{detail}</code>"
    await send_telegram_message(text)
```

---

## 9.12 `app/services/signal.py` - 신호 처리 서비스

```python
"""
Alert 신호 파싱 및 매매 실행 분기
"""
from __future__ import annotations

import logging
from datetime import datetime

from sqlalchemy.orm import Session

from app.models.order import OrderLog
from app.schemas.alert import TradingViewAlert
from app.services.kis_broker import KISBroker
from app.services.notify import notify_error, notify_trade
from app.services.upbit_broker import UpbitBroker

logger = logging.getLogger(__name__)


class SignalService:
    """TradingView 신호를 거래소 주문으로 실행"""

    def __init__(self) -> None:
        self.upbit: UpbitBroker | None = None
        self.kis: KISBroker | None = None

    async def get_upbit(self) -> UpbitBroker:
        if self.upbit is None:
            self.upbit = UpbitBroker()
        return self.upbit

    async def get_kis(self) -> KISBroker:
        if self.kis is None:
            self.kis = KISBroker()
        return self.kis

    async def handle_alert(
        self,
        alert: TradingViewAlert,
        db: Session,
    ) -> OrderLog:
        """
        Alert 처리 메인 진입점

        1. 로그 저장
        2. 거래소별 분기
        3. 매매 실행
        4. 결과 업데이트
        5. 알림 발송
        """
        # 1. 주문 로그 저장
        log = OrderLog(
            strategy=alert.strategy,
            symbol=alert.symbol,
            exchange=alert.exchange,
            action=alert.action,
            signal_price=alert.price,
            quantity_pct=alert.quantity_pct,
        )
        db.add(log)
        db.commit()
        db.refresh(log)

        # 2. 거래소별 분기 실행
        try:
            if alert.exchange == "UPBIT":
                result = await self._execute_upbit(alert)
            elif alert.exchange == "KIS":
                result = await self._execute_kis(alert)
            else:
                raise ValueError(f"지원하지 않는 거래소: {alert.exchange}")

            # 3. 결과 반영
            if result:
                log.executed = True
                log.executed_at = datetime.utcnow()
                log.order_uuid = str(result.get("order_no") or result.get("uuid", ""))
                db.commit()

                # 4. 텔레그램 알림
                await notify_trade(
                    action=alert.action,
                    symbol=alert.symbol,
                    price=alert.price,
                    strategy=alert.strategy,
                )
        except Exception as e:
            logger.exception("신호 처리 실패")
            log.error_message = str(e)
            db.commit()
            await notify_error("매매 실행 실패", str(e))

        return log

    async def _execute_upbit(self, alert: TradingViewAlert) -> dict | None:
        """업비트 매매 실행"""
        broker = await self.get_upbit()
        pct = alert.quantity_pct / 100.0

        if alert.action == "BUY":
            return await broker.buy_market_pct(alert.symbol, pct)
        elif alert.action in ("SELL", "CLOSE"):
            return await broker.sell_market_all(alert.symbol)
        return None

    async def _execute_kis(self, alert: TradingViewAlert) -> dict | None:
        """KIS 주식 매매 실행"""
        broker = await self.get_kis()

        if alert.action == "BUY":
            # 전체 예수금의 일정 비율
            cash = await broker.get_cash_balance()
            target_amount = int(cash * alert.quantity_pct / 100.0)
            return await broker.buy_by_krw(alert.symbol, target_amount)

        # SELL/CLOSE는 별도 수량 조회 로직 필요 (보유 수량 전량 매도)
        # 간단화를 위해 본 예제에서는 BUY만 처리. 실전에서는 잔고 조회 필요.
        logger.warning("KIS SELL/CLOSE는 보유 수량 조회 로직 구현 필요")
        return None


# 싱글톤 인스턴스
signal_service = SignalService()
```

---

## 9.13 `app/routers/webhook.py` - Webhook 수신 라우터

```python
"""
TradingView Webhook 수신 라우터
"""
from __future__ import annotations

from datetime import datetime, timedelta

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from app.core.security import verify_webhook_secret
from app.database import get_db
from app.models.order import OrderLog
from app.schemas.alert import AlertResponse, TradingViewAlert
from app.services.signal import signal_service

router = APIRouter(prefix="/webhook", tags=["webhook"])


def _is_duplicate(db: Session, alert: TradingViewAlert) -> bool:
    """60초 이내 동일 신호인지 확인"""
    cutoff = datetime.utcnow() - timedelta(seconds=60)
    exists = (
        db.query(OrderLog)
        .filter(
            OrderLog.strategy == alert.strategy,
            OrderLog.symbol == alert.symbol,
            OrderLog.action == alert.action,
            OrderLog.received_at >= cutoff,
        )
        .first()
    )
    return exists is not None


@router.post("", response_model=AlertResponse)
async def receive_webhook(
    alert: TradingViewAlert,
    db: Session = Depends(get_db),
) -> AlertResponse:
    """TradingView Alert 수신"""

    # 1. Secret 검증
    if not verify_webhook_secret(alert.secret):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid webhook secret",
        )

    # 2. 중복 체크
    if _is_duplicate(db, alert):
        return AlertResponse(
            status="ignored",
            message="Duplicate signal within 60s",
            received_at=datetime.utcnow(),
        )

    # 3. 신호 처리 실행
    log = await signal_service.handle_alert(alert, db)

    return AlertResponse(
        status="success" if log.executed else "error",
        message=log.error_message or f"{alert.action} {alert.symbol} executed",
        order_id=log.id,
        received_at=log.received_at,
    )
```

---

## 9.14 `app/routers/orders.py` - 수동 주문 API

```python
"""
수동 주문 및 주문 내역 조회 API
"""
from __future__ import annotations

from typing import Literal

from fastapi import APIRouter, Depends, Query
from pydantic import BaseModel, Field
from sqlalchemy.orm import Session

from app.database import get_db
from app.models.order import OrderLog
from app.services.signal import signal_service
from app.schemas.alert import TradingViewAlert

router = APIRouter(prefix="/orders", tags=["orders"])


class ManualOrder(BaseModel):
    """수동 주문 요청"""

    exchange: Literal["UPBIT", "KIS"]
    symbol: str
    action: Literal["BUY", "SELL", "CLOSE"]
    quantity_pct: float = Field(default=10.0, ge=0.0, le=100.0)


@router.post("/manual")
async def manual_order(
    order: ManualOrder,
    db: Session = Depends(get_db),
) -> dict:
    """수동 주문 실행 (테스트/관리용)"""
    from app.core.config import get_settings

    settings = get_settings()

    # 내부적으로 TradingViewAlert 형태로 변환
    alert = TradingViewAlert(
        secret=settings.webhook_secret,
        strategy="manual",
        symbol=order.symbol,
        exchange=order.exchange,
        action=order.action,
        price=0.0 or 1.0,  # 수동 주문은 price 무관 (버그 방지용 1.0)
        quantity_pct=order.quantity_pct,
    )

    log = await signal_service.handle_alert(alert, db)
    return {
        "order_id": log.id,
        "executed": log.executed,
        "error": log.error_message,
    }


@router.get("/history")
async def get_order_history(
    limit: int = Query(50, ge=1, le=500),
    db: Session = Depends(get_db),
) -> list[dict]:
    """주문 내역 조회"""
    logs = (
        db.query(OrderLog)
        .order_by(OrderLog.received_at.desc())
        .limit(limit)
        .all()
    )
    return [
        {
            "id": log.id,
            "strategy": log.strategy,
            "symbol": log.symbol,
            "exchange": log.exchange,
            "action": log.action,
            "signal_price": log.signal_price,
            "executed": log.executed,
            "received_at": log.received_at.isoformat(),
            "executed_at": log.executed_at.isoformat() if log.executed_at else None,
            "error_message": log.error_message,
        }
        for log in logs
    ]
```

---

## 9.15 `app/routers/status.py` - 시스템 상태 라우터

```python
"""
시스템 상태 및 잔고 조회 API
"""
from __future__ import annotations

from fastapi import APIRouter

from app.core.config import get_settings
from app.services.signal import signal_service

router = APIRouter(prefix="/status", tags=["status"])


@router.get("/")
async def health_check() -> dict:
    """헬스체크"""
    settings = get_settings()
    return {
        "status": "ok",
        "app_name": settings.app_name,
        "env": settings.app_env,
        "kis_mode": settings.kis_mode,
    }


@router.get("/balance/upbit")
async def upbit_balance() -> dict:
    """업비트 잔고 조회"""
    try:
        broker = await signal_service.get_upbit()
        krw = await broker.get_krw_balance()
        return {"krw_balance": krw}
    except Exception as e:
        return {"error": str(e)}


@router.get("/balance/kis")
async def kis_balance() -> dict:
    """KIS 예수금 조회"""
    try:
        broker = await signal_service.get_kis()
        cash = await broker.get_cash_balance()
        return {"cash": cash}
    except Exception as e:
        return {"error": str(e)}
```

---

## 9.16 `app/main.py` - FastAPI 메인 앱

```python
"""
FastAPI 메인 애플리케이션
"""
from __future__ import annotations

import logging
from contextlib import asynccontextmanager

from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
from fastapi import FastAPI

from app.core.config import get_settings
from app.database import SessionLocal, init_db
from app.routers import orders, status, webhook
from app.services.notify import send_telegram_message
from app.services.signal import signal_service

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s - %(message)s",
)
logger = logging.getLogger(__name__)


scheduler = AsyncIOScheduler()


async def daily_report_job() -> None:
    """매일 오후 6시 잔고 리포트"""
    try:
        upbit = await signal_service.get_upbit()
        krw = await upbit.get_krw_balance()

        report = f"📊 <b>일일 리포트</b>\n원화 잔고: {krw:,.0f} KRW"
        await send_telegram_message(report)
    except Exception as e:
        logger.error(f"일일 리포트 실패: {e}")


@asynccontextmanager
async def lifespan(app: FastAPI):
    """앱 시작/종료 시 실행할 작업"""
    # 시작
    logger.info("앱 초기화 시작")
    init_db()

    # APScheduler 등록
    scheduler.add_job(
        daily_report_job,
        CronTrigger(hour=18, minute=0),  # 매일 18:00
        id="daily_report",
    )
    scheduler.start()
    logger.info("스케줄러 시작됨")

    # 시작 알림
    await send_telegram_message("🚀 자동매매 봇이 시작되었습니다.")

    yield

    # 종료
    scheduler.shutdown()
    if signal_service.kis:
        await signal_service.kis.close()
    await send_telegram_message("🛑 자동매매 봇이 종료되었습니다.")
    logger.info("앱 종료")


settings = get_settings()

app = FastAPI(
    title=settings.app_name,
    version="1.0.0",
    debug=settings.debug,
    lifespan=lifespan,
)

# 라우터 등록
app.include_router(webhook.router)
app.include_router(orders.router)
app.include_router(status.router)


@app.get("/")
async def root() -> dict[str, str]:
    return {"message": f"{settings.app_name} is running"}
```

---

## 9.17 서버 실행 방법

### 9.17.1 패키지 설치

```bash
# uv로 프로젝트 초기화
uv init
uv add fastapi uvicorn pydantic pydantic-settings sqlalchemy httpx \
       pyupbit apscheduler python-dotenv

# 또는 pip
pip install fastapi uvicorn pydantic pydantic-settings sqlalchemy httpx \
            pyupbit apscheduler python-dotenv
```

### 9.17.2 `.env` 파일 생성

```bash
cp .env.example .env
# .env 파일을 편집하여 실제 API 키 입력
```

### 9.17.3 Uvicorn으로 실행

```bash
# 개발 모드 (코드 변경 시 자동 재시작)
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

# 프로덕션 모드
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 2
```

### 9.17.4 접속 확인

- **API 문서**: http://localhost:8000/docs
- **상태 확인**: http://localhost:8000/status/
- **수동 주문**: http://localhost:8000/docs → `/orders/manual`

### 9.17.5 Webhook URL 설정

TradingView Alert 설정 시:
- **Webhook URL**: `https://your-domain.com/webhook`
- **Message**: 7장에서 배운 JSON 포맷

> 💡 **로컬 테스트 팁**
> 로컬에서 TradingView Webhook을 받으려면 ngrok을 사용하세요:
> ```bash
> ngrok http 8000
> ```
> 발급된 HTTPS URL + `/webhook`을 TradingView에 입력합니다.

---

## 9.18 전체 흐름 테스트

### 테스트 시나리오

1. **서버 시작**: `uvicorn app.main:app --reload`
2. **텔레그램 알림 확인**: "🚀 자동매매 봇 시작" 메시지 수신
3. **상태 조회**: http://localhost:8000/status/
4. **수동 주문 테스트**: `/docs`에서 `/orders/manual` 호출
5. **Webhook 테스트**: curl로 POST 전송

### curl 테스트 명령어

```bash
curl -X POST http://localhost:8000/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "secret": "your-webhook-secret",
    "strategy": "manual_test",
    "symbol": "KRW-BTC",
    "exchange": "UPBIT",
    "action": "BUY",
    "price": 60000000,
    "quantity_pct": 1
  }'
```

### 예상 결과

1. FastAPI 콘솔에 로그 출력
2. DB에 `OrderLog` 저장
3. 업비트 API로 실제 매수 주문
4. 텔레그램 알림 수신
5. 응답으로 `{"status":"success","order_id":1,...}` 반환

---

## 9.19 9장 요약

- FastAPI 기반 자동매매 서버를 완전한 형태로 구축했습니다.
- **계층 분리**: `core`(설정/보안), `services`(브로커/알림/신호), `routers`(API), `models`/`schemas`(데이터)
- **비동기 처리**: `httpx`, `async/await`로 블로킹 최소화
- **APScheduler**: 정기 리포트 자동 실행
- **텔레그램 알림**: 매매 체결 및 에러 실시간 통보
- **Secret Token + 중복 방지**: Webhook 보안
- **다음 장에서는 백테스팅**으로 전략의 과거 성과를 검증합니다.

> ⚠️ **실전 투입 전 체크리스트**
> - [ ] `.env` 파일이 `.gitignore`에 포함되어 있는가?
> - [ ] Webhook Secret이 충분히 강력한가?
> - [ ] KIS는 `paper` 모드로 시작하는가?
> - [ ] 업비트는 **출금 권한이 비활성화**되어 있는가?
> - [ ] 소액 매매로 충분히 테스트했는가?
> - [ ] 텔레그램 알림이 정상 작동하는가?

---

> 📖 **다음 장**: [10장. 백테스팅 시스템 구축](./10_백테스팅_시스템.md)
