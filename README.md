# 백테스트 검증 기록

전략 **301개** · 단기 전략 **1개** · 모멘텀 **42조합**
2011 ~ 2026 · 얼마를 벌었는가가 아니라, 어떻게 믿지 않았는가

| | 범위 | 핵심 |
|---|---|---|
| **넓게** | 전략 301개 전수 | 비용 후 **160개**가 손실 |
| **깊게** | 단기 전략 1개 | 체결 가정만 바꿔 **+126.8% → −16.0%**. 운용 규모 수용 한도 **약 4억원** |
| **먼저** | 모멘텀 42조합 | 실행 전 규칙 등록 · **PBO 0.72** |

---

## 0. 결과 먼저

| 검증 | 결과 |
|---|---|
| 비용 반영 후 전체기간 연수익률 음수 | **160 / 301** |
| 비용 2배에서 OOS 샤프 ≤ 0 | **177 / 301** |
| 랜덤 진입 대조군에 패배 | **44 / 81** |
| DSR 95% 초과 | **1개** — 체결 불가 `ON05`. 빼면 **0개** (최고 0.644) |
| 알파 t > 2 | **6개 / 155개** — 우연 기대치 **3.53개**, 포아송 p = 0.15 |
| 알파 t > 3 (Harvey et al. 2016) | **0개** (최고 2.96, 그나마 단일 지수 보유) |
| 최종 1등급 | **4개** — 넷 다 자산배분 계열 |

**보드** → https://hsh2578.github.io/kr-strategy-board/

---

## 1. 데이터

| | |
|---|---|
| 원천 | 2010-01 ~ 2026-09 · **4,364 영업일** |
| 판정 시작 | 2011-01-01 — 앞 1년은 지표 워밍업 |
| 종목 | KOSPI·KOSDAQ 시총 1,000억원 이상 보통주 + ETF 48종 |
| 제외 | 우선주 · 스팩 · 리츠 · 외국주권 |
| 정합성 | 시총 ÷ 종가 = 주식 수 → 분할·감자·증자 탐지 · 키움·한투·KRX 3원 대조 |

가격제한폭이 ±30%이므로 **수정수익률 ±35% 초과는 데이터 오류**로 플래그.

---

## 2. 비용 → 여기서 160개가 죽었다

| 항목 | 값 |
|---|---|
| 편도 수수료 | 0.015% |
| 매도 거래세 | **연도별** 2010년 0.30% → 2026년 0.20% |
| 슬리피지 (전일 시총) | 5조↑ 0.1% / 1~5조 0.2% / 1조↓ **0.5%** |
| ETF 슬리피지 | 0.05% |
| 해외·채권·금·달러 ETF 매매차익 | **15.4% 과세** (연 1회 차감) |

같은 조건으로 전수 **3회** — 기본 · 비용 2배 · 슬리피지 0.

**거래세는 실제 세법과 대조해 맞다.** 코스피는 거래세 + 농어촌특별세 0.15%, 코스닥은 거래세만이고 **합계가 같게 설계**돼 있어 시장을 구분하지 않았다. ETF는 면제. 2026년 0.20%는 금투세 폐지로 환원된 값이다.

**이 비용 모델의 한계는 아래 §15에 적어 두었다.**

<details>
<summary><b>backtest/costs.py</b> — 연도별 거래세 테이블과 시총 구간 슬리피지</summary>

