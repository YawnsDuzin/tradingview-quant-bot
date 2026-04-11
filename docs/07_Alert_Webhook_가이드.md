# 7장. TradingView Alert + Webhook 완전 가이드

> TradingView의 Alert 기능과 Webhook을 이용하면, TradingView에서 발생한 매매 신호를
> 우리 서버(FastAPI)로 실시간 전송할 수 있습니다.
> 이것이 **자동매매 시스템의 핵심 연결 고리**입니다.

---

## 7.1 Alert(알림) 기본 개념

### Alert이란?

**Alert(알림)**은 **특정 조건이 충족되면 자동으로 알림을 발생**시키는 기능입니다.

**예시**:
- "비트코인이 60,000,000원을 넘으면 알려줘"
- "RSI가 30 이하로 떨어지면 알려줘"
- "골든크로스가 발생하면 알려줘"

### Alert의 종류

TradingView는 다음 조건으로 Alert을 설정할 수 있습니다:

1. **가격 조건**: 가격이 특정 값에 도달/돌파
2. **지표 조건**: 지표 값이 특정 조건 충족 (RSI, MACD 등)
3. **차트 패턴**: 추세선/수평선 돌파
4. **사용자 지정 스크립트**: Pine Script로 작성한 조건

### Alert 알림 수단

| 알림 수단 | 설명 | 무료 플랜 | Plus 이상 |
|-----------|------|-----------|-----------|
| 팝업 | 브라우저 팝업창 | O | O |
| 이메일 | 등록 이메일로 전송 | O | O |
| 모바일 앱 | TradingView 앱 푸시 | O | O |
| SMS | 휴대폰 문자 | X | O |
| **Webhook URL** | **외부 서버로 HTTP POST** | **X** | **O** |

> ⚠️ **중요**
> **Webhook은 Plus 플랜 이상**에서만 사용 가능합니다.
> 자동매매 시스템 구축을 위해서는 **반드시 Plus 이상**으로 업그레이드해야 합니다.

---

## 7.2 Webhook이란?

### Webhook의 개념

**Webhook**은 한 서비스에서 이벤트가 발생했을 때, **다른 서버의 URL로 HTTP 요청을 자동 전송**하는 방식입니다.

**비유**:
> 친구에게 "나한테 무슨 일이 생기면 이 전화번호로 바로 전화해줘"라고 부탁하는 것과 비슷합니다.
> TradingView에서 매매 신호가 발생하면, **우리 서버의 URL로 자동으로 전화(HTTP POST)**를 거는 방식입니다.

### Webhook 작동 원리

```
┌──────────────┐                    ┌──────────────┐
│  TradingView  │                    │  우리 서버     │
│              │                    │  (FastAPI)    │
│  조건 감시     │                    │               │
│  RSI < 30    │                    │  /webhook     │
└──────┬───────┘                    └──────┬───────┘
       │                                   │
       │   조건 충족!                       │
       │                                   │
       │   HTTP POST                       │
       │   ────────────────────────────→  │
       │   {                               │
       │     "symbol": "BTCUSDT",          │
       │     "action": "BUY",              │
       │     "price": 60000000             │
       │   }                               │
       │                                   │
       │                          ┌────────▼────────┐
       │                          │   업비트 API      │
       │                          │   매수 주문 실행   │
       │                          └─────────────────┘
```

### Webhook의 장점

1. **실시간성**: 조건 충족 즉시 전송 (지연 없음)
2. **자동화**: 사람이 개입하지 않음
3. **확장성**: 여러 조건, 여러 종목을 동시에 처리
4. **유연성**: 원하는 형식의 JSON 메시지 전송 가능

---

## 7.3 TradingView에서 Alert 설정하기

### 가격 Alert 기본 설정

