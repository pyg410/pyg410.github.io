---
layout: post
title: Langfuse & Phoenix SDK 동작 원리 및 FastAPI 성능 영향 & 배포시 주의사항
categories: [ai-engineering]
---

FastAPI + LangGraph 실서비스에 Langfuse 또는 Phoenix를 붙일 때 내부 동작을 이해하고 있으면 예상치 못한 문제를 피할 수 있습니다.
<br>이 글은 두 SDK의 내부 구조, FastAPI에 미치는 성능 영향, 그리고 Gunicorn/Uvicorn 배포 환경에서의 주의사항을 정리합니다.

---

## 1. SDK는 FastAPI에 부하를 주는가?

### Langfuse Python SDK

Langfuse SDK는 **백그라운드 스레드 + 내부 큐** 방식으로 동작합니다.

> "It uses a worker Thread and an internal queue to manage requests to the Langfuse backend asynchronously. Hence, the SDK adds only minimal latency to your application."
>
> — [Langfuse Python SDK 공식 문서](https://langfuse.com/docs/sdk/python)

콜백이 호출되면 span 데이터를 내부 큐에 넣고 즉시 리턴합니다. 실제 네트워크 전송은 별도의 백그라운드 스레드가 담당합니다.

```
FastAPI 요청 처리 (메인)
    ↓ span 생성 → 내부 큐에 넣고 즉시 리턴
    ↓ 응답 반환

백그라운드 스레드 (별도)
    ↓ 큐에서 꺼내서 Langfuse 서버로 배치 전송
```

백그라운드 스레드의 전송 트리거 조건은 두 가지입니다.

| 조건 | 파라미터 | 기본값 |
|------|---------|--------|
| 이벤트 개수 | `flush_at` | 512개 |
| 시간 경과 | `flush_interval` | 5초 |

둘 중 먼저 충족되는 조건에 따라 배치 전송이 트리거됩니다.

> 참고: [Langfuse Event Queuing/Batching 문서](https://langfuse.com/docs/observability/features/queuing-batching)

### Phoenix (OpenTelemetry BatchSpanProcessor)

Phoenix는 OpenTelemetry의 `BatchSpanProcessor`를 사용합니다.

> "The BatchSpanProcessor is the recommended implementation for production. It buffers spans in a queue and exports them in batches using a background worker thread."
>
> — [OpenTelemetry Python SDK DeepWiki](https://deepwiki.com/open-telemetry/opentelemetry-python/3.3-spanprocessor-and-pipeline)

트리거 조건은 환경변수로 제어합니다.

| 조건 | 환경변수 | 기본값 |
|------|---------|--------|
| span 개수 | `OTEL_BSP_MAX_EXPORT_BATCH_SIZE` | 512개 |
| 시간 경과 | `OTEL_BSP_SCHEDULE_DELAY` | 5000ms (5초) |

> **주의**: Phoenix `register()` 함수의 `batch` 파라미터 기본값은 `False`입니다. 프로덕션에서는 반드시 `batch=True`를 명시해야 합니다. 그렇지 않으면 `SimpleSpanProcessor`가 동작해서 span이 끝날 때마다 **동기로 즉시 전송**되어 FastAPI 응답 시간에 직접 영향을 줍니다.

```python
# 프로덕션 권장 설정
from phoenix.otel import register

tracer_provider = register(
    project_name="my-app",
    batch=True,  # 반드시 명시
    auto_instrument=True,
)
```

> 참고: [arize-phoenix-otel PyPI](https://pypi.org/project/arize-phoenix-otel/)

---

## 2. GIL과 백그라운드 스레드

Python의 GIL(Global Interpreter Lock)로 인해 스레드가 여러 개여도 CPU 작업은 한 번에 하나만 실행됩니다. 하지만 백그라운드 스레드가 하는 작업 대부분은 네트워크 전송 대기(I/O)이며, GIL은 I/O 대기 중에는 자동으로 해제됩니다. 실제 CPU를 사용하는 구간(직렬화 등)만 GIL 경합이 발생하는데, 이 구간은 매우 짧습니다.

이론적으로 나노초 수준의 오버헤드가 누적될 수 있지만, FastAPI를 `uvicorn`으로 운영하는 async 환경에서는 GIL 경합 자체가 훨씬 줄어들어 체감 성능 저하로 이어지는 경우는 드뭅니다.

---

## 3. 백그라운드 스레드 구현

### Langfuse SDK (v3/v4)

Langfuse SDK v3/v4는 span 전송을 위해 OpenTelemetry의 `BatchSpanProcessor`를 상속한 `LangfuseSpanProcessor`를 사용합니다.

```python
# langfuse/_client/span_processor.py
from opentelemetry.sdk.trace.export import BatchSpanProcessor

class LangfuseSpanProcessor(BatchSpanProcessor):
    """Langfuse 전용 필터링 및 인증을 추가한 BatchSpanProcessor 확장"""
    ...
```

`BatchSpanProcessor`는 내부적으로 Python 표준 `threading`과 `collections.deque`를 사용합니다.

백그라운드 스레드는 두 가지 역할로 분리되어 있습니다.

- **span/trace 전송**: `LangfuseSpanProcessor` (OTel `BatchSpanProcessor` 상속)
- **score ingestion, 미디어 업로드**: `LangfuseResourceManager`의 별도 consumer 스레드 (`threading` 직접 사용)

> 참고: [langfuse-python GitHub — span_processor.py](https://github.com/langfuse/langfuse-python/blob/main/langfuse/_client/span_processor.py)

### Langfuse SDK v2 (레거시)

v2는 구조가 완전히 다릅니다. OTel 기반이 아니라 자체 `TaskManager`가 `threading`으로 consumer 스레드를 직접 생성합니다.

```python
# langfuse/task_manager.py (v2)
def init_resources(self):
    for i in range(self._threads):
        ingestion_consumer = IngestionConsumer(...)
        ingestion_consumer.start()  # threading.Thread.start() 직접 호출
```

이 구조 때문에 Gunicorn `--preload` 환경에서 fork-unsafe 문제가 발생합니다. (자세한 내용은 5번 섹션 참고)

### OpenTelemetry BatchSpanProcessor

`BatchSpanProcessor` 자체는 얇은 래퍼이고, 실제 로직은 공통 클래스인 `BatchProcessor`에 위임됩니다. deque, 워커 스레드, 트리거 조건 모두 이 공통 클래스 안에 구현되어 있습니다.

```
opentelemetry-sdk/src/opentelemetry/sdk/
ㄴ trace/export/__init__.py         ← BatchSpanProcessor (래퍼)
ㄴ _shared_internal/__init__.py     ← BatchProcessor (실제 로직: deque, 워커 스레드)
```

> 참고: [OpenTelemetry Python SDK 소스코드 — `_shared_internal/__init__.py`](https://github.com/open-telemetry/opentelemetry-python/blob/main/opentelemetry-sdk/src/opentelemetry/sdk/_shared_internal/__init__.py)

---

## 4. 싱글톤 패턴과 스레드 초기화 시점

### Langfuse SDK

Langfuse SDK는 **싱글톤 패턴**을 사용합니다. `public_key`를 키로 하는 싱글톤으로 관리되며, 백그라운드 스레드는 `Langfuse()` 클라이언트가 **처음 초기화될 때 한 번** 생성됩니다.

```python
# 이 시점에 백그라운드 스레드 생성
langfuse = Langfuse(public_key="pk-lf-...", secret_key="sk-lf-...")

# 동일한 public_key로 재호출 시 기존 싱글톤 재사용, 스레드 새로 안 만듦
langfuse2 = Langfuse(public_key="pk-lf-...")  # 기존 인스턴스 반환
```

명시적으로 `Langfuse()`를 호출하지 않아도 `get_client()`를 처음 호출할 때 환경변수 기반으로 자동 초기화됩니다.

> 참고: [Langfuse Python SDK Overview](https://langfuse.com/docs/sdk/python/low-level-sdk)

### Phoenix (OpenTelemetry TracerProvider)

Phoenix는 Langfuse처럼 자체 싱글톤 패턴을 구현하지 않습니다. 대신 OpenTelemetry의 **전역 TracerProvider** 메커니즘을 사용합니다.

`register()`를 호출하면 `TracerProvider`가 생성되고, 기본적으로 OpenTelemetry 전역 provider로 등록됩니다.

```python
from phoenix.otel import register

# TracerProvider 생성 + 전역 등록 + BatchSpanProcessor 워커 스레드 시작
tracer_provider = register(
    project_name="my-app",
    batch=True,
    set_global_tracer_provider=True,  # 기본값 True
)
```

전역 `TracerProvider`는 **한 번만 설정 가능**합니다. 이후 `set_tracer_provider()`를 다시 호출하면 경고 로그가 남고 무시됩니다. 따라서 `register()`는 앱 시작 시점에 **한 번만** 호출해야 합니다.

`BatchSpanProcessor`의 워커 스레드는 `TracerProvider`가 생성될 때 함께 시작됩니다. 종료 시에는 `tracer_provider.shutdown()`을 명시적으로 호출해야 워커 스레드가 정상 종료되고, 큐에 남은 span이 flush됩니다.

```python
# FastAPI lifespan에서 Phoenix(OTel) 종료 처리
from contextlib import asynccontextmanager
from fastapi import FastAPI
from opentelemetry import trace

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    # 종료 시 미전송 span flush 후 워커 스레드 종료
    trace.get_tracer_provider().shutdown()

app = FastAPI(lifespan=lifespan)
```

> 참고: [Phoenix OTEL register() 공식 문서](https://arize-phoenix.readthedocs.io/projects/otel/en/latest/api/register.html)

---

## 5. Gunicorn / Uvicorn 멀티 프로세스 환경에서의 주의사항

### Gunicorn pre-fork 모델

Gunicorn은 **pre-fork** 모델을 사용합니다. HTTP 요청을 받기 전에 마스터 프로세스가 워커를 미리 `os.fork()`로 생성합니다.

> "The pre in pre-forked means that the master process creates the workers before handling any HTTP request."
>
> — [Gunicorn 아키텍처 설명](https://medium.com/building-the-system/gunicorn-3-means-of-concurrency-efbb547674b7)

`os.fork()`는 부모 프로세스의 **메모리는 복사하지만 스레드는 복사하지 않습니다.** POSIX 표준에 따른 동작입니다.

### Gunicorn `--preload` 옵션

`--preload`는 **메모리 절약**을 위한 옵션입니다.

기본적으로 Gunicorn은 fork 후 각 워커가 앱 코드를 개별적으로 로드합니다. `--preload`를 사용하면 마스터 프로세스에서 앱을 먼저 로드한 뒤 fork하여, OS의 copy-on-write 덕분에 워커들이 메모리 페이지를 공유할 수 있습니다.

> "By preloading an application you can save some RAM resources as well as speed up server boot times."
>
> — [Gunicorn 공식 문서](https://docs.gunicorn.org/en/stable/settings.html)

| | 기본 (no preload) | `--preload` |
|--|--|--|
| 앱 로드 시점 | fork 후 각 워커가 개별 로드 | fork 전 마스터에서 한 번 로드 |
| 메모리 | 워커 수 × 앱 크기 | copy-on-write로 공유 → 절약 |
| 부팅 속도 | 느림 | 빠름 |
| `--reload`와 호환 | 가능 | 불가 |

### SDK 버전별 fork 안전성

| SDK | 버전 | span 전송 방식 | `os.register_at_fork` | `--preload` 안전성 |
|-----|------|--------------|----------------------|-------------------|
| Langfuse | **v2** | `TaskManager` 자체 `threading` | 없음 | **위험** |
| Langfuse | **v3/v4** | `BatchSpanProcessor` 상속 (`LangfuseSpanProcessor`) | 상속받음 | 안전 |
| Phoenix (OTel) | - | `BatchSpanProcessor` | 있음 | 안전 |

#### Langfuse v2의 문제

v2는 `TaskManager.__init__`에서 바로 consumer 스레드를 시작합니다. `--preload` 환경에서 전역으로 `Langfuse()`를 초기화하면 마스터 프로세스에서 스레드가 생성된 뒤 fork가 일어나고, 자식 프로세스에는 스레드가 없어서 이벤트가 전송되지 않습니다. 최악의 경우 `RuntimeError: can't start new thread`가 발생합니다.

> 실제 사례: [Langfuse GitHub Issue #3405](https://github.com/langfuse/langfuse/issues/3405)

```python
# X v2, --preload 환경에서 위험
langfuse = Langfuse()  # 마스터에서 초기화 → fork 후 워커에 스레드 없음

# O v2, --preload 환경에서 안전: post_fork 훅 활용
# gunicorn.conf.py
def post_fork(server, worker):
    import app as application
    application.langfuse = None         # 마스터에서 생성된 인스턴스 초기화
    application.langfuse = Langfuse()   # fork 이후 워커에서 새로 초기화
```

> v2는 현재 유지보수가 종료된 레거시 버전입니다. 근본적인 해결은 v3/v4로 업그레이드입니다.

#### Langfuse v3/v4와 Phoenix는 안전한 이유

`BatchSpanProcessor`는 내부적으로 `os.register_at_fork`를 사용해 fork 후 자식 프로세스에서 워커 스레드를 자동으로 재초기화합니다.

```python
# opentelemetry-sdk 내부 (_shared_internal/__init__.py)
if hasattr(os, "register_at_fork"):
    weak_reinit = weakref.WeakMethod(self._at_fork_reinit)
    os.register_at_fork(after_in_child=lambda: weak_reinit()())
```

Langfuse v3/v4의 `LangfuseSpanProcessor`는 `BatchSpanProcessor`를 상속하므로 이 보호가 자동으로 적용됩니다.

### Uvicorn `--workers`

Uvicorn은 `--workers` 옵션으로 멀티 프로세스를 띄울 때 Gunicorn과 달리 `os.fork()`가 아닌 Python `multiprocessing` 라이브러리의 **spawn** 방식을 사용합니다. `spawn`은 완전히 새로운 Python 인터프리터를 시작하므로 앱 코드를 처음부터 다시 실행합니다. 따라서 모든 버전의 Langfuse SDK와 Phoenix가 각 워커에서 새로 초기화되어 fork 문제가 발생하지 않습니다.

```
uvicorn main:app --workers 4
    ↓ spawn → 새 Python 인터프리터 × 4
    ↓ 각 워커에서 앱 코드 처음부터 실행
    ↓ SDK 각자 초기화 → 정상
```

### 배포 환경별 정리

| 환경 | 멀티 프로세스 방식 | Langfuse v2 | Langfuse v3/v4 | Phoenix |
|------|-----------------|-------------|----------------|---------|
| `uvicorn --workers N` | spawn | 안전 | 안전 | 안전 |
| `gunicorn` (기본) | os.fork (앱은 워커에서 로드) | 안전 | 안전 | 안전 |
| `gunicorn --preload` | os.fork (앱을 마스터에서 로드) | **위험** | 안전 | 안전 |

---

## 6. FastAPI + Langfuse / Phoenix 권장 설정

### Langfuse (v3/v4)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from langfuse import get_client

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    # 종료 시 미전송 이벤트 모두 flush 후 스레드 종료
    get_client().shutdown()

app = FastAPI(lifespan=lifespan)
```

> `shutdown()` 없이 프로세스가 종료되면 큐에 남은 이벤트가 유실될 수 있습니다.
>
> 참고: [Langfuse Python SDK 공식 문서](https://langfuse.com/docs/sdk/python)

### Langfuse v2 + Gunicorn `--preload`

```python
# gunicorn.conf.py
preload_app = True

def post_fork(server, worker):
    import app as application
    application.langfuse = None
    application.langfuse = application.get_langfuse()

def worker_exit(server, worker):
    import app as application
    if application.langfuse is not None:
        application.langfuse.flush()
```

### Phoenix

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from phoenix.otel import register
from opentelemetry import trace

tracer_provider = register(
    project_name="my-app",
    batch=True,  # 프로덕션에서 반드시 명시
)

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    # 종료 시 미전송 span flush 후 워커 스레드 종료
    trace.get_tracer_provider().shutdown()

app = FastAPI(lifespan=lifespan)
```

> `tracer_provider.shutdown()`을 호출하지 않으면 BatchSpanProcessor 워커 스레드가 살아 있는 상태로 프로세스가 종료되어 큐에 남은 span이 유실될 수 있습니다.

---

## 참고 문서

- [Langfuse Python SDK](https://langfuse.com/docs/sdk/python)
- [Langfuse Event Queuing/Batching](https://langfuse.com/docs/observability/features/queuing-batching)
- [Langfuse SDK — span_processor.py](https://github.com/langfuse/langfuse-python/blob/main/langfuse/_client/span_processor.py)
- [Langfuse GitHub Issue #3405 (v2 fork 문제)](https://github.com/langfuse/langfuse/issues/3405)
- [arize-phoenix-otel PyPI](https://pypi.org/project/arize-phoenix-otel/)
- [Phoenix OTEL register() 공식 문서](https://arize-phoenix.readthedocs.io/projects/otel/en/latest/api/register.html)
- [Phoenix Self-hosting Architecture](https://arize.com/docs/phoenix/self-hosting/architecture)
- [OpenTelemetry Python SDK `_shared_internal` 소스](https://github.com/open-telemetry/opentelemetry-python/blob/main/opentelemetry-sdk/src/opentelemetry/sdk/_shared_internal/__init__.py)
- [Gunicorn Design](https://gunicorn.org/design/)
- [Gunicorn Application Preloading](https://www.joelsleppy.com/blog/gunicorn-application-preloading/)
- [langfuse-python GitHub](https://github.com/langfuse/langfuse-python)