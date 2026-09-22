# 개미 리서치 (Ant Research) 개발 기획서

> 목적: 뉴스와 주가를 동일 시간축에서 탐색하고, 과거 시점의 공개 정보만으로 판단을 연습하게 하는 교육용 웹 서비스. 이 문서는 개발자와 후속 AI 에이전트가 구현 범위, 데이터 규칙, API 계약, 품질 기준을 같은 방식으로 해석하도록 하는 기준 문서다.

## 1. 제품 원칙과 범위

- **교육 도구**: 매수·매도 추천, 목표가, 수익 보장은 제공하지 않는다. 뉴스와 가격의 동시성은 인과관계로 표현하지 않는다.
- **시간 누수 제로**: 시뮬레이션 기준일 이후의 시세, 뉴스, 공시, 파생 지표는 API 응답 자체에서 제외한다. UI의 숨김 처리만으로는 충분하지 않다.
- **근거 우선 AI**: AI는 기사 묶음의 요약과 관련도 보조 판단에만 사용한다. 가격·점수·판정 결과는 결정론적 서버 로직으로 계산한다.
- **MVP 범위**: KOSPI 시가총액 상위 30개 종목, 일봉, 1M/3M/6M/1Y, 한국어 뉴스, 비로그인 탐색 및 로그인 사용자 복기 기록. 실시간 시세·자동 매매·개별 투자 조언은 제외한다.

### 사용자 흐름

1. 사용자가 종목과 기간을 고른다.
2. 가격·거래량 차트에 관련 이슈 마커와 핵심 변동 구간을 함께 본다.
3. 이슈를 열어 기사 묶음, 날짜별 전개, 키워드, 원문 링크, 관계 유형을 확인한다.
4. 블라인드 기본 시뮬레이션에서 기준일까지의 정보만 보고 N거래일 후 방향/예상 가격을 제출한다.
5. 실제 결과·후속 이슈를 공개하고, 근거와 결과를 저장해 반복 오답 유형을 보여준다.

## 2. 권장 저장소 구조

현재 저장소 구조가 확인되지 않았으므로 다음을 목표 구조로 사용한다. 기존 Next.js 또는 Python 프로젝트가 있다면 동일 책임의 디렉터리에 병합한다.

```text
apps/
  web/                 # Next.js App Router, 차트/시뮬레이션 UI
  api/                 # FastAPI, 공개 조회·세션·판정 API
packages/
  contracts/           # OpenAPI 생성 타입 또는 공유 JSON Schema
  ui/                  # 공용 컴포넌트·디자인 토큰
workers/
  ingestion/           # BigKinds/종목/공시 수집 및 정규화
  enrichment/          # NER, 임베딩, 군집화, 관계 판정, 요약 검수
infra/
  docker/              # 개발용 PostgreSQL+pgvector, Redis
  migrations/          # Alembic SQL migration
docs/
  ANTLAB_개미리서치_개발기획서.md
```

## 3. 기술 선택

| 영역 | 선택 | 이유 |
|---|---|---|
| Web | Next.js + TypeScript + Tailwind CSS | 서버 렌더링, 타입 안정성, 빠른 배포 |
| 차트 | TradingView Lightweight Charts | 일봉·거래량·마커·줌/팬에 적합 |
| API | Python 3.12 + FastAPI + Pydantic v2 | 데이터·ML 파이프라인과 언어 통일, OpenAPI 자동화 |
| DB | PostgreSQL 16 + pgvector | 관계형 무결성과 벡터 유사도 검색 동시 충족 |
| ORM/마이그레이션 | SQLAlchemy 2 + Alembic | migration 중심 스키마 관리 |
| 배치 | Python worker + EventBridge Scheduler 또는 GitHub Actions cron | 월 갱신과 재실행 추적 |
| 분석 | KPF-BERT NER, kpfSBERT, HDBSCAN | 한국어 뉴스 개체·유사 기사 군집화 |
| 생성 AI | 구조화 출력 가능한 LLM API | 이슈 타임라인 요약/관계 재평가; 원문 근거 필수 |
| 인프라 | Vercel(web), AWS ECS Fargate/API·worker, RDS PostgreSQL, S3 | 프런트 분리 배포와 장기 배치 실행 |

