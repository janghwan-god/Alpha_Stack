# Alpha_Stack · 팀 Qurious — 주가 등락 방향 예측과 성과 검증 엔진

5일 뒤 등락 방향을 맞히는 모델과, 그 결과를 믿어도 되는지 판별하는 검증 엔진을 따로 세운 프로젝트입니다.

> **"맞히는 것보다 어려운 건, 맞혔다고 말해도 되는지 아는 것"**

![python](https://img.shields.io/badge/python-3.12-blue)
![holdout](https://img.shields.io/badge/홀드아웃-개봉%201회%20·%20확증%20가능-9f1239)
![dashboard](https://img.shields.io/badge/대시보드-HF%20Space%20공개-yellow)
![data](https://img.shields.io/badge/데이터-KRX%20비재배포-red)
![license](https://img.shields.io/badge/코드-MIT-green)

<!-- 🔴 그림이 절대 URL 인 이유: GitLab 은 마크다운 이미지 문법의 상대경로만 저장소 경로로
     리라이트하고, raw HTML <img src="상대경로"> 는 그대로 이미지 프록시에 넘겨 404 가 난다
     (2026-09-09 실측). 그래서 이 저장소의 그림은 GitHub raw 절대 URL 로 건다.
     <picture> 도 마찬가지로 GitLab 에서는 벗겨진다 — 허용 목록이 SanitizationFilter +
     span·abbr·details·summary 뿐이라 <picture>·<source> 가 지워지고 안쪽 <img> 만 남는다
     (2026-09-20 문서 확인). 깨지지는 않고 라이트 그림으로 강등될 뿐이라 이대로 둔다. -->
<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/devlee328288/Alpha_Stack/main/docs/media/architecture/alphastack-pipeline.dark.svg">
    <img src="https://raw.githubusercontent.com/devlee328288/Alpha_Stack/main/docs/media/architecture/alphastack-pipeline.light.svg" alt="AlphaStack 파이프라인 — KRX·DART 등 일곱 출처를 ingest 계층이 SQLite 하나에 적재하고, supply 계층의 as_of 정문을 통과한 개발구간 자료만 피처·모델로 가며, 봉인된 홀드아웃 478거래일은 unseal.log 가 한 줄일 때만 한 번 열린다. 그 뒤 evaluation·backtest 를 거쳐 dashboard·reports·HF private 두 레포로 나간다." width="880">
  </picture>
</p>

**최종 홀드아웃 478거래일** — KOSPI200 정확도 **0.4331**, 개별종목 **0.3807**. 학습 구간에서 정한
기준선(ADR 0006) 대비 **+26.78%p** · **+9.49%p** 입니다. 다만 MCC 는 −0.0114 · 0.0585 라
KOSPI200 은 사실상 0 에 가깝습니다 — 자세한 읽는 법은 [수행결과](#수행결과)에 있습니다.

> 🖥️ **대시보드** — 바로 열어 보기 [Hugging Face Space](https://huggingface.co/spaces/data-student/alphastack-qurious) (Docker · 정본 [`deploy/hf-space/`](deploy/hf-space/))
> · 팀 내부용 [Streamlit Cloud](https://alphastack-qurious.streamlit.app/) (로그인 필요 · `main` 머지마다 자동 반영)
> 버튼을 누르면 서버에서 학습이 돕니다 — 무료 CPU 에서는 느리거나 제한에 걸릴 수 있습니다([#271](https://github.com/devlee328288/Alpha_Stack/issues/271)).

---

## 이 문서를 읽는 사람

| 당신이 | 여기부터 | 그다음 |
|---|---|---|
| **강사님·채용 담당자** 라면 | [수행결과](#수행결과) → [깊이 볼 만한 것](#깊이-볼-만한-것) | [알려진 한계](#알려진-한계와-향후-보완) · [팀](#팀) |
| **팀원** 이라면 | [빠른 시작과 재현](#빠른-시작과-재현) → [저장소 구조](#저장소-구조) | [docs/README.md](docs/README.md) · [AGENTS.md](AGENTS.md) · [CONTRIBUTING.md](CONTRIBUTING.md) |
| **처음 보는 분** 이라면 | [왜 이 문제인가](#왜-이-문제인가) → 위 대시보드 | [시스템 아키텍처](#시스템-아키텍처) |

## 목차

- [프로젝트 한눈에 보기](#프로젝트-한눈에-보기)
- [왜 이 문제인가](#왜-이-문제인가)
- [무엇을 하는가](#무엇을-하는가)
- [시스템 아키텍처](#시스템-아키텍처)
- [팀](#팀)
- [수행결과](#수행결과)
- [이 결과를 어떻게 읽어야 하나](#이-결과를-어떻게-읽어야-하나)
- [알려진 한계와 향후 보완](#알려진-한계와-향후-보완)
- [깊이 볼 만한 것](#깊이-볼-만한-것)
- [배운 것](#배운-것)
- [빠른 시작과 재현](#빠른-시작과-재현)
- [환경 변수와 데이터 출처](#환경-변수와-데이터-출처)
- [저장소 구조](#저장소-구조)
- [문서](#문서)
- [기여와 문의](#기여와-문의)
- [운영 시 유의사항과 면책](#운영-시-유의사항과-면책)
- [라이선스](#라이선스)
- [감사의 말](#감사의-말)

---

## 프로젝트 한눈에 보기

주가지수와 개별 종목의 **5일 뒤 등락 방향**을 맞히는 모델을 만들고, 그 결과를 **믿어도 되는지
판별하는 검증 엔진**을 따로 세웠습니다. 정확도 하나로 끝내지 않고 기준선 대비·거래비용 차감·
워크포워드 분산까지 함께 냅니다. 자료는 KRX·DART·공공데이터포털·한국은행에서 **시점 정합(as-of)**
을 지켜 모았고, 마지막 2년은 **봉인했다가 2026-09-11 에 한 번만** 열어 최종 모델을 평가했습니다.

| | |
|---|---|
| **주제** | 주가지수 데이터 활용 머신러닝·딥러닝 (AI 퀀트 4기 1차 프로젝트) |
| **기간·인원** | 2026-09-01(화) ~ 09-15(화) · 4인 · koreaIT 노원 B강의실 |
| **저장소** | https://github.com/devlee328288/Alpha_Stack (PUBLIC) · GitLab 미러 |
| **화면** | Streamlit Overview + 9페이지 · 공개 배포 [HF Space](https://huggingface.co/spaces/data-student/alphastack-qurious) |
| **자료 전달** | HuggingFace **private** 데이터셋 두 레포 — 원자료는 저장소에 두지 않습니다 |
| **상태** | 🟢 1차 마무리 · 최종 모델 확정 · **홀드아웃 개봉 1회 — 확증 가능**<br>🟡 홀드아웃 비용 차감 성과는 1차 범위에서 내지 않았습니다 |

### 2주가 어떻게 흘렀나

| 시점 | 무슨 일 |
|---|---|
| 2026-08-25 | 개인 프로젝트 `data-service` 에서 **9,031줄 선별 이관** + 734줄 신규 · 저장소 첫 커밋 |
| 09-01 | 킥오프 — 4인 역할 확정, 필수 범위 다섯 항목 합의 |
| 09-01 ~ 09-10 | 수집·정문·피처·모델 후보 · 수정주가 보정 · 데이터 품질 게이트 |
| **09-11 16:18** | **홀드아웃 1회 개봉** (`reports/unseal.log` 1줄) 후 최종 평가 1회 |
| 09-13 | 1차 발표 |
| 09-15 | 결과 보고서 v1.2 · 발표 최종본 v1.1 · 대시보드 배포 확인 |

### 지금 무엇이 있나 — 2026-09-20 실측

`data/krx_cache.db` 를 읽기 전용으로 열어 셌습니다. 코드·문서 수치는 `git ls-files` 추적 파일 기준입니다.

<table>
<tr><td valign="top">

| 자료 | 규모 |
|---|---:|
| 일별시세 (수정주가 4칸 · 총수익 3칸) | **9,231,938행** |
| 종목기본정보 (주권종류 정본) | 9,229,173행 |
| 종목 식별 (법인번호·ISIN) | 4,197,242행 |
| 공시 목록 | 1,555,556행 |
| 재무제표 | 662,933행 |
| 지수 | 196,272행 |
| 배당 · 기업행위 사건 | 71,681 · 44,290행 |
| 업종 스냅샷 (사람이 받은 36장) | 40,874행 |
| 거시지표 · 텍스트 신호 | 17,864 · 18,600행 |

스키마 **v16 · 표 23개 · 6,685 MB** · 거래일 4,105일

</td><td valign="top">

| 코드·기록 | 규모 |
|---|---:|
| 파이썬 (시험 제외) | **77,096줄** · 238파일 |
| 시험 | 22,373줄 · 103파일 |
| 시험 결과 | **1,782 passed · 1 failed · 3 xfailed**<br><sub>anaconda 3.12.13 · 417.8초 · 실패 1건은 [#307](https://github.com/devlee328288/Alpha_Stack/issues/307)</sub> |
| `ruff` | 220건 (`dashboard/` 206 · 그 밖 14) |
| 노트북 | 290권 (모델 후보 실험 257권 포함) |
| 문서 (`docs/` 마크다운) | 463편 |
| PR · 이슈 | 205건 머지 · 4인 전원 작성 |

</td></tr>
</table>

| HF private | 커밋 | 무엇 |
|---|---|---|
| `qurious-quant/alphastack-krx-dev` | `b5e3d52c` | 개발구간(~2024-08-31) 반출본 37칸 · 7,888,945행 — 홀드아웃 봉인본과 **바뀐 행 0** |
| `qurious-quant/alphastack-krx-holdout` | `2aab6bd1` | 최종 홀드아웃 예측 셋 + `report.json` — 원자료 가격 없음 · **학습에 섞지 않습니다** |

---

## 왜 이 문제인가

### "정확도 62%" 는 아무 말도 하지 않는다

주가 예측 프로젝트는 대개 정확도 몇 %로 끝납니다. 그런데 그 숫자는 혼자서는 아무것도
증명하지 못합니다.

- 상승이 62% 나오는 구간에서는 **언제나 "오른다"고 답하기만 해도** 62%가 나옵니다
- 시계열을 무작위로 섞어 나누면 미래로 과거를 예측하게 되어(look-ahead) 성능이 부풀어 오릅니다
- 거래비용을 빼지 않은 수익률은 **실현할 수 없는 숫자**입니다. 일간 매매는 왕복 비용이
  수익을 통째로 먹는 일이 흔합니다

그래서 이 프로젝트는 "잘 맞히는 모델"보다 **"맞혔다고 말해도 되는지 판별하는 장치"** 를
먼저 만들었습니다.

### 왜 검증 엔진을 따로 떼는가 — 세 차수가 같은 자를 쓴다

| 차수 | 주제 |
|---|---|
| 1차 | 주가지수 데이터 활용 머신러닝·딥러닝 |
| 2차 | 나만의 로보 어드바이저 개발 **및 성과 검증** |
| 3차 | 나만의 투자 인디케이터 개발 **및 성과 검증** |

셋 다 "개발 및 성과 검증"으로 끝납니다. 차수마다 새로 만드는 것은 **앞쪽**(신호 →
포트폴리오 → 지표)이고, **재는 방법은 바뀌지 않습니다.** MDD 를 재는 법은 무엇이 그
수익률을 만들었든 같습니다.

검증 코드를 노트북 셀에 흩어 놓으면 1차는 통과하고 **2차에 전부 다시 짜게 됩니다.**
그 비용을 지금 없애는 것이 이 저장소 구조의 이유입니다.

---

## 무엇을 하는가

### 필수 범위 — 반드시 닫는다

| # | 항목 | 완료 조건 | 상태 |
|---|---|---|---|
| ① | 기술적 지표 기반 등락 방향 예측 모델 | 예측 결과가 재현된다 | ✅ 조합 C · K LogisticRegression · 설정 지문 `5299a08f…` |
| ② | ML 모델 성능 비교·평가 | 동일 조건 비교표가 나온다 | ✅ KOSPI200 100후보 · 개별종목 44후보 |
| ③ | 롤링(walk-forward) 백테스팅 | 폴드별 성능과 **분산**이 나온다 | ✅ 12폴드 expanding · 🟡 비용 차감 성과는 미산출 |
| ④ | 재현 가능한 단일 파이프라인 | 한 명령으로 같은 결과 | ✅ 데이터 `pipelines.refresh` 8단계 · 최종 `run_final_holdout.py` |
| ⑤ | **성과 검증 엔진 분리** (MDD·Sharpe·승률·거래비용) | 2·3차가 그대로 가져다 쓴다 | ✅ `evaluation/` 은 `models/` 를 import 하지 않습니다 (실측 0건) |

### 선택 범위

⑥ 딥러닝(LSTM/GRU) 동일 조건 비교 · ⑦ 뉴스·공시 제목 감성 피처(무료 인코더만 — **만들었고 신호가 없었습니다**) ·
⑧ 유니버스 확장 350종목 횡단면 랭킹 · ⑨ ADF 정상성 검정(**기구현**) · ⑩~⑫ 인프라·팀원 제안

---

## 시스템 아키텍처

### 계층 — 의존은 아래로만 흐른다

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/devlee328288/Alpha_Stack/main/docs/media/architecture/alphastack-layers.dark.svg">
    <img src="https://raw.githubusercontent.com/devlee328288/Alpha_Stack/main/docs/media/architecture/alphastack-layers.light.svg" alt="AlphaStack 계층과 의존 — 아래에서부터 common, ingest, supply, features, models 가 한 줄로 쌓여 위층이 바로 아래층에만 기댄다. supply 에는 as_of 정문과 봉인 입구 두 문이 있다. evaluation 은 이 줄 밖에 서서 포지션·수익률 배열만 받으므로 models 를 import 하지 않는다. backtest·timeseries·pipelines.refresh 가 그 옆에 서고, dashboard 는 맨 위에서 전부를 쓴다. features·models·evaluation 이 ingest 를 직접 import 하면 경계 시험이 막는다." width="960">
  </picture>
</p>

<sub>정본 [`docs/media/architecture/alphastack-layers.light.svg`](docs/media/architecture/alphastack-layers.light.svg) (다크 판은 `.dark.svg` — 색 값 여덟 개만 다릅니다) · 같은 그림의 mermaid 판은 [계층아키텍처.mmd](docs/아키텍처/version1.3/계층아키텍처.mmd) · 발표용 구성도 [시스템아키텍처.png](docs/아키텍처/version1.3/시스템아키텍처.png) · 전문은 [아키텍처 v1.3](docs/아키텍처/version1.3/아키텍처.md)</sub>

### 모듈 — 무엇이 무엇을 받아 무엇을 내놓나

| 모듈 | 역할 | 받는 것 | 내놓는 것 | 의존 |
|---|---|---|---|---|
| [`common/`](common/) | 설정 · 경로 · 자격증명 · 거래일 · 기업행위 판정 | — | 상수와 판정 함수 | **아무것도 import 하지 않는다** |
| [`ingest/`](ingest/) | 바깥과 통신해 SQLite 하나에 쌓는다 | 외부 API 응답 · 사람이 받은 표 | `data/krx_cache.db` (v16) | `common` |
| [`supply/`](supply/) | **as_of 정문** · 봉인 입구 | DB 와 기준일 `T` | `known_at ≤ T` 인 행만 | `common` · `ingest.store`(내부에서만) |
| [`features/`](features/) | 시세 → 기술적 지표 → 학습 가능한 표 | 정문 통과분 | `X`, `y` (조합 A~K) | `supply` |
| [`models/`](models/) | 표 → 등락 방향 분류기 | `X`, `y` | 예측 · 확률 · 모델 지문 | `features` |
| [`evaluation/`](evaluation/) | **예측 → 믿어도 되는가** | 포지션 배열 · 수익률 배열 | 정확도 · 기준선 대비 · MDD · Sharpe | `common` 뿐 — **`models` 를 모른다** |
| [`backtest/`](backtest/) | A · B · C 분할매매 · 비용 민감도 | 예측과 가격 | 체결 원장 · 비용별 성과 | `supply` · `evaluation` |
| [`timeseries/`](timeseries/) | numpy 로 손수 짠 ARIMA · ADF · ACF/PACF | 시계열 | 통계적 기준선 예측 | `numpy` 뿐 |
| [`dashboard/`](dashboard/) | Streamlit Overview + 9페이지 | HF 개발본 | 화면 | 위 전부 |
| [`pipelines/`](pipelines/) | 수집 → 게이트 → 반출 → HF 8단계 | 명령 하나 | 갱신된 DB 와 반출본 | 위 전부 |

**`evaluation/` 은 `models/` 를 import 하지 않습니다.** 받는 것은 포지션 배열과 수익률
배열뿐입니다. 무엇이 그 포지션을 만들었는지 모르게 짜 두면 2·3차가 그대로 가져다 씁니다.
이 무지(無知)가 재사용의 조건입니다.

> `evaluation` 안에서 `models` 를 import 하고 싶어지면 설계가 틀어진 신호입니다.
> import 를 추가하지 말고 **함수 인자를 늘립니다.**

### 홀드아웃은 한 번만, 기록과 함께 연다

```
open       사전등록 커밋 · 추적 파일 변경 0 · 데이터 게이트 → 봉인 반출 → reports/unseal.log 한 줄
authorize  기록 1줄 · 지문 일치 · 개발/홀드아웃 안 섞임 → 승인 JSON (실행기 계약 여섯 칸)
official   run_final_holdout.py --mode official (모델 파트 실행기 · 설정은 개봉 전에 봉인)
check      사전등록이 개봉보다 앞섬 · 결과 지문 · 산출물에 원자료·시크릿 없음 → "확증 가능"
```

두 번째 개봉은 **반출하기 전에** 거부합니다. 전문은 [홀드아웃 개봉 절차](docs/데이터파트/version4.6/홀드아웃_개봉_절차.md).

---

## 팀

**2026-08-29 킥오프 확정 · 4인.** 각자의 작업은 PR 로 추적됩니다.

| 이름 | 역할 | 맡은 곳 | 작성한 PR |
|---|---|---|---|
| **이동원** (팀장) | **데이터** — 수집·저장·정제·전처리 · 문서화 | [ingest/](ingest/) · [common/](common/) · [supply/](supply/) · [scripts/](scripts/) · [docs/](docs/) | [PR 목록](https://github.com/devlee328288/Alpha_Stack/pulls?q=is%3Apr+author%3Adevlee328288) · [TIL 78편](docs/TIL/이동원/) |
| **오준영** | **모델** — 표 → 등락 방향 분류기 | [models/](models/) | [PR 목록](https://github.com/devlee328288/Alpha_Stack/pulls?q=is%3Apr+author%3AGoonGuMa) |
| **강민석** | **백테스팅 · 성과지표 · 화면** | [evaluation/](evaluation/) · [backtest/](backtest/) · [dashboard/](dashboard/) | [PR 목록](https://github.com/devlee328288/Alpha_Stack/pulls?q=is%3Apr+author%3Arkdalstjr) |
| **신장환** | **피처 · 품질** — 지표 계산 | [features/](features/) | [PR 목록](https://github.com/devlee328288/Alpha_Stack/pulls?q=is%3Apr+author%3Ajanghwan-god) |

**대표 작업** — 이동원: `as_of` 정문 · 기업행위 사건 표(v16) · 총수익 축 · 상장 구간 판정 ·
**홀드아웃 개봉 통제** · 대시보드 배포 / 오준영: 조합 A~K · 최종 C · K LogisticRegression ·
최종 홀드아웃 실행기 · 마지막 구간(ADR 0010) / 강민석: 동적 기준선(CMA-ES) · A·B·C 분할매매 ·
비용 민감도 · **대시보드 9페이지** / 신장환: 원자 지표 · 파생 피처 넷 · 유동 임계값 · 발표 03·04번

> 파트 경계를 넘어야 할 때는 **먼저 담당자에게 묻고**, PR 본문에 왜 넘었는지 적고,
> 관련 문서에도 남깁니다. 화면과 공용 파일(루트 README·`pyproject.toml`·`conftest.py`)은
> 여러 사람이 함께 쓰므로 같은 절차를 따릅니다.

**코드 기반**: 이동원 개인 프로젝트 `data-service` 에서 필요한 것만 골라 이관 후 재구성
(2026-08-25 · 9,031줄 이관 + 734줄 신규 · 모듈 import 검증 32/32).

---

## 수행결과

### 최종 홀드아웃 — 한 번 열고, 한 번 돌렸다

> 2026-09-11 16:18 KST 팀 합의로 **한 번** 열었습니다(`reports/unseal.log` 1줄).
> ADR 0009 설정 그대로 **한 번** 돌렸고 판정은 **확증 가능**입니다.
> 🔴 결과를 보고 모델·피처·임계값을 다시 고르지 않았습니다.

**측정 조건** — 평가 478거래일 (2024-09-02 ~ 2026-08-24) · 12폴드 expanding walk-forward ·
시드 고정 · 설정 지문 `5299a08f4107…` · 기준선은 **학습에 쓴 개발구간의 최빈 클래스(중립)를
평가 전체에 예측한 정확도**([ADR 0006](docs/decisions/0006-개별종목-중립대와-평가기준선.md))

| 지표 | KOSPI200 · 조합 C LogReg | 개별종목 · 조합 K LogReg |
|---|---:|---:|
| Accuracy | 0.4331 | 0.3807 |
| **학습 최빈 기준선 대비** | 0.1653 → **+26.78%p** | 0.2858 → **+9.49%p** |
| Macro F1 | 0.3126 | 0.3740 |
| Balanced accuracy | 0.3285 | 0.3741 |
| 하락 재현율 | 0.1087 | 0.4498 |
| **MCC** | **−0.0114** | **0.0585** |
| PR-AUC (macro OvR) | 0.3611 | 0.3880 |
| 평가 행 수 | 478 | 23,567 |
| 매수 후보 (두 트랙 모두 상승) | — | **7,397** / 23,567건 |
| 참고 · 평가구간 최빈(상승) — 정답을 본 사후값 | 0.5460 → −11.30%p | 0.3655 → +1.52%p |

**재생산** — 결과 파일 [`reports/final_holdout/final-holdout-20260911_report.json`](reports/final_holdout/final-holdout-20260911_report.json)

```bash
python scripts/run_final_holdout.py --mode official   # 봉인 해제 기록이 1줄일 때만 실행됩니다
```

- 두 트랙 모두 기준선을 넘었습니다. 다만 중립 비율이 개발구간 38.6% · 38.0% 에서 평가구간
  16.5% · 28.6% 로 줄어 **차이가 부풀어 보입니다** — 그래서 MCC 를 함께 읽습니다.
  KOSPI200 의 −0.0114 는 사실상 **무작위와 구분되지 않습니다.**
- 평가구간 최빈(상승) 비율은 정답을 보고 센 사후값이라 우위 판정에 쓰지 않습니다(ADR 0006 · 0007).
- 개발구간 표본 밖에서는 KOSPI200 0.4542 · 개별종목 0.4211 이었습니다. 기간이 달라 위 기준선과
  직접 비교하지 않습니다.

**마지막 한 구간** ([ADR 0010](docs/decisions/) · 판단일 2026-08-24 → 청산 09-01) — 개봉 뒤 모델 파트가
최종 산출물로 정한 운영 평가입니다. KOSPI200 예측 상승 · 실제 상승 · 개별종목 50종목 정확도 0.52 ·
**매수 후보 33** ([종목표](notebooks/04-모델/최종결과/README.md)). 표본이 1일이라 장기 성능으로 읽지
않고 위 2년 결과와 섞지 않습니다.

### 데이터 파트가 막은 것 — 숫자로

| 무엇 | 실측 |
|---|---|
| 시점 칸이 있어도 날짜로 붙이면 샌다 (하루 누수) | 업종 5,300 · 소속부 6,276 · 상장 첫날 3,677행 → **0** (PR #243) |
| 원천 정문의 음성 대조군 — 틀린 구현이 실제로 새는가 | 결산기 끝 23.2% · 원본 접수일 9.1% vs 정문 **0** |
| 코드 재사용 자리에서 앞 회사 기록 | 개발 74 · 홀드아웃 187행 → **0** (PR #253) |
| 봉인본 개발구간 vs HF 개발본 | 37칸 · 7,888,945행 · **바뀐 행 0** |

---

## 이 결과를 어떻게 읽어야 하나

모델·데이터 카드 요약입니다. 이 네 가지를 모르면 위 숫자를 잘못 읽게 됩니다.

| | |
|---|---|
| **의도된 용도** | 학습·포트폴리오 목적의 **연구 재현**. 방법론(시점 정합·기준선·워크포워드)의 시연 |
| **범위 밖 사용** | 실매매 · 투자 자문 · 자동 주문. 성과는 **비용 차감 전**이고 슬리피지·체결 실패를 담지 않습니다 |
| **예측 대상과 라벨** | 5거래일 뒤 등락 **방향 3분류** — `-1` 하락 / `0` 중립 / `+1` 상승.<br>중립대는 **지수 ±1.0%**, **개별종목 ±2.0%**([ADR 0006](docs/decisions/0006-개별종목-중립대와-평가기준선.md)) |
| **기준선의 정의** | 학습 구간(개발구간)의 **최빈 클래스를 평가 전체에 예측**한 정확도.<br>평가구간의 분포로 잰 값은 **정답을 본 사후값**이라 우위 판정에 쓰지 않습니다 |
| **포지션 규약** | `{0, +1}` — 공매도를 쓰지 않으므로 **하락 예측은 실행상 중립과 같습니다** |
| **자료 시점** | 2010-01-04 ~ 2026-09-04 · 학습은 **~2024-08-31** 까지만 · 그 뒤 478거래일은 봉인 |

---

## 알려진 한계와 향후 보완

정직하게 적습니다. 이 목록이 비어 보이는 저장소는 대개 **안 재 본 것**입니다.

| 무엇 | 크기 | 지금 영향 | 읽는 사람이 할 일 |
|---|---|---|---|
| **계약 시험 1건 실패** | `signal_log` 가 19칸인데 계약 상수는 17칸 (`action` · `in_cooldown` 이 PR #301 에서 늘었습니다) | 원장 검증기 자체는 정상 · 칸 목록 대조만 실패 | 1차 코드 동결로 **고치지 않습니다** — 두 선택지를 적어 닫았습니다 [#307](https://github.com/devlee328288/Alpha_Stack/issues/307) |
| **gap 5 와 T+6 라벨 하루 겹침** | 학습 마지막 T=2024-08-23 의 청산일 = 평가 첫날 2024-09-02 | 평가 라벨을 보지는 않습니다 · 개봉 뒤라 바꾸지 않습니다 | [개봉 절차 §8.6](docs/데이터파트/version4.6/홀드아웃_개봉_절차.md) 을 먼저 읽으십시오 |
| 홀드아웃 끝을 2026-09-01 로 당김 | 09-02 ~ 09-04 수정주가 계산 전 8,294행 | 마지막 사흘을 평가에서 뺐습니다 | 2년 결과를 최신 3일과 이어 붙이지 마십시오 |
| 대시보드가 **화면 안에서 학습** | Baseline CMA-ES · Model Lab 4모델 | 무료 CPU 에서 제한 · 재시작하면 캐시 소실 · 결과 캐시를 모든 접속자가 공유 | 공식 성적은 화면이 아니라 위 [수행결과](#수행결과)에서 읽으십시오 ([#271](https://github.com/devlee328288/Alpha_Stack/issues/271)) |
| 화면 수치와 공식 수치가 다름 | LR 정확도 화면 0.3736 vs 공식 0.4542 | 화면은 조합·비용 프리셋이 공식과 다릅니다 | 보고서 숫자는 결과 파일에서 읽습니다 ([#280](https://github.com/devlee328288/Alpha_Stack/issues/280)) |
| **홀드아웃 비용 차감 성과 미산출** | Sharpe · MDD · 비용 차감 수익률 | 최종 결과가 분류 지표까지만 있습니다 | 결과 보고서 4.5절이 빈자리임을 전제하고 읽으십시오 |
| `ruff` 220건 | `dashboard/` 206 · 그 밖 14 | 형식만 · 기능 영향 없음 | — |
| 점 조회 `universe_rows` 가 상장 구간을 건넌다 | `036220` · `101970` 상장 다음 날 | 공식 경로는 쓰지 않습니다 — 닫음 | [#252](https://github.com/devlee328288/Alpha_Stack/issues/252) |

**반출 경로가 이걸 강제합니다.** `scripts/upload_to_hf.py` 가 검증기를 먼저 돌리고
하나라도 붉으면 **올리지 않습니다**. 알면서 내보내려면 `--skip-verify` 를 주고
데이터셋 카드에 그 사실을 적어야 합니다.

**2차에서 하려는 것** — 대시보드가 미리 계산한 결과를 읽기만 하게 · 공통 평가 지표 스펙과
비용 규격 확정 · 새로 봉인할 때 gap 을 라벨 지평(6)과 맞추기 · 버튼 갱신 게이트를 CI 로.

---

## 깊이 볼 만한 것

코드를 열어 볼 분에게 권하는 세 곳입니다. 나머지는 [핵심코드 v1.4](docs/핵심코드/version1.4/)
(줄 단위 해설 7편)에 있습니다.

### 1. 거래비용은 조용히 틀린다 — [evaluation/metrics.py](evaluation/metrics.py)

포지션이 바뀔 때만 비용이 듭니다. 그런데 **첫 시점의 진입 비용**을 빠뜨리기 쉽습니다.
`positions[0]` 이 0 이 아니면 아무것도 없던 상태에서 잡은 것이므로 비용이 드는데,
`np.diff` 로 짜면 첫 원소가 사라져 매 폴드마다 진입이 공짜가 됩니다.

```python
previous = np.concatenate(([0.0], pos[:-1]))   # 맨 앞에 0 을 덧대는 이유가 이것
turnover = np.abs(pos - previous)
```

→ [tests/test_evaluation.py](tests/test_evaluation.py) 의
`test_첫_시점_진입_비용을_빠뜨리지_않는다` 가 이걸 잡습니다.

### 2. 포지션 정렬 한 줄 — [evaluation/metrics.py](evaluation/metrics.py) 모듈 docstring

```python
signal   = model.predict(features)   # t 일 종가까지 보고 낸 신호
position = np.roll(signal, 1)        # t+1 일 수익률에 적용  ← 이 한 줄
position[0] = 0
```

이걸 빠뜨리면 정확도 60% 짜리 모델이 연 300% 를 벌어 줍니다.
**그런 결과가 나오면 기뻐하기 전에 이 정렬부터 의심합니다.**

### 3. 판정기가 대상과 같은 함수를 쓰면 초록은 무의미하다 — [supply/listing_segment.py](supply/listing_segment.py)

코드 재사용 자리에서 앞 회사의 업종이 붙은 74행은 **값으로 대조해도 안 보였습니다.** 옛 값('제약')이
새 회사의 나중 값과 우연히 같았고, 재배포 판정기도 반출과 같은 함수로 붙였기 때문입니다. 수정 확인은
판정기가 아니라 **구간을 지키는 단순 루프**(다른 코드)로 기대값을 만들어 했습니다.

<details>
<summary><b>더 볼 만한 것 세 가지</b> — gap 누수 · 생존 편향 · 손수 짠 ARIMA</summary>

**gap 이 없으면 5일 레이블이 학습에 샌다** — [evaluation/walk_forward.py](evaluation/walk_forward.py)
"5일 뒤 상승" 레이블은 t 행이 미래 가격을 알아야 만들어집니다. 학습 구간 마지막 행의 레이블이
검증 구간 초반과 **겹칩니다.** 레이블이 k 일 앞을 보면 `gap=k` 로 둡니다. **이걸 빠뜨려도 에러가
나지 않습니다** — 성능이 조용히 부풀어 오를 뿐입니다.

**상장폐지 종목을 지우지 않는 이유** — [sql/init/01-schema.sql](sql/init/01-schema.sql)
`is_delisted boolean NOT NULL DEFAULT false` — 지우면 **생존 편향**이 생깁니다. 망한 종목이 빠진
유니버스로 백테스트하면 실제보다 훨씬 좋은 결과가 나옵니다.

**numpy 로 손수 짠 ARIMA** — [timeseries/](timeseries/)
`scipy` · `statsmodels` 없이 ADF(MacKinnon 임계값) · ACF(Bartlett) · PACF(Durbin-Levinson) ·
ARIMA(Hannan-Rissanen)를 직접 구현했습니다. ⑨ 정상성 검정은 이미 구현되어 있고, ARIMA 는
우리 분류기가 겨룰 통계적 기준선입니다.

</details>

---

## 배운 것

**① 경로 깊이는 조용히 깨진다** — 원본의 `parents[2]` 는 폴더 구조를 바꾸자 루트가 아닌 곳을
가리켰지만 **import 는 멀쩡히 통과했습니다.** 파일을 실제로 읽는 순간에야 드러납니다.
그래서 [common/paths.py](common/paths.py) 에 기준점을 모았습니다.

**② "설치된다"와 "import 된다"는 다르다** — `uv pip compile` 은 numpy 2.5.1 + sklearn 조합을
문제없이 해석했지만, 의존성 해석 성공이 ABI 호환을 보장하지 않습니다. 배포에서도 같았습니다 —
진입점 폴더의 요구사항 파일이 루트 파일을 이겨 첫 화면이 멈췄고, `--extra-index-url` 한 줄이
다른 패키지 버전을 조용히 끌어내렸습니다(PR #266).

**③ 기준선은 정답을 보기 전에 정한다** — 개발구간 기준선과 홀드아웃 정확도를 비교해
"기준선 아래"라고 적었다가, 홀드아웃 구간의 최빈 비율로 다시 재 "+1.52%p 위"라고 고쳤습니다.
그런데 그 비율은 **평가구간 정답을 보고 센 사후값**이었습니다. 기간을 맞추는 것과 정답을 보지
않는 것은 다른 조건입니다.

**④ 한 번만 여는 것은 코드로 막아야 한다** — "홀드아웃은 한 번만"을 문서에만 적으면 드라이런이
진짜 기록을 채우는 순간 진짜 개봉이 두 번째가 됩니다. 그래서 기록 파일을 가르고(`reports/dryrun/`),
두 번째 줄은 **반출하기 전에** 거부하게 했습니다.

**⑤ 검사가 대상과 같은 잘못을 공유하면 초록은 무의미하다** — 판정기를 반출과 같은 함수로 짜면
같이 틀리고 같이 초록이 됩니다. 확인은 **다른 코드**로 합니다.

---

## 빠른 시작과 재현

### 요구사항

- **Python 3.12** (실측 환경은 anaconda 3.12.13)
- `uv` (없으면 `pip install uv`)
- 학습·검증까지는 **외부 연결도 API 키도 필요 없습니다.** 실제 수집만 `.env` 가 필요합니다

### 1. 설치

```bash
git clone https://github.com/devlee328288/Alpha_Stack.git
cd Alpha_Stack

uv venv .venv --python 3.12
uv pip install --python .venv/Scripts/python.exe -e ".[dev]"   # macOS/Linux 는 .venv/bin/python
```

### 2. 검증 엔진이 살아 있는지 확인

```bash
python -m pytest tests/ -q
#  기대 출력: 1,782 passed · 1 failed · 3 xfailed  (anaconda 3.12.13 · 417.8초 · 로컬 22코어)
#  실패 1건은 알려진 것입니다 — 위 "알려진 한계와 향후 보완" 참고

python -m ruff check .
#  기대 출력: 220건 (dashboard/ 206 · features/ 11 · evaluation/ 3) — 형식만입니다
```

> `.venv` 는 설치된 패키지에 따라 수가 다를 수 있습니다. `FinanceDataReader` 가 없으면
> 수집 관련 시험이 오류로 빠집니다. 위 기준선은 **anaconda 환경**에서 잰 것입니다.

### 3. 학습과 평가

```bash
python -m pipelines.refresh              # 수집 → 게이트 → 반출 → HF · 8단계 (키 필요)
python scripts/run_final_holdout.py --mode official   # 봉인 기록이 1줄일 때만 실행됩니다
```

### 4. 산출물이 어디에 남나

| 경로 | 무엇 |
|---|---|
| [`reports/final_holdout/`](reports/final_holdout/) | 최종 홀드아웃 성적 JSON (설정 지문 · 입력 SHA-256 포함) |
| `reports/unseal.log` | 봉인 해제 기록 — **행이 2개가 되는 순간 프로젝트는 실패로 보고합니다** |
| `reports/quality_ledger.jsonl` | 품질 원장 5축 17지표 이력 |
| HF private 두 레포 | 개발본 · 홀드아웃 결과 (원자료는 올리지 않습니다) |

### 5. 대시보드

```bash
streamlit run dashboard/app.py      # 또는 dashboard/run.bat
#  데이터는 HF private 에서 받습니다 — HF_TOKEN 이 필요합니다
```

공개 배포는 [HF Space](https://huggingface.co/spaces/data-student/alphastack-qurious) 입니다
(수동 Factory rebuild · 사용설명서 [#274](https://github.com/devlee328288/Alpha_Stack/issues/274)).
아래는 2026-09-15 에 배포본에서 직접 찍은 화면입니다 — [10화면 전체](docs/media/배포화면/2026-09-15/README.md).

<table>
<tr>
<td width="33%"><img src="https://raw.githubusercontent.com/devlee328288/Alpha_Stack/main/docs/media/%EB%B0%B0%ED%8F%AC%ED%99%94%EB%A9%B4/2026-09-15/00_Overview.png" alt="Overview 화면 — 자료 규모와 파이프라인 상태를 한 장에 요약한다"></td>
<td width="33%"><img src="https://raw.githubusercontent.com/devlee328288/Alpha_Stack/main/docs/media/%EB%B0%B0%ED%8F%AC%ED%99%94%EB%A9%B4/2026-09-15/02_Model_Lab.png" alt="Model Lab 화면 — 조합과 모델을 골라 화면 안에서 학습시키고 지표를 비교한다"></td>
<td width="33%"><img src="https://raw.githubusercontent.com/devlee328288/Alpha_Stack/main/docs/media/%EB%B0%B0%ED%8F%AC%ED%99%94%EB%A9%B4/2026-09-15/04_Backtest.png" alt="Backtest 화면 — A·B·C 분할매매 전략의 누적 성과와 체결을 그린다"></td>
</tr>
<tr>
<td><sub><b>Overview</b> — 자료 규모와 파이프라인 상태</sub></td>
<td><sub><b>Model Lab</b> — 조합·모델을 골라 화면 안에서 학습</sub></td>
<td><sub><b>Backtest</b> — 분할매매 전략의 누적 성과</sub></td>
</tr>
</table>

> ⚠️ 화면의 수치는 화면 설정으로 다시 계산한 값이라 **공식 성적이 아닙니다.**
> 공식 성적은 [수행결과](#수행결과)와 `reports/final_holdout/` 의 JSON 에서 읽습니다.

> 📓 **자료가 실제로 어떻게 생겼는지** 보려면 [`notebooks/`](notebooks/) 를 보세요.
> 마크다운 설명 + 코드 + 실행된 출력이 한 장에 있고, GitHub 웹이 그대로 렌더링하므로
> 클론하지 않고 링크만 눌러도 읽힙니다.

---

## 환경 변수와 데이터 출처

```bash
cp .env.example .env      # 값은 팀장(이동원)에게 요청
```

> ⚠️ **`.env` 를 절대 커밋하지 마세요.** 이 저장소는 PUBLIC 입니다.
> `.gitignore` 가 막고 있지만, push 전 `git status --short` 로 눈으로 확인합니다.

| 변수 | 용도 | 필수 |
|---|---|---|
| `STORE_BACKEND` | `sqlite` 고정 — 1차는 SQLite 하나입니다 | ✅ (기본값 있음) |
| `KRX_API_KEY` | 시세·지수 수집. 이것 없이는 학습 데이터가 없습니다 | ✅ |
| `DART_API_KEY` | 공시·재무 | 선택 |
| `ECOS_API_KEY` · `FRED_API_KEY` · `KOSIS_API_KEY` | 거시 통계 | 선택 |
| `DATA_GO_KR_API_KEY` | 공공데이터포털 — **계정당 키 하나**로 모든 API 를 씁니다 | 선택 |
| `HUGGINGFACE_ACCESS_TOKEN` | ⑦ 무료 인코더 · 데이터셋 전달 | 선택 |
| `NAVER_API_CLIENT_ID` · `NAVER_API_SECRET_KEY` | 뉴스·카페글 — ⚠️ **신규 발급 중단** | 선택 |
| `YOUTUBE_API_KEY` | 동영상 | 선택 |

**KRX 하나만 있으면 1차 필수범위(시세·지수·백테스트)는 전부 돕니다.**
자가 점검은 [`scripts/check_keys.py`](scripts/check_keys.py), 전원용 상세 가이드는
[API 키 발급 가이드](docs/데이터파트/version4.6/API키_발급_가이드.md) 입니다.

| 출처 | 쓰는 곳 | 한도 | 저작권·재배포 |
|---|---|---|---|
| [KRX OpenAPI](https://openapi.krx.co.kr) ★필수 | 시세·지수 | 문서상 10,000콜/일 | **재배포 금지** — 저장소·HF 공개본에 원자료를 두지 않습니다 |
| [DART](https://opendart.fss.or.kr) | 공시·재무 | 기본 20,000콜/일 (키별로 다를 수 있음) | 공공데이터 — 출처 표시 |
| ECOS · FRED · KOSIS | 거시 통계 | 여유 | 각 기관 조건 |
| 공공데이터포털 | 종목 식별·기업정보 | 개발계정 10,000회/일 | 공공누리 2유형 |
| KRX 업종 분류 | 업종 스냅샷 36장 | — | **약관이 자동 수집을 금지** → 사람이 받은 표만 씁니다 |
| **`finance.naver.com`** | — | — | 🚫 **수집하지 않습니다** (아래) |

<details>
<summary><b>네이버 검색 API 신규 발급 중단과, 수집하지 않는 곳</b></summary>

**네이버 검색 API — 2026-07-30 부로 신규 신청이 막혔습니다.** 개발자센터 약관 부칙 제2조 ①:
*"2026년 7월 30일 24:00를 기점으로 (…) Search API (…) 신규 이용 신청 접수를 중단합니다."*

- **7/30 이전에 발급받은 키가 있다면** 그대로 씁니다. 단 **2027-06-30 24:00** 이 하드
  데드라인입니다(부칙 제2조 ②·④). 그래서 코드는 `base_url` 과 인증 헤더를 상수로 분리해
  **HUB 전환이 한 줄 교체**가 되도록 짰습니다.
- **키가 없다면** [NAVER Cloud Platform 의 API HUB](https://www.ncloud.com) 로 가야 합니다.
  인증 헤더가 `X-NCP-APIGW-API-KEY-ID`/`X-NCP-APIGW-API-KEY` 로, 경로가
  `/v1/search/news.json` → `/search/v1/news` 로 바뀌고, 한도가 일 25,000 이 아니라
  **월 775,000건(월 단위 관리)** 이 됩니다.

⚠️ 한도는 **클라이언트 아이디(앱)별 합산**입니다 — 뉴스와 카페글이 같은 쿼터를 나눠 씁니다.
그리고 약관 7.3 ⑤ 가 **쿼터를 늘릴 목적의 다중 키 발급을 금지**하므로,
*"팀원마다 키를 받아 분산"* 은 설계에서 배제했습니다.

**`finance.naver.com` 은 수집하지 않습니다.** `robots.txt` 를 직접 받아 확인한 결과,
`?code=*` 를 허용하는 규칙은 전부 `User-agent: yeti`(네이버 자체 크롤러) 그룹 소속이고,
우리 같은 제3자 봇이 매칭되는 `User-agent: *` 그룹은 `Disallow: /` 한 줄입니다.
RFC 9309 상 크롤러는 자신에게 매칭되는 **그룹 하나만** 따르므로 **사이트 전체가 불허**입니다.
→ 종목토론방 수집은 **Won't** 로 내렸습니다.

</details>

### 기술 스택

| 영역 | 무엇을 씁니다 |
|---|---|
| 수집·저장 | `requests` · `FinanceDataReader` · `SQLAlchemy 2.0.52` · SQLite |
| 수치·표 | `numpy 2.5.1` · `pandas 3.0.5` · `scipy 1.18.1` |
| 모델 | `scikit-learn 1.9.0` · `LightGBM 4.7.0` · `XGBoost 3.4.1` · `joblib 1.5.3` |
| 시계열 | 직접 구현 (ARIMA · ADF · ACF/PACF — 외부 의존 없음) |
| 화면 | `Streamlit` (배포는 별도 [`dashboard/requirements.txt`](dashboard/requirements.txt) · torch 는 CPU wheel) |
| 노트북 | `nbformat 5.11.1` · `nbclient 0.11.0` · `ipykernel 7.3.0` |

버전 정본은 [`pyproject.toml`](pyproject.toml) 입니다.

> ⚠️ numpy 2.5 / pandas 3.0 은 상당히 앞선 버전이라 ML 라이브러리와 ABI 가 어긋날 수 있습니다.
> **2026-08-25 에 실제로 설치해 7종 전부 import 되는 것을 확인한 뒤** 이 조합을 고정했습니다.
> 임의로 올리지 마세요.

**학습을 어디서 돌리나** — LightGBM 학습·그리드, sklearn RF·보정·bootstrap, 전처리·피처·EDA 는
전부 **로컬**입니다(배포판 LightGBM 이 GPU 빌드가 아니고 나머지는 CPU 경로이며, 로컬이 22코어라
CPU 바운드 작업은 Colab 런타임보다 빠릅니다). 딥러닝(⑥ LSTM/GRU)만 Colab Pro 를 씁니다.

---

## 저장소 구조

```text
Alpha_Stack/
├── common/          3,838줄   설정·경로·자격증명·거래일·기업행위 판정 — 아무것도 import 하지 않는다
├── ingest/         16,151줄   바깥과 통신해 SQLite 하나에 쌓는다 (internal)
│   ├── clients/               외부 API 9종 — KRX·DART·ECOS·FRED·KOSIS·포털·예탁결제원·FDR·HF
│   └── store/                 원문 bytes 보존 · 워터마크 · 호출 예산
├── supply/          5,353줄   ★ as_of 정문 — 행 T 에 보이는 것은 known_at ≤ T 인 것뿐
│   └── holdout.py             봉인 입구 — reports/unseal.log 가 정확히 1줄일 때만 열린다
├── features/        3,291줄   시세 → 기술적 지표 → 학습 가능한 표 (조합 A~K)
├── models/          2,855줄   표 → 등락 방향 분류기 · 최종 C · K LogisticRegression
├── evaluation/      9,226줄   ★ 예측 → 믿어도 되는가 (워크포워드·기준선·MDD·Sharpe·거래비용)
├── backtest/        1,270줄   A · B · C 분할매매 · 비용 민감도
├── timeseries/      2,085줄   numpy 로 손수 짠 ARIMA·ADF·ACF/PACF — 통계적 기준선
├── pipelines/         960줄   refresh 8단계 버튼 갱신 · 잠금
├── dashboard/       9,603줄   Streamlit — Overview + 9페이지
├── scripts/        22,393줄   수집·반출·검사·문서 빌더 (run_final_holdout · upload_to_hf · check_*)
├── tests/          22,373줄   103파일 — 경계 시험이 계층 위반을 막는다
├── notebooks/        290권    01-데이터수집 9 · 02-품질·전처리 12 · 04-모델 257 · 06-검토·발견 12
├── docs/             463편    도메인 폴더 + versionN.N — 옛 버전을 지우지 않는다
├── sql/init/                  스키마 v16 · 표 23개
├── deploy/hf-space/           Hugging Face Space (Docker) 배포 정본
├── reports/                   결과 · unseal.log · 품질 원장
├── data/                      krx_cache.db 6,685 MB · sealed/ 봉인본  ← 둘 다 커밋하지 않는다
├── AGENTS.md                  팀 협업 규약 (Git·브랜치·코드 스타일)
├── CONTRIBUTING.md            PR 작성 규칙 7절 · TIL
└── .env.example               환경변수 견본
```

---

## 문서

> 👉 **먼저 [docs/README.md](docs/README.md) 를 봅니다** — 문서 인덱스이자 버전 규약입니다.
> `docs/` 는 **도메인 폴더 + `versionN.N`** 으로 관리합니다. 옛 버전은 지우지 않습니다 —
> 개정 이력이 곧 이 프로젝트의 발전 과정입니다.

| 문서 | 내용 |
|---|---|
| [문제정의 v2.0](docs/문제정의/version2.0/문제정의.md) | ★ **왜 만드나 · 무엇이 성공인가** |
| [요구사항 v2.0](docs/요구사항/version2.0/요구사항.md) | ★ **무엇을 만드나** — 기능 29건 · 데이터 요구사항 12건 |
| [시장조사 v2.0](docs/시장조사/version2.0/시장조사.md) | 근거 — 국내외 서비스 · 선행연구 · 한국 제도 제약 |
| [아키텍처 v1.3](docs/아키텍처/version1.3/아키텍처.md) | [시스템 구성도](docs/아키텍처/version1.3/시스템아키텍처.png) · [계층도](docs/아키텍처/version1.3/계층아키텍처.png) · [대조표](docs/아키텍처/version1.3/변경사항.md) |
| [기능명세 v2.0](docs/기능명세/version2.0/상장구간과_홀드아웃_개봉실행.md) | 상장 구간 판정과 홀드아웃 개봉·실행 (mermaid 4장) |
| [ERD v2.7](docs/erd/version2.7/ERD.md) | 데이터 모델 (v16 · 표 23개) · 정문 지도 |
| [핵심코드 v1.4](docs/핵심코드/version1.4/) | 줄 단위 해설 7편 — 코드를 안 열어 본 팀원용 |
| [결과 보고서 v1.2](docs/결과보고서/version1.2/) | 강사님 제출용 — 본문 10장 + 부록 4 · Word 51쪽 |
| [발표 v1.1](docs/발표/README.md) | [발표 최종통합본](docs/발표/version1.1/발표_최종통합본.html) · 데이터 파트 예상 질문 |
| [계획서 v6.0](docs/계획서/version6.0/) | 강사님께 내는 계획서 **최종본** |

<details>
<summary><b>파트별 문서와 살아 있는 운영 문서</b></summary>

| 문서 | 내용 |
|---|---|
| ★ [데이터파트 v4.6](docs/데이터파트/version4.6/요약본.md) · [명세서](docs/데이터파트/version4.6/명세서.md) | 닫을 것을 닫고, 한 번 연 판 · [홀드아웃 개봉 절차 §8](docs/데이터파트/version4.6/홀드아웃_개봉_절차.md) |
| [데이터 품질 규약](docs/데이터파트/version4.6/데이터_품질_규약.md) · [퀀트기준 대조](docs/데이터파트/version4.6/퀀트기준_대조.md) | 이상치·결측·자본변동·편향 · 표준에 넘은 것과 못 미치는 것 |
| [HF 배포 변경이력](docs/데이터파트/version4.6/HF_배포_변경이력.md) | HF 카드 `CHANGES.md` 정본 |
| [모델파트 v1.3](docs/모델파트/version1.3/최종결과-2년과-마지막-구간.md) | 최종 결과 둘 — 2년 확증 · 마지막 한 구간 |
| [평가파트 v1.4](docs/평가파트/version1.4/대시보드_9페이지와_배포.md) | 대시보드 9페이지와 배포 |
| [피처파트 v1.2](docs/피처파트/version1.2/파생_피처_4개와_시점_규칙.md) · [전처리 v1.4](docs/전처리/version1.4/전처리-설계-명세서.md) | 파생 피처와 시점 규칙 · 전처리 후보 여덟 전부 기각 |
| [docs/decisions/](docs/decisions/) | ADR — **번호가 곧 이력**이라 버전을 두지 않습니다 (최종 홀드아웃: 0009 · 0010) |
| [docs/회의안건/](docs/회의안건/) · [docs/TIL/](docs/TIL/) | 회의 안건과 결정 · 작업 기록(작성자별 폴더) |

</details>

문서를 옮기거나 버전 폴더를 새로 팔 때는 **반드시** 링크를 검사합니다 —
상대 경로가 전부 바뀌므로 사람 눈으로는 빠뜨립니다.

```bash
python scripts/check_doc_links.py      # 깨진 링크가 있으면 exit 1
```

---

## 기여와 문의

이 저장소는 **부트캠프 1차 팀 프로젝트**라 외부 기여를 받고 있지는 않습니다. 다만 코드를 읽다
이상한 곳을 발견하셨다면 [이슈](https://github.com/devlee328288/Alpha_Stack/issues)로 알려 주시면
고맙겠습니다.

**팀원이라면** 작업 전에 이 둘을 읽습니다.

- [**CONTRIBUTING.md**](CONTRIBUTING.md) — PR 본문 7절(Why·What·Who·How·Impact·Proof·Learn) ·
  ✅ 체크리스트 규칙 · TIL 작성법
- [**AGENTS.md**](AGENTS.md) — Git 흐름 · 브랜치 이름 · 커밋 메시지 · 코드 규약 · 절대 제약

짧게 줄이면 이렇습니다.

- **PUBLIC 입니다.** `.env` · API 키 · 데이터 원본을 커밋하지 않습니다
- **PR 로 올립니다.** `main` 직커밋 금지. 머지는 **사람이 GitHub 웹에서** 직접
- **`gh pr merge` 금지** — 계정 정지 이력이 있습니다
- **홀드아웃 봉인본(`data/sealed/`)은 올리지 않습니다** — HF 에는 결과만, 학습에 섞지 않습니다
- 파이썬은 `snake_case` · 4-space · line-length 100 · 주석과 문서는 한국어
- push 전 `git status --short` 로 PDF·ZIP·데이터·시크릿 혼입 확인

---

## 운영 시 유의사항과 면책

- **이 저장소는 교육·연구·포트폴리오 목적입니다.** 여기의 어떤 숫자도 **투자 자문이 아니며**,
  특정 종목의 매수·매도 권유가 아닙니다. 투자 판단과 그 결과의 책임은 전적으로 이용자 본인에게
  있습니다.
- **성과는 비용 차감 전입니다.** 슬리피지·호가 대기·체결 실패·세금을 담고 있지 않습니다.
  과거 자료에 대한 백테스트 성과는 미래 수익을 보장하지 않습니다.
- **원자료는 배포하지 않습니다.** KRX 자료는 재배포가 허용되지 않으므로 저장소에도 HF 공개본에도
  두지 않고, 팀 안에서만 HF **private** 로 주고받습니다. 이 저장소가 배포하는 것은 *자료를 만드는
  방법*이지 *자료*가 아닙니다.
- **대시보드는 화면 안에서 학습합니다.** 무료 CPU 배포라 느리거나 제한에 걸릴 수 있고, 결과 캐시를
  모든 접속자가 공유합니다. 공식 성적은 화면이 아니라 [수행결과](#수행결과)에서 읽으십시오.
- **수집 코드를 돌릴 때** 각 제공처의 이용약관·`robots.txt`·호출 한도를 반드시 확인하십시오.
  이 저장소는 KRX 업종 분류 자동 수집을 약관 때문에 하지 않고, `finance.naver.com` 은
  `robots.txt` 때문에 수집하지 않습니다.

---

## 라이선스

| 대상 | 조건 |
|---|---|
| **이 저장소의 코드·문서** | [MIT](LICENSE) |
| **수집 대상 자료** | MIT 가 **덮지 않습니다.** KRX·DART·공공데이터포털·한국은행 등 각 제공처의 이용조건을 따르며, **원자료는 이 저장소가 배포하지 않습니다** |
| **HF private 데이터셋** | 팀 내부 전달용 — 공개 재배포하지 않습니다 |

## 감사의 말

- 주제와 방향을 잡아 주신 **강사님**, 그리고 AI 퀀트 4기 동료들
- [FinanceDataReader](https://github.com/FinanceData/FinanceDataReader) · [pykrx](https://github.com/sharebook-kr/pykrx) —
  국내 시세 접근의 사실상 표준을 만들어 주신 분들
- [scikit-learn](https://scikit-learn.org) · [LightGBM](https://github.com/microsoft/LightGBM) ·
  [Streamlit](https://streamlit.io) · [Hugging Face](https://huggingface.co)
- 검증 설계에서 참고한 [*Machine Learning for Trading*](https://github.com/stefan-jansen/machine-learning-for-trading) 의
  evidence boundary 개념