1. 차트 우측 상단의 **시계 모양 아이콘** 클릭 (또는 Alt+A)
2. **"알람 만들기(Create Alert)"** 창이 열림
3. 다음 항목 설정:
   - **조건(Condition)**: 예) "Symbol" → "Price" → "Greater Than" → "60000000"
   - **옵션(Options)**:
     - Only Once: 한 번만 알림
     - Once Per Bar: 캔들당 한 번
     - Once Per Bar Close: 캔들 마감 시 한 번 (권장)
   - **유효 기간(Expiration)**: 무료는 최대 1일, 유료는 무기한
4. **"알림(Notifications)"** 탭에서 알림 수단 선택
5. **"Webhook URL"** 체크하고 URL 입력
6. **"Message"** 박스에 전송할 JSON 작성
7. **"생성(Create)"** 클릭

### 지표 조건 Alert 설정

1. 차트에 원하는 지표 추가 (예: RSI)
2. 지표 위에서 **우클릭** → **"지표에 대한 알림 추가"**
3. 조건 설정: 예) "RSI Crossing Down 30"
4. 이후 단계는 가격 Alert과 동일

---

## 7.4 Alert 메시지 JSON 포맷 설계

### JSON이란?

**JSON(JavaScript Object Notation)**은 데이터를 저장/전송하는 표준 형식입니다.
우리 서버는 TradingView가 보내는 JSON을 받아 파싱하여 매매를 실행합니다.

### 권장 JSON 포맷

```json
{
  "secret": "your-secret-token-here",
  "strategy": "golden_cross",
  "symbol": "BTCUSDT",
  "exchange": "UPBIT",
  "action": "BUY",
  "price": 60000000,
  "timestamp": "{{time}}",
  "quantity_pct": 10
}
```

### 각 필드 설명

| 필드 | 타입 | 설명 |
|------|------|------|
| `secret` | string | 보안 토큰 (위조 방지) |
| `strategy` | string | 전략 이름 (로깅/구분용) |
| `symbol` | string | 종목 심볼 (예: "BTCUSDT", "005930") |
| `exchange` | string | 거래소 ("UPBIT" 또는 "KIS") |
| `action` | string | "BUY", "SELL", "CLOSE" 중 하나 |
| `price` | number | 현재 가격 (참고용) |
| `timestamp` | string | TradingView 시간 변수 |
| `quantity_pct` | number | 자산 대비 매수 비율 (%) |

### TradingView 내장 변수

Alert 메시지에서 사용할 수 있는 변수:

| 변수 | 설명 | 예시 |
|------|------|------|
| `{{ticker}}` | 종목 심볼 | BTCUSDT |
| `{{close}}` | 현재 종가 | 60000000 |
| `{{time}}` | 현재 시간 | 2024-01-15T10:30:00Z |
| `{{exchange}}` | 거래소 | UPBIT |
| `{{interval}}` | 시간프레임 | 15 |
| `{{volume}}` | 거래량 | 125000 |
| `{{strategy.order.action}}` | 전략 주문 방향 | buy / sell |

### 실전 Alert JSON 예시

```json
{
  "secret": "my-super-secret-2024",
  "strategy": "RSI_BB_Strategy",
  "symbol": "{{ticker}}",
  "exchange": "{{exchange}}",
  "action": "BUY",
  "price": {{close}},
  "timestamp": "{{time}}",
  "quantity_pct": 20
}
```

---

## 7.5 Pine Script로 매매 신호 Alert 생성

### Pine Script란?

**Pine Script**는 TradingView 전용 프로그래밍 언어입니다.
사용자 정의 지표나 전략을 작성하고, Alert 조건을 코드로 정의할 수 있습니다.

### Pine Script 에디터 열기

1. TradingView 차트 하단의 **"Pine Editor"** 탭 클릭
2. **"새 빈 스크립트"** 또는 **"전략 시작"** 선택

### 예제 1: 골든크로스 Alert Pine Script