외부 데이터 어댑터는 반드시 인터페이스로 감싼다. `yfinance`는 개발/보조 데이터에 편리하지만 한국 시장의 운영용 원천으로 단독 의존하지 말고, KRX 허용 API 또는 라이선스가 확인된 시세 공급자를 운영 어댑터로 교체 가능하게 한다.

## 4. 데이터 모델

주요 테이블의 PK는 UUID, 모든 시간은 UTC로 저장하고 화면에서 KST로 변환한다. 날짜 기반 시세는 `trading_date`(KST 거래일)를 별도로 둔다.

| 테이블 | 핵심 열 | 용도 |
|---|---|---|
| `stocks` | code, name, market, sector, active | 종목 마스터 |
| `stock_aliases` | stock_id, alias, alias_type | 기업명·브랜드·자회사 검색 사전 |
| `articles` | source_article_id, publisher, published_at, title, body, url, categories, embedding | 원문 메타·본문, 원문 ID UNIQUE |
| `article_entities` | article_id, entity_text, entity_type, confidence, location | NER 결과 |
| `issues` | id, started_at, ended_at, title, timeline_summary, keywords, risk_flag, embedding | 동일 사건 기사 묶음 |
| `issue_articles` | issue_id, article_id, representative_rank | 이슈-기사 연결 |
| `stock_issue_relations` | stock_id, issue_id, relation_type, score, evidence, reviewed_at | 직접/경쟁사/공급망 관계 |
| `news_volume_daily` | stock_id, date, article_count, publisher_count, z_score | 보도량 급증 판정 입력 |
| `financial_disclosures` | stock_id, receipt_date, report_type, facts_json, source_url | 접수일 기준 공개 재무 정보 |
| `learning_cases` | stock_id, cutoff_date, horizon_days, selection_reason, status | 재사용 가능한 학습 케이스 |
| `simulation_sessions` | user_id nullable, case_id, issued_at, expires_at | 노출 범위가 고정된 세션 |
| `predictions` | session_id, predicted_price, direction, evidence_issue_ids, note, result | 제출 및 복기 |

`stock_issue_relations`에는 `relation_type IN ('DIRECT','COMPETITOR','SUPPLY_CHAIN')`, `score`(0~1), 근거 기사 ID와 근거 문장을 저장한다. 이 테이블만이 차트에 마커를 노출하는 권한 원천이다.

## 5. 수집·분석 파이프라인

### 5.1 월 배치

1. KRX 종목 기본정보로 활성 종목과 별칭 사전을 갱신한다.
2. BigKinds `/search/news`에서 지난 1개월 기사와 보정 구간을 수집한다. `source_article_id` 기준 upsert로 중복을 제거한다.
3. 제목과 첫 문단에 KPF-BERT NER를 적용하고 기업·인물·기관·상품 개체를 추출한다.
4. kpfSBERT 임베딩과 제목/핵심 개체를 이용해 7일 시간 창 안에서 HDBSCAN 군집화를 수행한다. 노이즈 기사는 보류 큐에 둔다.
5. 각 군집의 대표 기사·날짜별 기사 목록만 LLM에 전달해 `title`, `timeline`, `keywords`, `claims[]` JSON을 생성한다. `claims`의 수치·고유명사는 원문에서 문자열 또는 정규화된 값으로 대조하고 실패 시 재시도/검수 큐로 보낸다.
6. 직접 명칭, 산업/키워드 BM25, 임베딩 top-k 후보를 합쳐 종목 후보를 만든다. LLM 재평가는 후보와 증거 문장만 받아 `relevant`, `relation_type`, `score`, `rationale`를 구조화 출력한다. 임계값 미만은 저장하지 않는다.
7. `/time_line` 일별 보도량으로 60거래일 rolling z-score를 계산한다. z-score 임계값은 초기 2.0으로 두고 운영 데이터에서 조정한다.
8. 품질 리포트, 실패 건, 실행 manifest(입력 기간·모델 버전·프롬프트 버전·건수)를 저장한다.

### 5.2 요청 시 계산

- 가격 데이터는 `MarketDataProvider`에서 조회하고 거래일 캘린더로 정렬한다.
- 장 마감 후 발행 기사는 다음 거래일에 `display_date`를 매핑한다. 장중/마감 판정은 KRX 거래시간과 KST를 기준으로 한 단일 함수에서 수행한다.
- 변곡점은 종가의 변화율, 20일 실현 변동성, 거래량 z-score를 입력으로 하는 결정론적 함수로 계산한다. 보도 급증일과 ±3거래일 안에 겹친 구간만 `highlighted=true`로 반환한다.

