---
layout: post
title: FastAPI + LangGraph AI Agent self-hosted 모니터링 플랫폼 비교(Langfuse vs Arize Phoenix)
categories: [ai-engineering]
---

# FastAPI + LangGraph AI Agent 모니터링: Langfuse vs Arize Phoenix

## 1. Langfuse vs Arize Phoenix 선택 기준

### LangGraph 연동 방식

Langfuse는 콜백 핸들러 방식으로 LangGraph와 연동합니다.

```python
from langfuse.callback import CallbackHandler

handler = CallbackHandler()
graph.invoke(input, config={"callbacks": [handler]})
```

Phoenix는 OpenInference의 `LangChainInstrumentor`를 사용합니다. LangGraph는 별도의 instrumentation 패키지가 없으며, `langchain-core`를 공유 기반으로 사용하기 때문에 LangChain instrumentation으로 커버됩니다.

```python
from openinference.instrumentation.langchain import LangChainInstrumentor
from phoenix.otel import register

tracer_provider = register(project_name="my-llm-app", auto_instrument=True)
```

### 기능 비교

| 항목 | Langfuse | Arize Phoenix |
|------|----------|---------------|
| LangGraph 공식 지원 | 콜백 핸들러 | LangChain instrumentor 경유 |
| Self-host 구성 요소 | 6개 (web, worker, PG, ClickHouse, Redis, Minio) | 1~2개 (앱 + DB) |
| 프롬프트 버전 관리 | 내장 | 없음 |
| LLM 비용 자동 집계 | O | O (SpanCostCalculator) |
| RAG 평가 | O | O (강점) |
| 수신 프로토콜 | 자체 HTTP API | OTLP 표준 (HTTP / gRPC) |
| DB | ClickHouse + PostgreSQL | SQLite 또는 PostgreSQL |
| 외부 큐 | Redis (BullMQ) | In-memory Queue |

---

## 2. 기본 개념 정리

### Trace와 Span

- **Trace**: 유저 요청 하나의 전체 흐름
- **Span**: 그 안의 개별 작업 단위 (LLM 호출, 툴 호출, DB 조회 등)

각 Span은 시작/종료 시간, 소요 시간, 성공/실패 여부, 입출력 메타데이터를 담습니다.

LangGraph 에이전트를 실행하면 노드 하나하나가 Span으로 기록됩니다.

### Ingestion

SDK에서 모니터링 서버로 trace/span 데이터를 전송하는 행위 자체를 ingestion이라고 합니다.

---

## 3. Langfuse Self-hosted 아키텍처

### 구성 요소

Langfuse v3 self-hosted는 6개의 서비스로 구성됩니다.

```
├── langfuse-web     (port 3000 — UI + API)
├── langfuse-worker  (port 3030 — 비동기 처리)
├── ClickHouse       (8123, 9000 — OLAP 분석 DB)
├── MinIO / S3       (9090 — 오브젝트 스토리지)
├── Redis            (6379 — 큐 + 캐시)
└── PostgreSQL       (5432 — 트랜잭션 DB)
```

### 각 구성 요소의 역할

| 구성 요소 | 역할 |
|----------|------|
| **langfuse-web** (Next.js) | UI 콘솔 서빙, 모든 API 처리 |
| **langfuse-worker** (Express) | 비동기 이벤트 처리, LLM-as-a-Judge, 배치 export |
| **PostgreSQL** | 유저, 조직, 프로젝트, API 키, 프롬프트 등 트랜잭션 데이터 |
| **ClickHouse** | trace, observation, score 등 대용량 로그 데이터 |
| **Redis** | BullMQ 이벤트 큐 + API 키/프롬프트 캐시 |
| **S3 / MinIO** | raw 이벤트 원본, 멀티모달 미디어, 배치 export 파일 |

### 데이터 흐름

```
SDK
 1. HTTP POST /api/public/ingestion
langfuse-web
 2. S3에 raw 이벤트 저장
 3. Redis에 S3 참조(경로)만 큐에 넣음
 4. 207 반환(HTTP Response)
langfuse-worker
 5. Redis에서 S3 참조 꺼냄
 6. S3에서 실제 데이터 읽어서 enriching (토큰/비용 계산 등)
 7. ClickHouse에 씀
```

핵심은 **web이 DB에 직접 쓰지 않는다**는 것입니다. Redis 큐에는 S3 경로(참조)만 넘기고, 실제 데이터는 S3에 있습니다. 수신과 저장을 분리해서 트래픽이 몰려도 web이 죽지 않는 구조입니다.

### langfuse-web vs langfuse-worker 분리 이유

Langfuse v3는 Event-Driven 백엔드 아키텍처를 채택했습니다. SDK에서 오는 HTTP 요청을 즉시 수신한 뒤 큐에 넣고 비동기로 처리합니다. 이를 통해 DB에 부하를 주지 않고 더 많은 요청을 처리할 수 있습니다.

> v2는 단일 컨테이너 + PostgreSQL만으로 구성되었으나, 수백만 row의 tracing 데이터에서 병목이 발생해 v3에서 ClickHouse, Redis, S3, worker 컨테이너가 추가됐습니다.

### S3 / MinIO 사용 목적

Langfuse는 S3를 세 가지 용도로 사용합니다.