```pinescript
//@version=5
indicator("Golden Cross Alert", overlay=true)

// 이동평균선 계산
shortLen = input.int(5, "Short MA")
longLen = input.int(20, "Long MA")
sma5 = ta.sma(close, shortLen)
sma20 = ta.sma(close, longLen)

// 화면에 표시
plot(sma5, color=color.yellow, linewidth=2, title="SMA 5")
plot(sma20, color=color.blue, linewidth=2, title="SMA 20")

// 교차 감지
goldenCross = ta.crossover(sma5, sma20)
deadCross = ta.crossunder(sma5, sma20)

// 차트에 마커 표시
plotshape(goldenCross, title="Golden Cross",
          location=location.belowbar, color=color.green,
          style=shape.labelup, text="BUY")
plotshape(deadCross, title="Dead Cross",
          location=location.abovebar, color=color.red,
          style=shape.labeldown, text="SELL")

// Alert 조건
alertcondition(goldenCross, title="Golden Cross Buy",
               message='{"secret":"my-secret","strategy":"golden_cross","symbol":"{{ticker}}","exchange":"{{exchange}}","action":"BUY","price":{{close}},"timestamp":"{{time}}","quantity_pct":20}')
alertcondition(deadCross, title="Dead Cross Sell",
               message='{"secret":"my-secret","strategy":"golden_cross","symbol":"{{ticker}}","exchange":"{{exchange}}","action":"SELL","price":{{close}},"timestamp":"{{time}}","quantity_pct":100}')
```

### 예제 2: RSI + 볼린저 밴드 Alert Pine Script

```pinescript
//@version=5
indicator("RSI + BB Strategy", overlay=false)

// RSI 계산
rsiLen = input.int(14, "RSI Length")
rsi = ta.rsi(close, rsiLen)

// 볼린저 밴드 계산
bbLen = input.int(20, "BB Length")
bbMul = input.float(2.0, "BB StdDev")
bbBasis = ta.sma(close, bbLen)
bbDev = bbMul * ta.stdev(close, bbLen)
bbUpper = bbBasis + bbDev
bbLower = bbBasis - bbDev

// 매수/매도 조건
buyCondition = rsi < 30 and close < bbLower
sellCondition = rsi > 70 and close > bbUpper

// RSI 플로팅
plot(rsi, color=color.purple, title="RSI")
hline(30, "Oversold", color=color.green)
hline(70, "Overbought", color=color.red)

// 신호 표시
bgcolor(buyCondition ? color.new(color.green, 80) : na)
bgcolor(sellCondition ? color.new(color.red, 80) : na)

// Alert
alertcondition(buyCondition, title="RSI+BB Buy",
     message='{"secret":"my-secret","strategy":"rsi_bb","symbol":"{{ticker}}","exchange":"{{exchange}}","action":"BUY","price":{{close}},"quantity_pct":15}')
alertcondition(sellCondition, title="RSI+BB Sell",
     message='{"secret":"my-secret","strategy":"rsi_bb","symbol":"{{ticker}}","exchange":"{{exchange}}","action":"SELL","price":{{close}},"quantity_pct":100}')
```

### Pine Script 적용 단계

1. Pine Editor에 코드 붙여넣기
2. **"저장"** (이름 지정)
3. **"차트에 추가(Add to chart)"** 클릭
4. 차트에 지표가 표시되는지 확인
5. Alert 설정 시 **"Condition"**에서 추가한 지표 선택
6. **"Alert() function calls only"** 또는 조건별 선택
7. **Webhook URL** 입력 및 활성화

---

## 7.6 FastAPI Webhook 수신 서버

이제 TradingView가 보내는 Webhook을 받아 처리하는 FastAPI 서버를 구축합니다.

### 7.6.1 필요 라이브러리 설치

```bash
# uv로 설치 (권장)
uv pip install fastapi uvicorn pydantic pydantic-settings sqlalchemy python-dotenv

# 또는 pip
pip install fastapi uvicorn pydantic pydantic-settings sqlalchemy python-dotenv
```