## 6. 미래 데이터 차단 규격

기준 시점 `cutoff_date`는 **해당 거래일 장 시작 직전**이다. 따라서 노출 가능 정보는 `cutoff_date` 전 거래일 15:30 KST 이후부터 `cutoff_date` 장 시작 전까지 발행된 기사까지이며, 재무는 `receipt_date < cutoff_date`만 허용한다.

`GET /v1/simulations/{id}`는 아래만 반환한다.

```json
{
  "cutoffDate": "2025-03-10",
  "series": [{"time":"2025-03-07","open":0,"high":0,"low":0,"close":0,"volume":0}],
  "issues": [{"displayDate":"2025-03-10","id":"...","title":"...","summary":"..."}],
  "financials": [{"receiptDate":"2025-02-28","facts":{}}],
  "horizonTradingDays": 10
}
```

미래 데이터는 빈 값·마스킹 값·암호화된 값으로도 전송하지 않는다. API 테스트는 의도적으로 미래 기사 제목과 미래 종가를 삽입한 fixture에서 응답 문자열 전체에 그 값이 없는지 검증한다.

## 7. API 계약

| Method / path | 책임 | 핵심 응답 |
|---|---|---|
| `GET /v1/stocks?q=` | 종목 자동완성 | code, name, market |
| `GET /v1/chart?stockCode=&from=&to=` | 탐색 차트 | OHLCV, issues, highlights, market index |
| `GET /v1/issues/{issueId}` | 이슈 상세 | 요약, 키워드, 기사 목록, 관계·근거 |
| `POST /v1/simulations` | 블라인드 세션 생성 | sessionId, cutoffDate, 허용 데이터 |
| `GET /v1/simulations/{id}` | 고정 세션 조회 | 기준일 이전 데이터만 |
| `POST /v1/simulations/{id}/submit` | 예측 판정 | predicted direction/price, actual result, explanation data |
| `GET /v1/reviews/me` | 복기·취약 유형 | 오답 유형, 관계 유형별 적중률 |

`POST /submit` 요청은 `predictedPrice`, `evidenceIssueIds[]`, `note`만 받는다. 실제 방향은 `actual_return = close[t+N] / close[t] - 1`으로 계산한다. `abs(actual_return) <= 60거래일 일수익률 표준편차`면 `FLAT`, 그 외 부호에 따라 `UP`/`DOWN`이다. 서버가 동일 규칙으로 제출 가격의 예상 방향도 계산해 적중 여부를 저장한다.

## 8. 화면·디자인 명세

### 공통

- 투자 판단 UI가 아닌 학습 서비스라는 문구와 “과거 사례 기반, 투자 조언 아님” 고지를 항상 표시한다.
- 한국어 우선, 숫자는 천 단위 구분과 KST 날짜 표기, 색만으로 관계·상승·하락을 구분하지 않는다.
- 키보드 탐색, 마커 aria-label, 4.5:1 이상 대비, 모바일 360px 대응을 필수로 한다.

### 탐색 화면

- 상단: 종목 검색, 기간 프리셋, 블라인드 모드 전환.
- 본문 좌측(데스크톱 70%): 일봉 캔들, 거래량, 핵심 구간 배경 강조, 뉴스 마커. 색+아이콘으로 직접(파랑/연결), 경쟁사(보라/비교), 공급망(주황/사슬)을 구분한다.
- 본문 우측: 선택 이슈 패널 - 한 줄 결론이 아니라 “무슨 일이 있었나 / 언제 / 관련 근거 / 이후 가격 흐름”을 분리한다. 원문 URL과 기사 발행 시각을 명시한다.
- 모바일: 차트 아래 bottom sheet로 상세 패널을 전환한다.

### 시뮬레이션 화면

- 종목명·정확한 날짜는 기본 블라인드 처리하되, 사용자가 접근성 또는 학습 목적상 해제할 수 있게 한다.
- 기준일을 세로선으로 표시하고 이후 영역은 API에 존재하지 않는 상태로 렌더링한다.
- 예측은 수치 입력과 추세선 드래그 둘 다 지원한다. 제출 전 근거 이슈를 최소 1개 선택하도록 유도하되 강제하지 않는다.
- 결과 화면은 실제 차트 공개 → 수익률/방향 판정 → 선택 근거 대비 → 다음 학습 제안 순서다. AI가 “왜 올랐다”고 단정하는 문구를 쓰지 않는다.