```python
"""한국 주식 거래비용. vectorbt 의 fees / slippage 인자로 그대로 넣을 배열을 만든다.

vectorbt 는 fees 를 매수·매도에 같은 비율로 부과하므로, 매도에만 붙는 거래세는
양쪽에 반씩 나눠 넣는다(왕복 합계는 동일).
"""
COMMISSION = 0.00015  # 편도 수수료 0.015%

# 매도 거래세(농특세 포함, 주식). ETF 는 0.
# 2026: 2025 세법개정으로 0.20% 환원 — 구현 시점 확인값
SELL_TAX = {
    2010: 0.0030, 2011: 0.0030, 2012: 0.0030, 2013: 0.0030, 2014: 0.0030,
    2015: 0.0030, 2016: 0.0030, 2017: 0.0030, 2018: 0.0030, 2019: 0.0030,
    2020: 0.0025, 2021: 0.0023, 2022: 0.0023, 2023: 0.0020, 2024: 0.0018,
    2025: 0.0015, 2026: 0.0020,
}

# 편도 슬리피지: 전일 시총 기준
SLIP_TIERS = [(5e12, 0.001), (1e12, 0.002), (0, 0.005)]
if os.environ.get("BT_SLIP"):  # 민감도 테스트: BT_SLIP="0.0005,0.001,0.002"
    SLIP_TIERS = [(cap, float(v)) for (cap, _), v in zip(SLIP_TIERS, os.environ["BT_SLIP"].split(","))]
ETF_SLIP = 0.0005


def fee_rate(index: pd.DatetimeIndex, etf: bool = False) -> pd.Series:
    """편도 수수료 + 거래세/2 (날짜별)."""
    tax = np.zeros(len(index)) if etf else index.year.map(SELL_TAX).to_numpy(float)
    return pd.Series(COMMISSION + tax / 2, index=index)


def slippage(prev_mktcap: pd.DataFrame) -> pd.DataFrame:
    out = pd.DataFrame(SLIP_TIERS[-1][1], index=prev_mktcap.index, columns=prev_mktcap.columns)
    for cap, s in reversed(SLIP_TIERS[:-1]):  # 작은 구간부터 덮어써서 큰 시총이 최종
        out = out.mask(prev_mktcap >= cap, s)
    return out
```

</details>

---

## 3. 신호가 나도 못 사는 날

- **상한가 매수 차단** — 파는 사람이 없으면 체결되지 않는다
- 가격제한폭 **2015-06-15**에 ±15% → ±30%. 날짜로 기준을 바꿈
- **거래정지** 구간 무체결 — 손절 신호가 떠도 못 판다
- 손절 기준가는 실체결가, 손절 체결에도 슬리피지 부과

<details>
<summary><b>backtest/engine.py</b> — 가격제한폭 날짜 분기</summary>

```python
LIMIT_UP_OPEN = 0.29
LIMIT_CHANGE = pd.Timestamp("2015-06-15")   # 가격제한폭 ±15% → ±30%


def limit_up(index):
    """상한가로 간주하는 상승률(제한폭 - 1%p): 2015-06-15 이전 14%, 이후 29%. 미국은 가격제한 없음."""
    if os.environ.get("BT_MARKET") == "us":
        return pd.Series(10.0, index=index)
    return pd.Series(np.where(index < LIMIT_CHANGE, 0.14, LIMIT_UP_OPEN), index=index)
```

</details>

---

## 4. 생존편향 — 상장폐지 손실을 실측했다

| | |
|---|---|
| 정리매매 **197건** 중앙값 | **−94.2%** (사분위 −98% ~ −80%) |
| 부실 / 합병 구분 | 정리매매 없이 폐지된 **119건** 중 **116건**이 정지 11~20거래일 → 합병·공개매수 |
| 기준 | **60거래일 이상 정지만 부실 폐지** |

> 끝까지 정지 중인 종목은 아직 상장 상태라 청산하지 않는다 — **정지 직전 청산은 미래참조**

<details>
<summary><b>backtest/engine.py</b> — 상장폐지 강제청산</summary>

```python
# 상장폐지 청산(2026-09-25): 정리매매 197건의 '정지 직전가 대비 최종가' 중앙값 -94.2%(사분위 -98% ~ -80%).
# 정리매매 가격 없이 폐지된 119건 중 116건은 정지 11~20거래일(합병·공개매수) → 60거래일 이상만 부실로 본다.
DELIST_HAIRCUT, LONG_HALT = -0.942, 60


def delist_exits(o: pd.DataFrame, c: pd.DataFrame) -> dict:
    """상장폐지 종목 → (청산일, 청산가). 종가가 패널 끝 전에 끝난 종목만 폐지로 본다
    (끝까지 정지 중인 종목은 아직 상장 상태라 청산하지 않는다 — 정지 직전 청산은 미래참조).
    마지막 날 거래(정리매매)가 있으면 그 종가, 없으면 정지 60거래일 이상은 직전가×(1-94.2%), 짧으면(합병 등) 직전가."""
    out, end = {}, c.index[-1]
    for col in c.columns:
        lc = c[col].last_valid_index()
        if lc is None or lc >= end:
            continue
        px = float(c.at[lc, col])
        if lo is None or lo < lc:
            n = c.index.get_loc(lc) - (c.index.get_loc(lo) if lo is not None else 0)
            if n >= LONG_HALT:
                px *= 1 + DELIST_HAIRCUT
        out[col] = (lc, px)
    return out
```