### 7.6.2 프로젝트 구조 (7장용 최소 구성)

```
webhook_server/
├── main.py              # FastAPI 진입점
├── schemas.py           # Pydantic 스키마 (Alert 파싱)
├── models.py            # SQLAlchemy 모델 (DB)
├── database.py          # DB 연결
├── security.py          # Secret Token 검증
└── .env                 # 환경변수
```

### 7.6.3 Pydantic 스키마 정의 (`schemas.py`)

```python
"""
TradingView Alert Webhook을 위한 Pydantic 스키마
"""
from __future__ import annotations

from datetime import datetime
from typing import Literal, Optional

from pydantic import BaseModel, Field


class TradingViewAlert(BaseModel):
    """TradingView에서 전송되는 Alert 메시지 구조"""

    secret: str = Field(..., description="보안 토큰")
    strategy: str = Field(..., description="전략 이름")
    symbol: str = Field(..., description="종목 심볼 (예: BTCUSDT)")
    exchange: Literal["UPBIT", "KIS", "BINANCE"] = Field(..., description="거래소")
    action: Literal["BUY", "SELL", "CLOSE"] = Field(..., description="매매 동작")
    price: float = Field(..., gt=0, description="현재 가격")
    timestamp: Optional[str] = Field(None, description="신호 발생 시간")
    quantity_pct: float = Field(
        default=10.0,
        ge=0.0,
        le=100.0,
        description="자산 대비 매수 비율 (0~100)",
    )

    class Config:
        json_schema_extra = {
            "example": {
                "secret": "my-secret-token",
                "strategy": "golden_cross",
                "symbol": "KRW-BTC",
                "exchange": "UPBIT",
                "action": "BUY",
                "price": 60000000,
                "timestamp": "2024-01-15T10:30:00Z",
                "quantity_pct": 20,
            }
        }


class AlertResponse(BaseModel):
    """Webhook 응답"""

    status: Literal["success", "error", "ignored"]
    message: str
    alert_id: Optional[int] = None
    received_at: datetime
```

### 7.6.4 SQLAlchemy 모델 (`models.py`)

```python
"""
Alert 로그를 저장하기 위한 DB 모델
"""
from __future__ import annotations

from datetime import datetime

from sqlalchemy import Column, DateTime, Float, Integer, String
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass


class AlertLog(Base):
    """수신한 Alert의 로그 테이블"""

    __tablename__ = "alert_logs"

    id: int = Column(Integer, primary_key=True, autoincrement=True)
    strategy: str = Column(String(50), nullable=False)
    symbol: str = Column(String(20), nullable=False)
    exchange: str = Column(String(20), nullable=False)
    action: str = Column(String(10), nullable=False)
    price: float = Column(Float, nullable=False)
    quantity_pct: float = Column(Float, default=10.0)
    received_at: datetime = Column(DateTime, default=datetime.utcnow, nullable=False)
    processed: bool = Column(Integer, default=0)

    def __repr__(self) -> str:
        return (
            f"<AlertLog(id={self.id}, strategy={self.strategy}, "
            f"symbol={self.symbol}, action={self.action})>"
        )
```

### 7.6.5 데이터베이스 연결 (`database.py`)

```python
"""
SQLAlchemy DB 연결 설정
"""
from __future__ import annotations

from collections.abc import Generator

from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

from models import Base

DATABASE_URL = "sqlite:///./webhook.db"

engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False},
    echo=False,
)

SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False)


def init_db() -> None:
    """테이블 생성"""
    Base.metadata.create_all(bind=engine)


def get_db() -> Generator[Session, None, None]:
    """DB 세션 의존성 주입용 제너레이터"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### 7.6.6 보안 검증 (`security.py`)

```python
"""
Webhook 요청의 Secret Token 검증
"""
from __future__ import annotations

