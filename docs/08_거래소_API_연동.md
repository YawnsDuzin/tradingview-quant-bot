# 8장. 거래소 API 연동

> 이제 실제 매매를 실행할 거래소 API를 연동합니다.
> **암호화폐는 업비트**, **국내 주식은 한국투자증권 KIS Developers**를 사용합니다.

---

## 8.0 왜 이 두 거래소를 선택했는가

### 암호화폐: 업비트

- 국내 **최대 거래량**의 암호화폐 거래소
- **원화(KRW) 직접 거래** 가능
- **Open API 공식 제공** (https://upbit.com/service_center/open_api_guide)
- Python 전용 **pyupbit 라이브러리**로 매우 쉬운 연동
- REST API + WebSocket 지원

### 국내 주식: 한국투자증권 KIS Developers

- 한국투자증권에서 공식 제공하는 개발자 API
- **공식 포탈**: https://apiportal.koreainvestment.com
- **모의투자 환경** 제공 → 실전 전 안전한 테스트 가능
- **국내주식, 해외주식, 선물/옵션** 모두 지원
- REST API + WebSocket (실시간 체결) 지원

### ⚠️ 토스증권은 사용 불가

| 항목 | 설명 |
|------|------|
| **이유** | 토스증권은 **외부 개발자용 자동매매 API를 제공하지 않음** |
| **현황** | 2024년 현재, 토스증권 앱 내에서만 거래 가능 |
| **대안** | **KIS Developers**가 사실상 국내 주식 자동매매의 유일한 공식 API |
| **참고** | 키움증권 OpenAPI는 32bit Windows 전용이라 서버 배포가 어렵고, 한투 API가 현대적 REST 방식으로 가장 편리 |

---

## 8-1. 업비트 Open API (암호화폐)

### 8-1.1 업비트 계정 생성

자동매매를 하려면 먼저 업비트 계정이 필요합니다.

**단계별 계정 생성**:

1. **업비트 웹사이트** 접속: https://upbit.com
2. 우측 상단 **"회원가입"** 클릭
3. **카카오 계정** 연동 (업비트는 카카오 로그인만 지원)
4. **이메일 및 휴대폰 인증**
5. **신원 확인(KYC)**:
   - 신분증 촬영
   - 본인 얼굴 촬영
   - 주소 확인
6. **케이뱅크 계좌 연결** (업비트는 케이뱅크 실명계좌만 사용 가능)
7. **계좌 인증** 완료 → 원화 입금 가능

> ⚠️ **주의**
> 계정 생성과 인증에는 **1~3일** 소요될 수 있습니다.
> 미리 준비하세요.

### 8-1.2 업비트 API 키 발급

API 키는 **내 계정을 프로그램이 사용할 수 있도록 허가**하는 키입니다.

**단계별 발급 절차**:

1. 업비트 로그인 후, 우측 상단 **프로필** → **"Open API 관리"** 클릭
2. **"Open API 사용하기"** 버튼 클릭
3. **약관 동의** 후 진행
4. **사용 목적** 선택 (퀀트 매매, 시세 조회 등)
5. **권한 설정** (매우 중요!):
   - ☑️ **자산 조회**: 필수
   - ☑️ **주문 조회**: 필수
   - ☑️ **주문 하기**: 매매에 필수
   - ☐ **출금 하기**: **절대 체크하지 마세요!** (해킹 시 자금 탈취 위험)
6. **IP 주소 등록**: 서버 IP 입력 (화이트리스트 보안)
7. **2차 인증** (카카오페이/PASS)
8. **Access Key**와 **Secret Key**가 발급됨

> ⚠️ **보안 경고**
> - **Secret Key는 다시 볼 수 없습니다**. 즉시 안전한 곳에 복사/저장하세요.
> - **출금 권한은 절대 활성화하지 마세요.** (해킹 피해 예방)
> - **IP 제한은 반드시 설정하세요.**
> - 키는 **환경변수(.env)**로 관리하고, 절대 **Git에 커밋하지 마세요.**

### 8-1.3 pyupbit 라이브러리 설치

```bash
# 권장: uv
uv pip install pyupbit

# 또는 pip
pip install pyupbit
```

### 8-1.4 환경변수 설정 (`.env`)

```bash
# 업비트 API 키 (절대 Git에 커밋하지 말 것)
UPBIT_ACCESS_KEY=your-access-key-here
UPBIT_SECRET_KEY=your-secret-key-here
```

### 8-1.5 pyupbit 인증 및 기본 사용법

```python
"""
업비트 API 기본 사용 예제
"""
from __future__ import annotations

import os

import pyupbit
from dotenv import load_dotenv

load_dotenv()

ACCESS_KEY: str = os.getenv("UPBIT_ACCESS_KEY", "")
SECRET_KEY: str = os.getenv("UPBIT_SECRET_KEY", "")

# Upbit 객체 생성 (인증 필요한 API용)
upbit = pyupbit.Upbit(ACCESS_KEY, SECRET_KEY)


def test_connection() -> None:
    """연결 테스트: 잔고 조회"""
    balances = upbit.get_balances()
    print("=== 내 잔고 ===")
    for b in balances:
        print(f"{b['currency']}: {b['balance']}")
```

### 8-1.6 잔고 조회

```python
"""
업비트 잔고 조회
"""
from __future__ import annotations

import pyupbit


def get_krw_balance(upbit: pyupbit.Upbit) -> float:
    """
    원화(KRW) 잔고 조회

    Args:
        upbit: 인증된 Upbit 객체

    Returns:
        보유 원화 (사용 가능 잔고)
    """
    balance = upbit.get_balance("KRW")
    return float(balance) if balance else 0.0


def get_coin_balance(upbit: pyupbit.Upbit, ticker: str) -> float:
    """
    특정 코인의 보유 수량 조회

    Args:
        upbit: 인증된 Upbit 객체
        ticker: 종목 심볼 (예: "KRW-BTC")

    Returns:
        보유 수량
    """
    balance = upbit.get_balance(ticker)
    return float(balance) if balance else 0.0


def get_all_balances(upbit: pyupbit.Upbit) -> list[dict]:
    """
    전체 자산 조회 (상세 정보)

    Returns:
        [{"currency": "KRW", "balance": "100000", "avg_buy_price": "0"}, ...]
    """
    return upbit.get_balances()


def calculate_total_asset_krw(upbit: pyupbit.Upbit) -> float:
    """
    총 자산을 원화 기준으로 계산

    Returns:
        총 자산 (KRW)
    """
    balances = upbit.get_balances()
    total_krw = 0.0

    for b in balances:
        currency = b["currency"]
        balance = float(b["balance"])
        if currency == "KRW":
            total_krw += balance
        else:
            ticker = f"KRW-{currency}"
            current_price = pyupbit.get_current_price(ticker)
            if current_price:
                total_krw += balance * current_price

    return total_krw
```

### 8-1.7 현재가 및 캔들 데이터 조회

```python
"""
시세 및 캔들 데이터 조회 (인증 불필요)
"""
from __future__ import annotations

import pandas as pd
import pyupbit


def get_current_price(ticker: str = "KRW-BTC") -> float:
    """
    현재가 조회

    Args:
        ticker: 종목 심볼

    Returns:
        현재 가격
    """
    price = pyupbit.get_current_price(ticker)
    return float(price) if price else 0.0


def get_multiple_prices(tickers: list[str]) -> dict[str, float]:
    """
    여러 종목의 현재가를 한 번에 조회

    Args:
        tickers: 종목 심볼 리스트

    Returns:
        {"KRW-BTC": 60000000, "KRW-ETH": 3500000, ...}
    """
    prices = pyupbit.get_current_price(tickers)
    return {ticker: float(price) for ticker, price in prices.items()}


def get_ohlcv_data(
    ticker: str = "KRW-BTC",
    interval: str = "day",
    count: int = 200,
) -> pd.DataFrame:
    """
    OHLCV 캔들 데이터 조회

    Args:
        ticker: 종목 심볼
        interval: "minute1", "minute3", "minute5", "minute15",
                  "minute30", "minute60", "minute240",
                  "day", "week", "month"
        count: 캔들 개수 (최대 200)

    Returns:
        OHLCV DataFrame
    """
    df = pyupbit.get_ohlcv(ticker, interval=interval, count=count)
    return df


def get_orderbook(ticker: str = "KRW-BTC") -> dict:
    """
    호가 정보 조회

    Args:
        ticker: 종목 심볼

    Returns:
        호가 정보 딕셔너리
    """
    return pyupbit.get_orderbook(ticker)


def get_tickers_krw() -> list[str]:
    """
    KRW 마켓의 모든 종목 심볼 조회

    Returns:
        ["KRW-BTC", "KRW-ETH", ...]
    """
    return pyupbit.get_tickers(fiat="KRW")
```

### 8-1.8 주문 실행 (매수/매도)

```python
"""
업비트 주문 실행 함수들
"""
from __future__ import annotations

from typing import Any

import pyupbit


def market_buy_by_krw(
    upbit: pyupbit.Upbit,
    ticker: str,
    krw_amount: float,
) -> dict[str, Any]:
    """
    시장가 매수 (원화 금액 기준)

    Args:
        upbit: 인증된 Upbit 객체
        ticker: 종목 심볼 (예: "KRW-BTC")
        krw_amount: 매수할 원화 금액 (최소 5,000원)

    Returns:
        주문 결과 딕셔너리

    Example:
        >>> market_buy_by_krw(upbit, "KRW-BTC", 10000)
        {'uuid': '...', 'side': 'bid', 'ord_type': 'price', ...}
    """
    if krw_amount < 5000:
        raise ValueError("업비트 최소 주문 금액은 5,000원입니다.")

    # pyupbit.buy_market_order는 수수료를 고려하여 0.9995 곱함
    result = upbit.buy_market_order(ticker, krw_amount * 0.9995)
    return result


def limit_buy(
    upbit: pyupbit.Upbit,
    ticker: str,
    price: float,
    volume: float,
) -> dict[str, Any]:
    """
    지정가 매수

    Args:
        upbit: 인증된 Upbit 객체
        ticker: 종목 심볼
        price: 매수 희망 가격
        volume: 매수할 수량

    Returns:
        주문 결과
    """
    result = upbit.buy_limit_order(ticker, price, volume)
    return result


def market_sell_all(
    upbit: pyupbit.Upbit,
    ticker: str,
) -> dict[str, Any] | None:
    """
    시장가로 보유 중인 모든 수량 매도

    Args:
        upbit: 인증된 Upbit 객체
        ticker: 종목 심볼 (예: "KRW-BTC")

    Returns:
        주문 결과 또는 None (보유 수량 없음)
    """
    coin = ticker.split("-")[1]
    balance = upbit.get_balance(coin)
    if not balance or float(balance) <= 0:
        print(f"{ticker} 보유 수량이 없습니다.")
        return None

    result = upbit.sell_market_order(ticker, float(balance))
    return result


def market_sell_partial(
    upbit: pyupbit.Upbit,
    ticker: str,
    volume: float,
) -> dict[str, Any]:
    """
    시장가 매도 (수량 지정)

    Args:
        upbit: 인증된 Upbit 객체
        ticker: 종목 심볼
        volume: 매도할 수량

    Returns:
        주문 결과
    """
    result = upbit.sell_market_order(ticker, volume)
    return result


def limit_sell(
    upbit: pyupbit.Upbit,
    ticker: str,
    price: float,
    volume: float,
) -> dict[str, Any]:
    """
    지정가 매도

    Args:
        upbit: 인증된 Upbit 객체
        ticker: 종목 심볼
        price: 매도 희망 가격
        volume: 매도 수량

    Returns:
        주문 결과
    """
    result = upbit.sell_limit_order(ticker, price, volume)
    return result
```

### 8-1.9 주문 내역 조회

```python
"""
업비트 주문 내역 조회
"""
from __future__ import annotations

from typing import Any

import pyupbit


def get_open_orders(
    upbit: pyupbit.Upbit,
    ticker: str | None = None,
) -> list[dict[str, Any]]:
    """
    미체결 주문 목록 조회

    Args:
        upbit: 인증된 Upbit 객체
        ticker: 특정 종목만 조회 (None이면 전체)

    Returns:
        미체결 주문 리스트
    """
    if ticker:
        return upbit.get_order(ticker, state="wait")
    return upbit.get_order("", state="wait")


def get_order_by_uuid(
    upbit: pyupbit.Upbit,
    uuid: str,
) -> dict[str, Any]:
    """
    UUID로 특정 주문 조회

    Args:
        upbit: 인증된 Upbit 객체
        uuid: 주문 UUID

    Returns:
        주문 상세 정보
    """
    return upbit.get_order(uuid)


def cancel_order(
    upbit: pyupbit.Upbit,
    uuid: str,
) -> dict[str, Any]:
    """
    주문 취소

    Args:
        upbit: 인증된 Upbit 객체
        uuid: 취소할 주문 UUID

    Returns:
        취소 결과
    """
    return upbit.cancel_order(uuid)
```

### 8-1.10 업비트 통합 브로커 클래스

위 함수들을 하나의 클래스로 통합한 예제입니다.

```python
"""
업비트 거래 통합 클래스
"""
from __future__ import annotations

import logging
import os
from typing import Any

import pyupbit
from dotenv import load_dotenv

load_dotenv()
logger = logging.getLogger(__name__)


class UpbitBroker:
    """업비트 거래 통합 브로커"""

    MIN_ORDER_KRW = 5000  # 최소 주문 금액

    def __init__(
        self,
        access_key: str | None = None,
        secret_key: str | None = None,
    ) -> None:
        self.access_key = access_key or os.getenv("UPBIT_ACCESS_KEY", "")
        self.secret_key = secret_key or os.getenv("UPBIT_SECRET_KEY", "")

        if not self.access_key or not self.secret_key:
            raise ValueError("UPBIT_ACCESS_KEY, UPBIT_SECRET_KEY가 필요합니다.")

        self.upbit = pyupbit.Upbit(self.access_key, self.secret_key)

    # ---------- 조회 ----------

    def get_krw_balance(self) -> float:
        """원화 잔고 조회"""
        balance = self.upbit.get_balance("KRW")
        return float(balance) if balance else 0.0

    def get_coin_balance(self, ticker: str) -> float:
        """코인 보유 수량 조회"""
        balance = self.upbit.get_balance(ticker)
        return float(balance) if balance else 0.0

    def get_current_price(self, ticker: str) -> float:
        """현재가 조회"""
        price = pyupbit.get_current_price(ticker)
        return float(price) if price else 0.0

    # ---------- 매매 ----------

    def buy_by_pct(self, ticker: str, pct: float) -> dict[str, Any] | None:
        """
        원화 잔고의 일정 비율로 매수

        Args:
            ticker: 종목 심볼 (예: "KRW-BTC")
            pct: 매수 비율 (0.0 ~ 1.0)

        Returns:
            주문 결과 또는 None
        """
        krw = self.get_krw_balance()
        amount = krw * pct

        if amount < self.MIN_ORDER_KRW:
            logger.warning(
                f"매수 금액이 최소 주문 금액보다 작음: {amount:,.0f} KRW"
            )
            return None

        try:
            # 수수료 0.05% 반영 (0.9995)
            result = self.upbit.buy_market_order(ticker, amount * 0.9995)
            logger.info(f"[BUY] {ticker} @ {amount:,.0f} KRW -> {result}")
            return result
        except Exception as e:
            logger.error(f"매수 실패: {e}")
            return None

    def sell_all(self, ticker: str) -> dict[str, Any] | None:
        """보유 전량 매도"""
        coin = ticker.split("-")[1]
        volume = self.get_coin_balance(coin)

        if volume <= 0:
            logger.warning(f"{ticker} 보유 수량 없음")
            return None

        # 최소 매도 금액 체크
        current_price = self.get_current_price(ticker)
        if volume * current_price < self.MIN_ORDER_KRW:
            logger.warning(
                f"매도 금액이 최소 주문 금액보다 작음: {volume * current_price:,.0f}"
            )
            return None

        try:
            result = self.upbit.sell_market_order(ticker, volume)
            logger.info(f"[SELL] {ticker} x {volume} -> {result}")
            return result
        except Exception as e:
            logger.error(f"매도 실패: {e}")
            return None
```

### 8-1.11 사용 예제

```python
"""
UpbitBroker 사용 예제
"""
from upbit_broker import UpbitBroker


def main() -> None:
    broker = UpbitBroker()

    # 1. 잔고 확인
    krw = broker.get_krw_balance()
    print(f"원화 잔고: {krw:,.0f} KRW")

    # 2. 현재가 확인
    btc_price = broker.get_current_price("KRW-BTC")
    print(f"BTC 현재가: {btc_price:,.0f} KRW")

    # 3. 원화의 10%로 BTC 매수 (테스트 시 소액 주의!)
    # result = broker.buy_by_pct("KRW-BTC", 0.1)
    # print(f"매수 결과: {result}")

    # 4. BTC 전량 매도
    # result = broker.sell_all("KRW-BTC")
    # print(f"매도 결과: {result}")


if __name__ == "__main__":
    main()
```

> ⚠️ **테스트 주의사항**
> - 실제 주문 테스트는 **반드시 소액**으로 시작 (5,000원 ~ 10,000원)
> - 업비트는 **모의투자 환경이 없습니다**. 실전과 동일합니다.
> - 매수/매도 함수를 주석 처리 후 조회 함수부터 테스트하세요.

---

## 8-2. 한국투자증권 KIS Developers (국내/해외 주식)

### 8-2.1 KIS Developers 개요

**KIS Developers**는 한국투자증권이 공식 제공하는 **REST API 기반 개발자 포털**입니다.

- **공식 포탈**: https://apiportal.koreainvestment.com
- **지원 자산**: 국내주식, 해외주식, 선물/옵션, ETF
- **인증 방식**: OAuth 2.0 (Access Token)
- **프로토콜**: REST API + WebSocket (실시간)
- **모의투자 환경**: 제공 (학습/테스트에 매우 유용)
- **제공 언어**: 언어 제한 없음 (REST이므로 Python, JavaScript 등 모두 가능)

### 8-2.2 KIS Developers 신청 절차

#### 1단계: 한국투자증권 계좌 개설

KIS API 사용을 위해서는 **한국투자증권의 실계좌가 필수**입니다.

1. **한국투자증권 앱 또는 웹사이트**에서 계좌 개설
2. **비대면 계좌 개설** 가능 (신분증 + 영상통화)
3. 계좌 개설 후 **1영업일 대기** (초기 설정)

#### 2단계: KIS Developers 포털 가입

1. https://apiportal.koreainvestment.com 접속
2. 우측 상단 **"로그인"** → **"회원가입"**
3. 한국투자증권 계정으로 연동 가입
4. 이메일 인증 완료

#### 3단계: 앱 등록 (API 키 발급)

1. 로그인 후 **"My Page"** → **"앱 관리"** 클릭
2. **"앱 등록"** 버튼 클릭
3. 정보 입력:
   - **앱 이름**: 원하는 이름 (예: "quant-bot")
   - **서비스 구분**:
     - **모의투자** (권장, 무료, 가상 자금으로 테스트)
     - **실전투자** (실제 자금, 검증 후 사용)
   - **계좌번호**: 한국투자증권 계좌번호
   - **앱 설명**: 간단한 용도 설명
4. **약관 동의** 후 등록
5. 발급 정보 확인:
   - **APP Key** (공개 키)
   - **APP Secret** (비밀 키, 한 번만 표시됨!)

#### 4단계: 모의투자 환경 신청 (권장)

1. **"모의투자 환경"** 탭
2. 모의투자 **가상 자금 충전** (기본 1억원 제공)
3. 모의투자용 **계좌번호** 확인

> ⚠️ **중요: 모의투자 vs 실전투자**
> - **모의투자**: 가상의 자금으로 실제 API와 동일하게 동작. 학습/테스트에 필수.
> - **실전투자**: 실제 주식 거래. 모의투자에서 충분히 테스트한 후 사용.
> - 두 환경은 **API 엔드포인트 URL이 다릅니다**. 설정 시 주의.

### 8-2.3 KIS API 엔드포인트

| 환경 | Base URL |
|------|----------|
| **모의투자** | `https://openapivts.koreainvestment.com:29443` |
| **실전투자** | `https://openapi.koreainvestment.com:9443` |

### 8-2.4 필요 라이브러리 설치

```bash
uv pip install requests pydantic-settings python-dotenv

# 또는
pip install requests pydantic-settings python-dotenv
```

### 8-2.5 환경변수 설정 (`.env`)

```bash
# KIS Developers (모의투자로 시작)
KIS_MODE=paper  # paper(모의) 또는 real(실전)
KIS_APP_KEY=your-app-key-here
KIS_APP_SECRET=your-app-secret-here
KIS_ACCOUNT_NO=12345678-01  # 계좌번호 (하이픈 포함 형식)
```

### 8-2.6 설정 로더 (`kis_config.py`)

```python
"""
KIS API 설정 로더
"""
from __future__ import annotations

import os
from typing import Literal

from dotenv import load_dotenv

load_dotenv()


class KISConfig:
    """KIS API 설정"""

    # 모드: paper(모의) / real(실전)
    MODE: Literal["paper", "real"] = os.getenv("KIS_MODE", "paper")  # type: ignore

    # API 키
    APP_KEY: str = os.getenv("KIS_APP_KEY", "")
    APP_SECRET: str = os.getenv("KIS_APP_SECRET", "")

    # 계좌번호 (형식: "12345678-01")
    _ACCOUNT_NO: str = os.getenv("KIS_ACCOUNT_NO", "")

    @property
    def base_url(self) -> str:
        """환경별 Base URL"""
        if self.MODE == "real":
            return "https://openapi.koreainvestment.com:9443"
        return "https://openapivts.koreainvestment.com:29443"

    @property
    def cano(self) -> str:
        """계좌번호 앞 8자리 (종합계좌번호)"""
        return self._ACCOUNT_NO.split("-")[0] if "-" in self._ACCOUNT_NO else self._ACCOUNT_NO[:8]

    @property
    def acnt_prdt_cd(self) -> str:
        """계좌 상품 코드 (뒤 2자리)"""
        return self._ACCOUNT_NO.split("-")[1] if "-" in self._ACCOUNT_NO else self._ACCOUNT_NO[8:]


config = KISConfig()
```

### 8-2.7 Access Token 발급 및 갱신

KIS API는 **OAuth 2.0** 방식이라 매 요청마다 Access Token이 필요합니다.
토큰은 **24시간 유효**하며, 만료되면 재발급해야 합니다.

```python
"""
KIS API Access Token 관리
"""
from __future__ import annotations

import json
import os
from datetime import datetime, timedelta
from pathlib import Path

import requests

from kis_config import config

TOKEN_FILE = Path(".kis_token.json")


class KISAuth:
    """KIS API 인증 관리"""

    def __init__(self) -> None:
        self.access_token: str | None = None
        self.expires_at: datetime | None = None
        self._load_cached_token()

    def _load_cached_token(self) -> None:
        """캐시된 토큰 불러오기"""
        if not TOKEN_FILE.exists():
            return

        try:
            data = json.loads(TOKEN_FILE.read_text())
            expires_at = datetime.fromisoformat(data["expires_at"])
            if expires_at > datetime.now() + timedelta(minutes=5):
                self.access_token = data["access_token"]
                self.expires_at = expires_at
        except Exception:
            pass

    def _save_token(self) -> None:
        """토큰을 파일에 저장"""
        if self.access_token and self.expires_at:
            TOKEN_FILE.write_text(
                json.dumps({
                    "access_token": self.access_token,
                    "expires_at": self.expires_at.isoformat(),
                })
            )

    def get_access_token(self, force_refresh: bool = False) -> str:
        """
        Access Token 발급 (캐시된 토큰 활용)

        Args:
            force_refresh: 강제로 재발급

        Returns:
            유효한 Access Token
        """
        # 유효한 토큰이 있으면 재사용
        if (
            not force_refresh
            and self.access_token
            and self.expires_at
            and self.expires_at > datetime.now() + timedelta(minutes=5)
        ):
            return self.access_token

        # 신규 발급
        url = f"{config.base_url}/oauth2/tokenP"
        headers = {"content-type": "application/json"}
        body = {
            "grant_type": "client_credentials",
            "appkey": config.APP_KEY,
            "appsecret": config.APP_SECRET,
        }

        response = requests.post(url, headers=headers, data=json.dumps(body), timeout=10)
        response.raise_for_status()

        data = response.json()
        self.access_token = data["access_token"]
        # 기본 만료 시간: 24시간
        self.expires_at = datetime.now() + timedelta(seconds=data.get("expires_in", 86400))
        self._save_token()

        return self.access_token


kis_auth = KISAuth()
```

### 8-2.8 국내 주식 현재가 조회

```python
"""
KIS API - 국내 주식 시세 조회
"""
from __future__ import annotations

from typing import Any

import requests

from kis_auth import kis_auth
from kis_config import config


def get_stock_price(stock_code: str) -> dict[str, Any]:
    """
    국내 주식 현재가 조회

    Args:
        stock_code: 종목 코드 (예: "005930" = 삼성전자)

    Returns:
        가격 정보 딕셔너리
        {
            "price": 현재가,
            "change": 전일 대비,
            "change_rate": 등락률,
            "volume": 거래량
        }
    """
    url = f"{config.base_url}/uapi/domestic-stock/v1/quotations/inquire-price"
    headers = {
        "content-type": "application/json; charset=utf-8",
        "authorization": f"Bearer {kis_auth.get_access_token()}",
        "appkey": config.APP_KEY,
        "appsecret": config.APP_SECRET,
        "tr_id": "FHKST01010100",
    }
    params = {
        "FID_COND_MRKT_DIV_CODE": "J",  # 주식
        "FID_INPUT_ISCD": stock_code,
    }

    response = requests.get(url, headers=headers, params=params, timeout=10)
    response.raise_for_status()
    data = response.json()

    if data["rt_cd"] != "0":
        raise RuntimeError(f"조회 실패: {data['msg1']}")

    output = data["output"]
    return {
        "price": int(output["stck_prpr"]),
        "change": int(output["prdy_vrss"]),
        "change_rate": float(output["prdy_ctrt"]),
        "volume": int(output["acml_vol"]),
    }


if __name__ == "__main__":
    # 삼성전자 현재가 조회
    price_info = get_stock_price("005930")
    print(f"삼성전자 현재가: {price_info['price']:,}원")
    print(f"전일대비: {price_info['change']:+,}원 ({price_info['change_rate']:+.2f}%)")
```

### 8-2.9 국내 주식 주문

```python
"""
KIS API - 국내 주식 주문
"""
from __future__ import annotations

import json
from typing import Any, Literal

import requests

from kis_auth import kis_auth
from kis_config import config


OrderSide = Literal["BUY", "SELL"]
OrderType = Literal["MARKET", "LIMIT"]


def place_stock_order(
    stock_code: str,
    side: OrderSide,
    quantity: int,
    price: int = 0,
    order_type: OrderType = "MARKET",
) -> dict[str, Any]:
    """
    국내 주식 주문 실행

    Args:
        stock_code: 종목 코드 (6자리)
        side: "BUY" 또는 "SELL"
        quantity: 주문 수량
        price: 주문 가격 (MARKET이면 0)
        order_type: "MARKET"(시장가) 또는 "LIMIT"(지정가)

    Returns:
        주문 결과
    """
    url = f"{config.base_url}/uapi/domestic-stock/v1/trading/order-cash"

    # tr_id는 환경과 매매 방향에 따라 다름
    # 실전 매수: TTTC0802U / 매도: TTTC0801U
    # 모의 매수: VTTC0802U / 매도: VTTC0801U
    prefix = "V" if config.MODE == "paper" else "T"
    if side == "BUY":
        tr_id = f"{prefix}TTC0802U"
    else:
        tr_id = f"{prefix}TTC0801U"

    headers = {
        "content-type": "application/json; charset=utf-8",
        "authorization": f"Bearer {kis_auth.get_access_token()}",
        "appkey": config.APP_KEY,
        "appsecret": config.APP_SECRET,
        "tr_id": tr_id,
    }

    # 주문 구분 코드
    # 01: 시장가, 00: 지정가
    ord_dvsn = "01" if order_type == "MARKET" else "00"

    body = {
        "CANO": config.cano,
        "ACNT_PRDT_CD": config.acnt_prdt_cd,
        "PDNO": stock_code,
        "ORD_DVSN": ord_dvsn,
        "ORD_QTY": str(quantity),
        "ORD_UNPR": str(price) if order_type == "LIMIT" else "0",
    }

    response = requests.post(
        url,
        headers=headers,
        data=json.dumps(body),
        timeout=10,
    )
    response.raise_for_status()
    data = response.json()

    if data["rt_cd"] != "0":
        raise RuntimeError(f"주문 실패: {data['msg1']}")

    return {
        "order_no": data["output"]["ODNO"],
        "order_time": data["output"]["ORD_TMD"],
        "message": data["msg1"],
    }
```

### 8-2.10 잔고 및 보유 종목 조회

```python
"""
KIS API - 잔고 조회
"""
from __future__ import annotations

from typing import Any

import requests

from kis_auth import kis_auth
from kis_config import config


def get_account_balance() -> dict[str, Any]:
    """
    국내 주식 잔고 및 보유 종목 조회

    Returns:
        {
            "cash": 예수금,
            "total_eval": 총평가금액,
            "total_profit_loss": 총손익,
            "holdings": [
                {"code": "005930", "name": "삼성전자",
                 "quantity": 10, "avg_price": 70000,
                 "current_price": 72000, "profit_loss": 20000},
                ...
            ]
        }
    """
    url = f"{config.base_url}/uapi/domestic-stock/v1/trading/inquire-balance"

    prefix = "V" if config.MODE == "paper" else "T"
    tr_id = f"{prefix}TTC8434R"

    headers = {
        "content-type": "application/json; charset=utf-8",
        "authorization": f"Bearer {kis_auth.get_access_token()}",
        "appkey": config.APP_KEY,
        "appsecret": config.APP_SECRET,
        "tr_id": tr_id,
    }
    params = {
        "CANO": config.cano,
        "ACNT_PRDT_CD": config.acnt_prdt_cd,
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

    response = requests.get(url, headers=headers, params=params, timeout=10)
    response.raise_for_status()
    data = response.json()

    if data["rt_cd"] != "0":
        raise RuntimeError(f"잔고 조회 실패: {data['msg1']}")

    # 종합 정보
    summary = data["output2"][0] if data["output2"] else {}

    # 보유 종목
    holdings: list[dict[str, Any]] = []
    for item in data.get("output1", []):
        if int(item["hldg_qty"]) > 0:
            holdings.append({
                "code": item["pdno"],
                "name": item["prdt_name"],
                "quantity": int(item["hldg_qty"]),
                "avg_price": int(item["pchs_avg_pric"]),
                "current_price": int(item["prpr"]),
                "profit_loss": int(item["evlu_pfls_amt"]),
                "profit_rate": float(item["evlu_pfls_rt"]),
            })

    return {
        "cash": int(summary.get("dnca_tot_amt", 0)),
        "total_eval": int(summary.get("tot_evlu_amt", 0)),
        "total_profit_loss": int(summary.get("evlu_pfls_smtl_amt", 0)),
        "holdings": holdings,
    }
```

### 8-2.11 WebSocket 실시간 체결가 수신

```python
"""
KIS API - WebSocket 실시간 체결가 수신
"""
from __future__ import annotations

import asyncio
import json

import websockets

from kis_auth import kis_auth
from kis_config import config


def _get_ws_url() -> str:
    """WebSocket URL"""
    if config.MODE == "real":
        return "ws://ops.koreainvestment.com:21000"
    return "ws://ops.koreainvestment.com:31000"  # 모의투자


async def subscribe_realtime_price(stock_codes: list[str]) -> None:
    """
    국내 주식 실시간 체결가 구독

    Args:
        stock_codes: 구독할 종목 코드 리스트 (예: ["005930", "000660"])
    """
    url = _get_ws_url()

    async with websockets.connect(url, ping_interval=None) as ws:
        # 각 종목별 구독 요청
        for code in stock_codes:
            subscribe_msg = {
                "header": {
                    "approval_key": kis_auth.get_access_token(),
                    "custtype": "P",
                    "tr_type": "1",  # 구독
                    "content-type": "utf-8",
                },
                "body": {
                    "input": {
                        "tr_id": "H0STCNT0",  # 국내주식 실시간체결가
                        "tr_key": code,
                    }
                },
            }
            await ws.send(json.dumps(subscribe_msg))

        print(f"실시간 체결가 구독 시작: {stock_codes}")

        # 데이터 수신 루프
        while True:
            try:
                data = await ws.recv()
                # 실시간 체결 데이터는 파이프(|) 구분자로 전송됨
                if data[0] == "0":  # 실시간 데이터
                    parts = data.split("|")
                    if len(parts) >= 4:
                        _, tr_id, _, payload = parts[:4]
                        fields = payload.split("^")
                        if tr_id == "H0STCNT0" and len(fields) > 2:
                            code = fields[0]
                            time = fields[1]
                            price = fields[2]
                            print(f"[{time}] {code}: {int(price):,}원")
                else:
                    # 제어 메시지 (구독 응답 등)
                    print(f"제어: {data[:200]}")
            except websockets.ConnectionClosed:
                print("연결 종료")
                break


if __name__ == "__main__":
    # 삼성전자, SK하이닉스 실시간 가격 수신
    asyncio.run(subscribe_realtime_price(["005930", "000660"]))
```

### 8-2.12 KIS 통합 브로커 클래스

```python
"""
KIS API 거래 통합 클래스
"""
from __future__ import annotations

import logging
from typing import Any

from kis_auth import kis_auth
from kis_config import config
from kis_orders import place_stock_order
from kis_quotes import get_stock_price
from kis_balance import get_account_balance

logger = logging.getLogger(__name__)


class KISBroker:
    """한국투자증권 KIS API 거래 브로커"""

    def __init__(self) -> None:
        # 토큰 사전 발급
        kis_auth.get_access_token()
        logger.info(f"KIS Broker 초기화 완료 (MODE={config.MODE})")

    def get_price(self, stock_code: str) -> int:
        """현재가 조회"""
        return get_stock_price(stock_code)["price"]

    def get_balance(self) -> dict[str, Any]:
        """잔고 조회"""
        return get_account_balance()

    def get_cash(self) -> int:
        """예수금만 조회"""
        return self.get_balance()["cash"]

    def buy_market(self, stock_code: str, quantity: int) -> dict[str, Any]:
        """시장가 매수"""
        logger.info(f"[BUY] {stock_code} x {quantity}")
        return place_stock_order(
            stock_code=stock_code,
            side="BUY",
            quantity=quantity,
            order_type="MARKET",
        )

    def sell_market(self, stock_code: str, quantity: int) -> dict[str, Any]:
        """시장가 매도"""
        logger.info(f"[SELL] {stock_code} x {quantity}")
        return place_stock_order(
            stock_code=stock_code,
            side="SELL",
            quantity=quantity,
            order_type="MARKET",
        )

    def buy_limit(
        self, stock_code: str, quantity: int, price: int,
    ) -> dict[str, Any]:
        """지정가 매수"""
        logger.info(f"[LIMIT BUY] {stock_code} x {quantity} @ {price}")
        return place_stock_order(
            stock_code=stock_code,
            side="BUY",
            quantity=quantity,
            price=price,
            order_type="LIMIT",
        )

    def sell_limit(
        self, stock_code: str, quantity: int, price: int,
    ) -> dict[str, Any]:
        """지정가 매도"""
        logger.info(f"[LIMIT SELL] {stock_code} x {quantity} @ {price}")
        return place_stock_order(
            stock_code=stock_code,
            side="SELL",
            quantity=quantity,
            price=price,
            order_type="LIMIT",
        )

    def buy_by_krw(
        self, stock_code: str, krw_amount: int,
    ) -> dict[str, Any] | None:
        """
        원화 금액 기준 시장가 매수
        (수량을 자동 계산)

        Args:
            stock_code: 종목 코드
            krw_amount: 매수할 원화 금액

        Returns:
            주문 결과 또는 None
        """
        current_price = self.get_price(stock_code)
        quantity = krw_amount // current_price

        if quantity <= 0:
            logger.warning(
                f"매수 수량이 0주입니다. 가격={current_price}, 금액={krw_amount}"
            )
            return None

        return self.buy_market(stock_code, int(quantity))
```

### 8-2.13 KIS Broker 사용 예제

```python
"""
KIS Broker 사용 예제
"""
from kis_broker import KISBroker


def main() -> None:
    broker = KISBroker()

    # 1. 잔고 조회
    balance = broker.get_balance()
    print(f"예수금: {balance['cash']:,}원")
    print(f"총평가: {balance['total_eval']:,}원")
    print(f"손익:   {balance['total_profit_loss']:+,}원")

    # 2. 삼성전자 현재가
    price = broker.get_price("005930")
    print(f"삼성전자: {price:,}원")

    # 3. 모의투자에서 10주 매수 테스트
    # result = broker.buy_market("005930", 10)
    # print(f"매수 결과: {result}")

    # 4. 10만원어치 매수
    # result = broker.buy_by_krw("005930", 100_000)
    # print(f"매수 결과: {result}")


if __name__ == "__main__":
    main()
```

> ⚠️ **KIS API 주의사항**
> 1. **모의투자 환경에서 충분히 테스트**한 후 실전 전환
> 2. **토큰은 24시간 유효**, 자동 갱신 로직 필수
> 3. **TR_ID가 환경에 따라 다름** (모의: V로 시작, 실전: T로 시작)
> 4. **주문 수량 단위**: 국내 주식은 **1주 단위** (일부 ETF는 다름)
> 5. **장 시간 외 주문 불가**: 평일 09:00 ~ 15:30만 가능
> 6. **API 호출 제한**: 초당 20건 정도로 제한되므로 대량 조회 시 주의

---

## 8.3 8장 요약

- 암호화폐 자동매매에는 **업비트 + pyupbit**을 사용합니다.
- 국내 주식 자동매매에는 **한국투자증권 KIS Developers API**를 사용합니다.
- **토스증권은 외부 자동매매 API를 제공하지 않아** 사용 불가합니다.
- 업비트는 **출금 권한을 절대 활성화하지 마세요**. 해킹 시 자금 탈취 위험이 있습니다.
- KIS는 **모의투자 환경**을 제공하므로, 실전 전에 충분히 테스트할 수 있습니다.
- 두 거래소 모두 **UpbitBroker**, **KISBroker** 클래스로 추상화하여 사용합니다.
- **환경변수(.env)**로 API 키를 관리하고, **절대 Git에 커밋하지 마세요**.

---

> 📖 **다음 장**: [9장. FastAPI 기반 자동매매 시스템 구축](./09_FastAPI_자동매매_시스템.md)