</details>

---

## 5. 미래참조 차단

```
시가총액  CAP.shift(1)   →  순위·슬리피지 모두 전일 기준

장 마감  →  종가로 신호 계산  →  다음날 종가에 체결
```

종가로 계산한 신호를 그 종가에 체결하면 미래참조. 하루 미루면 성과가 나빠진다.

<details>
<summary><b>backtest/engine.py</b> — 신호 정의 규약</summary>

```python
@dataclass
class Spec:
    """전략 신호 정의. 신호는 모두 '그날 장 마감 후 알 수 있는 정보'로 만든다(일봉 인덱스)."""
    entry: pd.DataFrame                  # bool
```

</details>

---

## 6. 구간 분리

| 구간 | 기간 | 역할 |
|---|---|---|
| IS | 2011 ~ 2019 | 전략을 다듬을 때 보는 구간 |
| **OOS** | 2020 ~ | 손대지 않고 남겨둔 시험지 |
| **TO24** | 2011 ~ 2024 | **2025~26 급등장 제외** · 등급 판정 구간 |

OOS만 좋고 IS가 마이너스면 그 구간에만 통한 전략 → **양쪽 다 플러스**를 공통 관문으로.

<details>
<summary><b>backtest/run_all.py</b> — 네 구간 산출</summary>

```python
OOS = pd.Timestamp("2020-01-01")
TAX = 0.154   # 국내상장 과세 ETF 매매차익 배당소득세
STOCK_START = pd.Timestamp("2011-01-01")   # 지표 워밍업 1년


def windows(daily, trades, slots=10):
    out = {}
    for tag, lo, hi in (("ALL", None, None), ("IS", None, OOS - pd.Timedelta(days=1)), ("OOS", OOS, None),
                        ("TO24", None, pd.Timestamp("2024-12-31"))):   # 2025~26 급등장 제외 구간
        dd = daily.loc[lo:hi]
        if len(dd) < 60:
            continue
        ...
    return out
```

</details>

---

## 7. 우연 걸러내기

### 7-1. Deflated Sharpe Ratio

| | |
|---|---|
| 301개 시험 시 우연히 도달하는 초과 샤프 | **0.716** (연율) |
| DSR 95% 초과 | 1개 — 체결 불가 `ON05` |
| **그것을 빼면** | **0개** · 최고 **0.644** (`AA17`, 293개 기준) |

Bailey & López de Prado (2014) · n = 301, T = 4,120일(16.3년)

<details>
<summary><b>backtest/report.py</b> — DSR 구현</summary>

```python
def dsr(daily: pd.DataFrame) -> pd.Series:
    """'진짜 샤프 > 0' 일 확률. 수익률은 단기채 대비 초과, N = daily 의 전략 수."""
    from scipy.stats import kurtosis, norm, skew
    import engine as E
    ex = daily.sub(E.rf_daily(daily.index), axis=0)
    sr = ex.mean() / ex.std()
    n, g = daily.shape[1], 0.5772156649
    emax = (1 - g) * norm.ppf(1 - 1 / n) + g * norm.ppf(1 - 1 / (n * np.e))   # N개 무실력 전략의 최대값 배수
    out = {}
    for c in ex:
        x = ex[c].dropna()
        if len(x) < 60 or x.std() == 0:
            continue
        sr0 = emax / np.sqrt(len(x) - 1)     # 귀무(샤프 0) 추정 분산 1/(T-1)
        z = (sr[c] - sr0) * np.sqrt(len(x) - 1) / np.sqrt(
            1 - skew(x) * sr[c] + (kurtosis(x, fisher=False) - 1) / 4 * sr[c] ** 2)
        out[c] = norm.cdf(z)
    return pd.Series(out)
```

</details>

### 7-2. 알파 t값

**2011~2024 구간에 포지션이 250일 이상 있던 155개**에만 산출한다. 거래가 드문 전략은 표본이 부족해 t값을 신뢰할 수 없으므로 회귀 자체를 하지 않는다.

| | |
|---|---|
| 155개 시험 시 우연 기대치 | t > 2 가 **3.53개** |
| 실제로 나온 것 | **6개** — 기대보다 많으나 **포아송 p = 0.15** 로 유의하지 않음 |
| t > 3 (Harvey et al. 2016) | **0개** · 최고 **2.96** |
| 최고 2.96의 정체 | `H05` **단일 지수 보유** — 전략이 아니다. 전략만 보면 최고 **2.82**(`AA17`), t > 2 는 **5개** |