import hmac
import os

from dotenv import load_dotenv

load_dotenv()

WEBHOOK_SECRET: str = os.getenv("WEBHOOK_SECRET", "change-this-secret")


def verify_secret(received_secret: str) -> bool:
    """
    받은 secret 토큰이 환경변수의 secret과 일치하는지 검증

    타이밍 공격(timing attack)을 방지하기 위해 hmac.compare_digest 사용

    Args:
        received_secret: 요청에 포함된 토큰

    Returns:
        일치 여부
    """
    return hmac.compare_digest(received_secret, WEBHOOK_SECRET)
```

### 7.6.7 FastAPI 메인 서버 (`main.py`)

```python
"""
TradingView Webhook 수신 FastAPI 서버
"""
from __future__ import annotations

from contextlib import asynccontextmanager
from datetime import datetime, timedelta

from fastapi import Depends, FastAPI, HTTPException, status
from sqlalchemy.orm import Session

from database import SessionLocal, get_db, init_db
from models import AlertLog
from schemas import AlertResponse, TradingViewAlert
from security import verify_secret


@asynccontextmanager
async def lifespan(app: FastAPI):
    """앱 시작 시 DB 초기화"""
    init_db()
    yield


app = FastAPI(
    title="TradingView Webhook Receiver",
    description="TradingView Alert 수신 및 매매 실행 서버",
    version="1.0.0",
    lifespan=lifespan,
)


@app.get("/")
async def root() -> dict[str, str]:
    """헬스체크 엔드포인트"""
    return {"status": "ok", "service": "TradingView Webhook Receiver"}


def is_duplicate_alert(
    db: Session,
    alert: TradingViewAlert,
    window_seconds: int = 60,
) -> bool:
    """
    중복 신호 확인 (같은 심볼/전략/액션이 60초 이내 발생했는지)

    Args:
        db: DB 세션
        alert: 수신한 Alert
        window_seconds: 중복 체크 윈도우 (초)

    Returns:
        중복 여부
    """
    cutoff = datetime.utcnow() - timedelta(seconds=window_seconds)
    existing = (
        db.query(AlertLog)
        .filter(
            AlertLog.strategy == alert.strategy,
            AlertLog.symbol == alert.symbol,
            AlertLog.action == alert.action,
            AlertLog.received_at >= cutoff,
        )
        .first()
    )
    return existing is not None


@app.post("/webhook", response_model=AlertResponse)
async def receive_webhook(
    alert: TradingViewAlert,
    db: Session = Depends(get_db),
) -> AlertResponse:
    """
    TradingView Webhook 수신 엔드포인트

    처리 흐름:
    1. Secret 토큰 검증
    2. 중복 신호 체크
    3. DB에 로그 저장
    4. (다음 장에서 실제 매매 실행 연결)
    """
    # 1. Secret 토큰 검증
    if not verify_secret(alert.secret):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid secret token",
        )

    # 2. 중복 신호 체크
    if is_duplicate_alert(db, alert):
        return AlertResponse(
            status="ignored",
            message="Duplicate signal within 60 seconds",
            received_at=datetime.utcnow(),
        )

    # 3. DB에 저장
    log = AlertLog(
        strategy=alert.strategy,
        symbol=alert.symbol,
        exchange=alert.exchange,
        action=alert.action,
        price=alert.price,
        quantity_pct=alert.quantity_pct,
    )
    db.add(log)
    db.commit()
    db.refresh(log)

    print(
        f"[ALERT] {alert.strategy} | {alert.symbol} | "
        f"{alert.action} @ {alert.price:,.0f} ({alert.exchange})"
    )

    # 4. TODO: 거래소 API를 호출하여 실제 매매 실행 (8, 9장에서 구현)

    return AlertResponse(
        status="success",
        message=f"Alert received: {alert.action} {alert.symbol}",
        alert_id=log.id,
        received_at=datetime.utcnow(),
    )