## 9. 구현 단계와 완료 조건

| 단계 | 산출물 | 완료 조건 |
|---|---|---|
| 0. 기반 | monorepo, env schema, Docker, CI, DB migration | 빈 환경에서 한 명령으로 web/API/DB 기동 |
| 1. 수집 | 종목·기사 정규화, 재실행 가능한 batch | 중복 없이 30개 종목 최근 3개월 적재 |
| 2. 이슈화 | NER, 임베딩, 군집, 관계 저장 | 수동 표본 200건 오탐률 10% 이하 |
| 3. 탐색 | 차트 결합 API, 마커, 상세 패널 | 최초 렌더 3초 이내, 상세 1초 이내 |
| 4. 시뮬레이션 | 차단 API, 제출/판정, 결과 공개 | 미래 데이터 노출 자동 테스트 0건 |
| 5. 복기 | 로그인, 기록, 취약 유형 집계 | 동일 세션 중복 제출·변조 방지 |
| 6. 운영 | 관측성, 관리자 검수 큐, 배포 | 월 배치 10시간 이내, 실패 알림 |

## 10. 테스트·관측성·보안

- **단위 테스트**: 거래일 매핑, 장 마감 후 기사 규칙, 변곡점, 방향/횡보 판정, cutoff 필터.
- **통합 테스트**: BigKinds·DART·시세 provider는 VCR/fixture로 격리한다. 네트워크·API 키가 없는 CI에서도 실행돼야 한다.
- **E2E**: 종목 검색 → 마커 상세 → 시뮬레이션 생성 → 예측 제출 → 결과 복기. Playwright로 데스크톱과 모바일 1종씩 검증한다.
- **데이터 품질**: 원문 ID 중복률, 관계 유형별 표본 정확도, 요약 claim 대조 통과율, 군집 크기 분포를 매 배치 기록한다.
- **SLO**: 차트+마커 3초, 상세 1초, 세션 생성 2초, 월 배치 10시간 이내. 기준일 이후 데이터 노출은 0건.
- **보안**: 모든 키는 Secret Manager/Vercel env로 주입하고 저장소에 커밋하지 않는다. 원문은 이용 약관·저작권 정책을 준수해 필요한 범위만 저장하며, 사용자 기록에는 최소 개인정보만 둔다.
- **관측성**: request ID, provider latency, 배치 단계별 건수/실패율, LLM 모델·프롬프트 버전, 비용 추정치를 구조화 로그와 대시보드에 남긴다.

## 11. AI 에이전트 구현 가드레일

1. API·DB·화면을 동시에 대규모로 만들지 말고 migration → 계약 테스트 → API → UI 순으로 작은 PR을 만든다.
2. 새 외부 API 호출은 provider interface, timeout, retry, rate limit, fixture를 반드시 함께 추가한다.
3. LLM 출력은 Pydantic JSON schema로 검증하고, 검증 실패/근거 부족 결과를 사용자에게 노출하지 않는다.
4. `cutoff_date` 필터는 프런트가 아닌 repository query와 serializer 양쪽에서 강제하고, 필터 없는 raw table 접근을 금지한다.
5. 금융 정보의 표현은 사실·출처·시점을 분리한다. “영향을 줬다” 대신 “해당 시점에 함께 관찰됐다”를 기본 문구로 사용한다.
6. 각 PR에는 migration 영향, 데이터 backfill 여부, feature flag, 롤백 절차, 테스트 증거를 포함한다.

## 12. 출시 판단 체크리스트

- [ ] 데이터 공급자별 이용 약관과 기사 원문 보관/링크 정책을 법무 또는 주최 측 기준으로 확인했다.
- [ ] 상위 30개 종목의 별칭 사전과 관계 판정 표본 검수를 마쳤다.
- [ ] 모든 시뮬레이션 엔드포인트의 미래 데이터 비노출 테스트가 통과했다.
- [ ] AI 요약의 수치·고유명사 검수 통과율이 95% 이상이다.
- [ ] 성능 SLO와 장애 알림을 스테이징·운영 환경에서 검증했다.
- [ ] 투자 조언 아님 고지, 출처 링크, 개인정보·삭제 정책을 배포 화면에 반영했다.