<details>
<summary><b>backtest/report.py</b> — KODEX200 회귀 알파·베타</summary>

```python
def alpha_beta(daily: pd.DataFrame, end="2024-12-31", start="2011-01-01") -> pd.DataFrame:
    """KODEX200(H01) 초과수익에 대한 회귀: 연율 알파·베타·알파 t값(start~end).
    시장이 오른 덕(베타)을 뺀 나머지."""
    import engine as E
    rf = E.rf_daily(daily.index)
    mk = daily["H01"] - rf
    out = {}
    for c in daily:
        z = pd.concat([daily[c] - rf, mk], axis=1).loc[start:end].dropna()
        if (daily[c].loc[z.index] != 0).sum() < 250:
            continue
        ...
```

</details>

### 7-3. 랜덤 진입 대조군

```
날짜별 진입 수는 그대로 · 종목만 그날 유니버스에서 무작위
```

**44 / 81** — 제비뽑기가 전략보다 나았다.

<details>
<summary><b>backtest/run_all.py</b> — 대조군 생성</summary>

```python
def randomize(spec, seed=0):
    """랜덤 진입 대조군: 날짜별 진입 수는 그대로, 종목만 그날 유니버스에서 무작위."""
    rng = np.random.default_rng(seed)
    ent = spec.entry
    pool = S.d.U.reindex_like(ent).fillna(False).to_numpy()
    n = ent.sum(axis=1).to_numpy()
    out = np.zeros(ent.shape, dtype=bool)
    for i in np.nonzero(n)[0]:
        cand = np.nonzero(pool[i])[0]
        if len(cand):
            out[i, rng.choice(cand, size=min(n[i], len(cand)), replace=False)] = True
    kw = {k: v for k, v in spec.__dict__.items() if k != "entry"}
    return E.Spec(entry=pd.DataFrame(out, index=ent.index, columns=ent.columns), **kw)
```

</details>

---

## 8. 엔진 교차검증 — 내 버그 두 개를 찾았다

backtrader 로 비용·슬리피지·가격제한·거래정지를 **백지에서 재구현** (엔진 코드 `import` 0), 7개 케이스 대조.

| | |
|---|---|
| 체결가 일치 오차 | **2e−16** |
| 청산일 | **100% 일치** |

그런데 버그가 나왔다.

1. 슬롯 경쟁에서 밀린 종목이 종목별 독립 시뮬레이션에서 '보유 중'으로 남아 이후 신호를 막음
2. `min_size`·`init_cash` 설정으로 비싼 종목 주문이 **경고 없이** 무시됨

| | |
|---|---|
| `SW22` 연수익률 | −7.39% → **−4.42%** |
| 수치가 바뀐 전략 | 슬롯형 주식 전략 **134개** |
| 2026-09-30 수정 후 | 전 유니버스 재실행 · **수정 전 수치는 인용하지 않는다** |

기록 → `backtest/VERIFY_ENGINE.md` · `backtest/VERIFY_SIGNALS.md`

---

## 9. 등급 게이트

**먼저 제외** — 단일 지수 보유(`H*`, 전략이 아니라 비교선) · 체결 불가·데이터 결함 8종
**공통 관문** — TO24 연수익 > 0 · **IS 연수익 > 0** · 비용 2배 OOS 샤프 > 0

| 등급 | 조건 | 수 |
|---|---|---|
| **1 검증 통과** | 초과 샤프 ≥ 0.5 **그리고** 알파 t > 2 | **4** |
| 2 경계 | 초과 샤프 ≥ 0.5 (t 미달) | 3 |
| 3 지수보다 나음 | 수익률 또는 샤프가 코스피200 초과 | 23 |
| 4 수익 플러스 | 지수 이하 | 8 |
| 주의 (숨김) | 지수 초과이나 출처 판정 '문제' | 5 |

<details>
<summary><b>backtest/site_build.py</b> — 게이트와 제외 목록</summary>

