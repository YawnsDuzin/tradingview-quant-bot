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

> 📖 **다음**: [8-2에서는 한국투자증권 KIS API를 학습합니다]