1. **Raw 이벤트 저장 (필수)**: SDK에서 들어오는 이벤트 원본을 그대로 저장. 모든 배포에서 필수 설정입니다. ClickHouse가 일시적으로 불가해도 이벤트 유실을 방지하는 복구용 백업이기도 합니다.
2. **멀티모달 입출력 (선택)**: 이미지, 오디오 등 base64 인코딩된 미디어를 trace에 포함할 경우 SDK가 자동으로 S3에 업로드하고 trace에는 참조 문자열만 남깁니다.
3. **배치 export (선택)**: UI에서 CSV/JSON 대용량 내보내기 시 사용합니다.

MinIO는 S3 API와 호환되는 오픈소스 오브젝트 스토리지 서버로, AWS 환경이라면 MinIO 없이 S3를 그대로 사용할 수 있습니다. Docker Compose / Helm 배포에서는 기본값으로 MinIO가 포함됩니다.


### S3 데이터 삭제 정책

| 데이터 종류 | 삭제 시점 |
|------------|----------|
| Raw 이벤트 (text) | S3 lifecycle 정책으로 직접 설정 (기본값 없음, Langfuse Cloud는 30일) |
| 멀티모달 미디어 | 정책 설정 권장 안 함 (삭제 시 UI 참조 깨짐) |
| ClickHouse 데이터 | Data Retention 기능으로 프로젝트별 설정 (최소 3일) |

Data Retention 기능을 활성화하면 이벤트 만료 시 `blob_storage_file_log` 테이블을 참고해 S3의 해당 파일도 함께 삭제합니다.

### Redis 역할 상세

- **BullMQ 이벤트 큐**: web이 받은 이벤트의 S3 참조를 worker에게 전달
- **API 키 캐시**: 모든 API 호출마다 DB를 조회하지 않도록 인메모리 캐싱(보통 환경변수에 적는 API KEY입니다.)
- **프롬프트 캐시**: 자주 사용되는 프롬프트를 read-through 캐시로 빠르게 제공

---

## 4. Arize Phoenix Self-hosted 아키텍처

### 구성 요소

Phoenix는 **단일 컨테이너**로 구성됩니다.

```
├── Phoenix 컨테이너 (port 6006 — UI + HTTP OTLP)
│                   (port 4317 — gRPC OTLP)
└── SQLite 또는 PostgreSQL
```

### 데이터 흐름

```
SDK (openinference instrumentation)
 1. OTLP (HTTP: 6006/v1/traces 또는 gRPC: 4317)
Phoenix 단일 컨테이너
 2. In-memory Span Queue (최대 20,000개)
 3. BulkInserter (배치로 묶어서)
 4. SQLite 또는 PostgreSQL
```

Span Queue가 20,000개 한도에 도달하면 새로운 span 요청에 HTTP 429를 반환합니다.

### 스토리지

- **SQLite**: 기본값. 로컬 개발 및 소규모 배포에 적합.
- **PostgreSQL**: 프로덕션 권장. 동시 접근, 백업/복제 등 표준 DB 도구 활용 가능.

---

## 5. 구조 비교 요약

| 항목 | Langfuse | Phoenix |
|------|----------|---------|
| 컨테이너 수 | 6개 | 1개 |
| 큐 방식 | Redis BullMQ (외부) | In-memory (내장, 20k 한도) |
| 분석 DB | ClickHouse | 없음 (PostgreSQL만 사용) |
| 오브젝트 스토리지 | S3 / MinIO (필수) | 없음 |
| 수신 프로토콜 | 자체 HTTP API | OTLP 표준 (HTTP / gRPC) |
| 운영 복잡도 | 높음 | 낮음 |
| 대규모 트래픽 대응 | 유리 (이벤트 드리븐 + ClickHouse) | 불리 (큐 20k 한도) |
| Self-host 시작 난이도 | 높음 | 낮음 (컨테이너 1개) |

---

## 6. 결론

**FastAPI + LangGraph 실서비스 기준:**

- 운영 안정성과 실시간 모니터링 & 대규모 트래픽이 중요하다면 → **Langfuse**
- 빠르게 띄우고 RAG 평가에 집중하고 싶다면 → **Phoenix**
- 이미 OpenTelemetry 인프라가 있다면 → **Phoenix**가 자연스럽게 연동됨(OTLP)

두 도구는 목적이 다르므로 병행 사용도 가능합니다. Phoenix로 로컬/스테이징 디버깅, Langfuse로 프로덕션 운영 모니터링을 나누는 방식도 실용적입니다.

개인적으로는 Self-hosted로 둘 다 띄워봤을 때, UI/UX적으로 Langfuse가 통계 그래프가 많이 제공되어 더 좋았습니다.

---

## 참고

- [Langfuse 공식 아키텍처 문서](https://langfuse.com/handbook/product-engineering/architecture)
- [Langfuse Self-hosting](https://langfuse.com/self-hosting)
- [Langfuse S3 / Blob Storage 설정](https://langfuse.com/self-hosting/deployment/infrastructure/blobstorage)
- [Langfuse v2 to v3 업그레이드 가이드](https://langfuse.com/self-hosting/upgrade/upgrade-guides/upgrade-v2-to-v3)
- [Phoenix Self-hosting 아키텍처](https://arize.com/docs/phoenix/self-hosting/architecture)
- [Phoenix Tracing & Observability](https://deepwiki.com/Arize-ai/phoenix/5.1-tracing-and-observability)
- [OpenInference LangChain Instrumentation](https://github.com/Arize-ai/openinference/tree/main/python/instrumentation/openinference-instrumentation-langchain)