```python
EXCL = ["ON05", "CD01", "ON07", "SW14", "SW15", "SW16", "LT14", "BM02"]   # 체결 불가·수급 결함·벤치마크


def build_tiers(write=True) -> pd.DataFrame:
    """
    공통: 11~24 연수익 > 0, 앞 구간(IS) 연수익 > 0, 비용 2배 OOS 샤프 > 0. 단일 지수 보유(H*)·EXCL 제외.
    1 검증 통과: 11~24 초과 샤프 ≥ 0.5 & 알파 t > 2 / 2 경계: 샤프 ≥ 0.5 /
    3: 수익률 또는 샤프가 코스피200 초과 / 4: 코스피200 이하 & 샤프 > 0 /
    주의(숨김): 코스피200 초과지만 퀀트 판정 '문제'."""
    for sid in ...:
        if sid in EXCL or sid.startswith("H0") or sid not in d:
            continue
        tier = ("주의(숨김)" if beat else None) if prob else (
            "1 검증 통과" if s >= 0.5 and t > 2 else "2 경계" if s >= 0.5 else ...)
```

</details>

---

## 10. 남은 넷

| ID | 계열 | 연수익 11~24 | 초과 샤프 | 알파 t | 출처 등급 |
|---|---|---|---|---|---|
| `LT32` | 중장기 | 9.13% | 0.724 | 2.637 | **D** 원 규칙 특정 불가 |
| `AA09` | 자산배분 | 9.54% | 0.671 | 2.456 | **C** 미국 신호를 총수익으로 수정 |
| `AA17` | 자산배분 | 8.03% | 0.656 | 2.891 | **D** 출처가 블로그뿐 |
| `AA10` | 자산배분 | 6.18% | 0.512 | 2.279 | **B** 약세 신호가 16년 7회 |
| 코스피200 | *비교선* | *4.7%* | *0.23* | — | — |

**넷 다 종목을 고르는 전략이 아니라 자산을 옮기는 전략.**

출처 등급에 **A가 없다.** 원문 PDF를 확인한 `AA13`(Faber)은 초과 샤프가 약했다.

---

## 11. 조합도 시험했고, 넷 다 실패했다

| 시도 | 결과 |
|---|---|
| 매달 상위 5개 자동선택 | 초과 샤프 0.417 → **코스피200(0.591)에 패배** |
| 국면 판단 4종 | 전부 '국면 없음'보다 낮음 · **Holm 보정 p = 1.0** |
| 전 전략 동일비중 | **−17.8%** · MDD −94.6% |
| 오버레이 9변형 (변동성 타겟·모멘텀 필터·국면 노출) | 기준 OOS 샤프 0.80을 **아무도 못 넘음** |

**손을 더 보태는 쪽은 전부 나빠졌다.**

---

## 12. 한 전략을 끝까지 — 체결 가정만 바꿨다

`ML1L` 머신러닝 오버나잇 상위 5, 시총 1조↑ · 거래 **10,281건**

| 비용 가정 | 연수익률 | 샤프 | IS 구간 |
|---|---|---|---|
| 세금 0 — **모의투자 대회 조건** | **+126.8%** | 3.84 | +70.8% |
| 슬리피지 0 | +88.6% | 2.97 | +25.5% |
| 제곱근 충격 η = 0.5 | +67.9% | 2.44 | +29.4% |
| 제곱근 충격 η = 1 | +39.8% | 1.59 | +4.1% |
| 제곱근 충격 η = 2 | **−3.1%** | −0.12 | −32.5% |
| 정액 슬리피지 0.1~0.2% — 엔진 기본 | **−16.0%** | −0.78 | −49.1% |

**전략은 한 글자도 바꾸지 않았다.** 같은 종목, 같은 날, 같은 매매.

```
가격충격 = η × 20일 변동성 × √( 주문금액 ÷ 20일 평균거래대금 )
반호가   = 0.5 × 호가단위 ÷ 가격
```

주문 ×4 → 충격 ×2. 원금 1억 기준 평균 충격 **0.076%**, 평균 반호가 **0.066%**.

### 그래서 얼마까지 넣을 수 있나 — 운용 규모를 바꿔 다시 돌렸다

전략·데이터·기간 모두 그대로 두고 **원금만** 바꿨다.

| 원금 | 편도 충격 | 거래당 | 연수익률 | 샤프 | 승률 |
|---|---|---|---|---|---|
| **1억** | 0.076% | **+0.148%** | **+39.8%** | 1.59 | 53% |
| **2억** | 0.108% | +0.086% | **+20.1%** | 0.88 | 51% |
| **4억** | 0.153% | **−0.003%** | **−3.0%** | −0.11 | 48% |
| **10억** | 0.241% | −0.179% | **−36.5%** | −2.09 | 43% |
| **30억** | 0.418% | −0.529% | **−72.7%** | −5.98 | 34% |
| **100억** | 0.762% | −1.206% | **−94.7%** | −13.21 | 22% |