@app.get("/alerts/recent")
async def get_recent_alerts(
    limit: int = 20,
    db: Session = Depends(get_db),
) -> list[dict]:
    """최근 수신한 Alert 목록 조회"""
    logs = (
        db.query(AlertLog)
        .order_by(AlertLog.received_at.desc())
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
            "price": log.price,
            "received_at": log.received_at.isoformat(),
        }
        for log in logs
    ]
```

### 7.6.8 환경변수 설정 (`.env`)

```bash
# Webhook 보안 토큰 (반드시 변경!)
WEBHOOK_SECRET=your-super-secret-token-here-change-me

# 데이터베이스
DATABASE_URL=sqlite:///./webhook.db
```

### 7.6.9 서버 실행

```bash
# Uvicorn으로 서버 실행
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

서버가 실행되면:
- **API 문서**: http://localhost:8000/docs
- **Webhook URL**: http://your-server.com/webhook
- **최근 알림**: http://localhost:8000/alerts/recent

### 7.6.10 테스트: curl로 Webhook 전송

```bash
curl -X POST http://localhost:8000/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "secret": "your-super-secret-token-here-change-me",
    "strategy": "golden_cross",
    "symbol": "KRW-BTC",
    "exchange": "UPBIT",
    "action": "BUY",
    "price": 60000000,
    "quantity_pct": 20
  }'
```

**예상 응답**:
```json
{
  "status": "success",
  "message": "Alert received: BUY KRW-BTC",
  "alert_id": 1,
  "received_at": "2024-01-15T10:30:45.123456"
}
```

---

## 7.7 외부에서 접근 가능하게 만들기

### 문제: localhost는 TradingView가 접근 불가

개발 단계에서는 `localhost:8000`으로 테스트하지만,
TradingView는 **인터넷에서 접근 가능한 URL**이 필요합니다.

### 해결책 1: ngrok (개발/테스트용)

```bash
# ngrok 설치 후
ngrok http 8000
```

결과:
```
Forwarding  https://abc123.ngrok.io -> http://localhost:8000
```

이 HTTPS URL을 TradingView Webhook URL에 입력:
```
https://abc123.ngrok.io/webhook
```

### 해결책 2: 실서버 배포 (실운영용)

- **Oracle Cloud Free Tier** (무료)
- **AWS EC2**
- **Vultr / DigitalOcean** ($5/월~)

자세한 배포는 **12장**에서 다룹니다.

> ⚠️ **보안 주의사항**
> 1. `WEBHOOK_SECRET`은 **복잡하고 긴 랜덤 문자열** 사용
> 2. **HTTPS 필수** (HTTP는 패킷 탈취 위험)
> 3. **방화벽 설정**: 8000 포트만 열기
> 4. **IP 화이트리스트**: TradingView 공식 IP만 허용 (권장)

TradingView Webhook IP 목록 (공식):
- 52.89.214.238
- 34.212.75.30
- 54.218.53.128
- 52.32.178.7

---

## 7.8 7장 요약

- **Alert**은 조건 충족 시 알림을 발생시키는 TradingView 기능입니다.
- **Webhook**은 Alert이 발생했을 때 외부 서버로 HTTP POST를 보내는 방식입니다.
- Webhook은 **Plus 플랜 이상**에서 사용 가능합니다.
- **Pine Script**로 커스텀 지표/전략과 Alert 조건을 코드로 작성합니다.
- **FastAPI**로 Webhook을 수신하는 서버를 만들고, **Pydantic**으로 JSON을 파싱합니다.
- **Secret Token** 검증과 **중복 신호 방지** 로직은 필수입니다.
- 실운영 시에는 **HTTPS + 방화벽 + IP 화이트리스트**로 보안을 강화해야 합니다.

---

> 📖 **다음 장**: [8장. 거래소 API 연동](./08_거래소_API_연동.md)