**제곱근 법칙이 실측으로 확인된다.**

```
원금 ×2   →  충격 ×1.41   0.076% → 0.108%   (이론 0.1075%)
원금 ×4   →  충격 ×2      0.076% → 0.153%   (이론 0.1526%)
원금 ×10  →  충격 ×3.16   0.076% → 0.241%   (이론 0.2412%)
원금 ×100 →  충격 ×10     0.076% → 0.762%   (이론 0.7630%)
```

**손익분기는 원금 약 4억원이다.** 4억에서 거래당 수익이 **−0.003%** 로 정확히 0에 닿는다.
(√ 법칙으로 역산한 값 3.9억과 직접 실행 결과가 일치한다.)

**이 전략의 수용 한도는 수억 원 규모다.** 기관 자금으로는 쓸 수 없다.
**"연 39.8%"는 원금 1억이라는 조건에서만 성립하는 수치**이므로, 수익률 하나가 아니라 이 표를 결론으로 남긴다.

<details>
<summary><b>realistic.py</b> — 제곱근 충격 모델</summary>

```python
"""현실 비용 모델(대회 환경: 원금 1억, 매도세 0.20% 고정)로 재판정 → validation_realistic.csv

편도 슬리피지 = 가격충격 [+ 반호가]
- 가격충격 = ETA × 20일 일간변동성 × √(주문금액 / 20일 평균거래대금)   (제곱근 법칙, Vibe-Trading impact.py 방식)
  주문금액 = CAPITAL / 슬롯 수. 모두 전일까지 정보.
- 반호가 = 0.5 × 호가단위 / 가격 (KRX 2023년 이후 호가표, 원시 종가 기준)
- 시가봉(시가 동시호가) 체결: 가격충격만. 종가봉: 종가 동시호가 체결이면 가격충격만,
  손절·익절이 있는 전략은 종가봉 안에서 장중 손절이 나므로 반호가를 더한다(보수적).
  장중 진입봉: 반호가 + 가격충격.
ETF 는 기존대로(수수료만, 슬리피지 0.05%).

    python realistic.py                 # 기본 목록
    ETA=0.5 python realistic.py         # 충격 계수 민감도
"""
CAPITAL = float(os.environ.get("CAPITAL", 1e8))
ETA = float(os.environ.get("ETA", 1.0))      # 0 = 체결 비용 없음(수수료만)
TAX = float(os.environ.get("TAX", 0.0020))   # 대회(모의투자)는 세금 없음 → TAX=0
SPREAD = float(os.environ.get("SPREAD", 1.0))
```

</details>

---

## 13. 가짜라면 걸리는 시험

| 시험 | 통과 조건 |
|---|---|
| ① **위약** — 학습 목표를 날짜 안에서 무작위로 섞어 학습 | 수익이 **0 근처**여야 정상. 아니면 특성에 미래 정보 누수 |
| ② **하루 늦은 특성** — t−1 특성으로 t 신호 | 성과가 **크게 줄어야** 정상 |
| ③ **ML 없는 단순 규칙** — 회전율 상위 절반 중 종가 위치·VWAP 최하위 3 | 이걸 못 이기면 머신러닝은 장식 |

<details>
<summary><b>falsify.py</b> — 반증 시험 전문</summary>

```python
"""ML1LV 반증 시험 — 가짜라면 걸리는 시험들. 대회 종목, 상위 3,
비용(수수료 0.0140527%×2 + 세금 0.20%), 오버나잇(종가→익일 시가).

① 위약: 학습 목표를 날짜 안에서 무작위로 섞어 학습 → 수익이 0 근처여야 정상(아니면 특성에 미래 정보 누수)
② 하루 늦은 특성: t-1 특성으로 t 신호 → 성과가 크게 줄어야 정상(오늘 장중 정보를 쓰는 전략이므로)
③ 머신러닝 없는 단순 규칙: 회전율 상위 절반 중 종가 위치(cpos) 최하위 3 / VWAP 대비 최하위 3
시험 구간: 2024-01~2026-09 (각 연도 모델은 그 전 해까지 학습, 기존 walk_forward 와 같은 설정)
"""
COST = 2 * 0.000140527 + 0.002


def fit_predict(X, shuffle=False, lag=False):
    dates = X.index.get_level_values(0)
    feats = [c for c in X.columns if c != "y"]
    if lag:   # 특성을 하루 늦춤(종목별): t 행에 t-1 특성
        Xl = X[feats].groupby(level=1).shift(1)
        X = pd.concat([Xl, X["y"]], axis=1)
    preds = []
    for yr in YEARS:
        cut = ...
        tr = X[dates < cut].dropna(subset=["y"])
        y = tr["y"]
        if shuffle:
            rng = np.random.default_rng(yr)
            y = y.groupby(level=0).transform(lambda v: rng.permutation(v.to_numpy()))
        mdl = HistGradientBoostingRegressor(...).fit(tr[feats], y)
        ...


if __name__ == "__main__":
    top3(M.walk_forward("onv"), "기준: ML1LV (그대로)")
    top3(fit_predict(X, shuffle=True), "① 위약: 정답 섞어 학습")
    top3(fit_predict(X, lag=True), "② 하루 늦은 특성으로 예측")
    hi_turn = f["turn"].where(ok).rank(axis=1, pct=True) >= 0.5
    top3((-f["cpos"]).where(hi_turn), "③a 단순: 회전율 상위 + 저가 근처 마감")
    top3((-f["c_vwap_rng"]).where(hi_turn), "③b 단순: 회전율 상위 + VWAP 대비 최하")
    top3(pd.DataFrame(np.random.default_rng(0).random(d.C.shape), ...), "무작위 3종목(대조군)")
```

</details>

---

## 14. 실행 전에 규칙을 적었다

**2026-10-08 작성** · 신호 7 × 방식 6 = **42조합**
initial 2009~2021 Net Sharpe 1위를 '왕'으로 뽑고 **test·prediction 에서 바꾸지 않는다**

| | initial 09~21 | test 22~23 | prediction 24~26.07 |
|---|---|---|---|
| 왕 `high52 × buf_drift` | 0.553 | **−0.370** | 1.202 |
| 시장 (동일가중·비용 없음) | 0.477 | −0.100 | 0.822 |

| | |
|---|---|
| 과적합 확률 **PBO** | **0.72** — 42개 중 1위를 고르는 방식이 표본 밖에서 중앙값 아래로 떨어질 확률 |
| DSR | 0.91 |
| 왕 − 시장 Sharpe 차 (블록 부트스트랩 21일·5,000회) | initial p=0.30 / test p=0.78 / **prediction p=0.04** |

**결론의 단위를 낮췄다** — 42개 중 1~5위가 모두 `high52`. 방식 간 차이는 0.02 미만.
→ *"조합 하나"가 아니라 "52주 고가 근접도가 모든 리밸런싱 방식에서 꾸준히 낫다"*는 **신호 단위** 결론.
→ *"이전 공식안·시장보다 통계적으로 낫다고 말할 수는 없다."*

<details>
<summary><b>preregistration_king.md</b> — 사전 등록 원문 발췌</summary>

```markdown
# 모멘텀 "왕" 선발 사전 등록 (2026-10-08, 실행 전 작성)

## 기간 (팀 합의 2026-10-08)
initial 2009-01-09~2021 / test 2022~2023 / prediction 2024~2026.07.

## 선발 규칙
1. initial 구간 일간 비용 후 Net Sharpe 가 가장 높은 조합 = 왕. (동률이면 회전율 낮은 쪽)
2. 왕은 test·prediction 에서 바꾸지 않는다. test·prediction 결과는 보고만 한다.
3. 보고: 42개 전부의 구간별 CAGR·Sharpe·MDD·회전율·연비용,
   왕 vs 현재 공식안 Sharpe 차이의 블록 부트스트랩 p(21일 블록, 5,000회, 구간별),
   42개 시험에 대한 PBO·DSR.

## 미리 적는 예상
- 신호: 12-1 계열이 무난하고 단기(6-1)는 회전율이 높아 비용 후 불리할 것.
- 방식: cost_aware 와 buf_drift 는 회전율·비용을 줄이지만 Sharpe 개선 폭은 작을 것.

## 지키는 것
- 결과를 보고 후보·규칙·기간을 바꾸지 않는다. 바꾸면 새 등록으로 따로 적는다.
- 2017 년 이후 데이터는 앞선 실험에서 이미 봤다.
  test·predict 결과는 "사전 등록 후 확인"이지 완전 미열람 검증이 아니다.
```

</details>

---

## 15. 한계 — 먼저 적는다

### 비용 모델

- **슬리피지 구간(0.1 / 0.2 / 0.5%)은 실측이 아니다.** 기존 프로젝트(HollyKR)의 구간을 따랐다.
  그리고 슬리피지를 결정하는 것은 유동성인데 **시가총액으로 나눴다.** 둘은 상관이 높지만 같지 않다.
  → 그래서 **비용 2배 전수 재실행**과 `BT_SLIP` 민감도를 병행했다
- **엔진의 정액 슬리피지에는 주문 크기가 들어가지 않는다.** 1억을 굴리든 1,000억을 굴리든 같은 비용이다.
  규모를 반영한 제곱근 충격 모델과 **수용 한도(capacity) 산출은 단기 전략 한 건(`ML1L`)에만** 했다(§12).
  **전략 301개의 수용 한도는 계산하지 않았다** — 표의 연수익률은 규모와 무관한 값으로 읽어야 한다
- **2019년 중 세율 변경 미반영.** 거래세는 2019-06-03에 0.30% → 0.25% 로 내렸으나 연 단위 테이블이라
  2019년 전체를 0.30% 로 뒀다 (비용 과대 방향)
- **거래세 분배 방식.** vectorbt 가 매수·매도에 같은 비율을 부과하므로 매도세를 양쪽에 반씩 넣었다.
  왕복 합계는 같지만 세금 기준금액이 매도금액이 아니라 (매수+매도)/2 가 되어,
  **수익 난 거래의 세금이 조금 과소 계상**된다 (거래당 세율 × 수익률/2, 이 전략군에서는 무시할 수준)
- **수수료 값이 파일마다 다르다.** 엔진 `costs.py` 0.015%, 단기 `falsify.py` 0.0140527%
- **ETF 슬리피지가 단일 값(0.05%)이다.** 대형 지수 ETF에는 보수적이나 원유·레버리지·소형 ETF에는 낙관적이다

### 데이터

- **수정주가는 가격수익률이며 현금배당이 반영되지 않았다.**
  KRX 등락률(권리락·분할 조정 기준가 대비)을 누적해 만들었는데, 한국은 **현금배당에 기준가를 조정하지 않는다.**
  따라서 국내 개별주·국내 ETF의 배당·분배금 수익이 빠져 있다. 비교선인 코스피200도 같은 기준이라 상대 비교는 일부 상쇄되나,
  **고배당 전략과 저배당 전략의 비교는 왜곡된다.** (`AA` 계열이 신호로 쓰는 미국 ETF만 `Adj Close` 총수익이다)

### 그 외

- **수급 전략 6종은 순위에서 제외.** 수급 데이터에 생존편향과 수집 차단이 있어 결과를 신뢰하지 않는다
- 통과 전략 12개 중 **1차 자료를 확인하지 못한 것이 5건**, 원문과 다르게 구현된 것이 2건.
  **"출처대로 구현했다"고 말할 수 없다**
- PBO는 계산은 하되 **파일로 저장하지 않아** 값을 재확인할 수 없다
- 분봉 전략(ORB·VWAP·동시호가), 야간선물, 컨센서스 기반 이익 모멘텀은 **데이터가 없어 제외**
- 원천 시세·재무 패널(약 1.5GB)은 용량 때문에 올리지 않았다.
  `results/` 의 CSV로 위 표의 모든 수치를 재확인할 수 있다

---

## 저장소

| | |
|---|---|
| 전략 301개 엔진 · 전수 결과 CSV | [hsh2578/kr-strategy-board](https://github.com/hsh2578/kr-strategy-board) |
| 검증 보드 | [hsh2578.github.io/kr-strategy-board](https://hsh2578.github.io/kr-strategy-board/) |
| 비용 모델 | `backtest/costs.py` |
| 체결 엔진 | `backtest/engine.py` |
| DSR · 알파 · 랜덤 대조군 | `backtest/report.py` |
| 엔진 교차검증 기록 | `backtest/VERIFY_ENGINE.md` |

---

<sub>개인 학습·기록 목적입니다. 전부 백테스트 결과이며 실계좌 성과가 아닙니다. 특정 종목이나 전략의 매매를 권유하지 않습니다.</sub>
