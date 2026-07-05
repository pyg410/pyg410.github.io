---
layout: post
title: Gunicorn --preload 환경에서 Langfuse SDK가 데이터를 조용히 잃어버리던 문제 (langfuse/langfuse-python#1658)
categories: [ai-engineering]
---

## TL;DR

Gunicorn을 `--preload` 옵션으로 띄우면 Langfuse Python SDK가 백그라운드로 보내야 할 trace/score/media 데이터를 **아무 에러도 없이 유실**하고, `flush()`를 호출하면 워커가 영원히 멈춰 있다가 타임아웃으로 죽는 문제가 있었다. 원인은 `os.fork()`가 "메모리는 복사하지만 스레드는 복사하지 않는다"는 POSIX 표준 동작 때문이었고, 이걸 고치는 과정에서 스레드 재시작 → 데드락 → 세그폴트까지 순서대로 만나며 7번의 커밋을 거쳤다.

---

## 1. Langfuse Python SDK는 내부적으로 어떻게 동작하는가

먼저 왜 이 버그가 생겼는지 이해하려면 SDK 내부 구조를 알아야 한다. `langfuse/_client/resource_manager.py`의 `LangfuseResourceManager`가 핵심이다.

- **싱글턴 패턴**: `public_key`(프로젝트 API 키)를 키로 하는 딕셔너리(`_instances`)에 인스턴스를 캐싱한다. `Langfuse()` 클라이언트를 여러 번 생성해도 같은 키라면 내부 리소스(스레드, 커넥션 풀 등)는 하나만 존재한다.
- 이 싱글턴이 초기화될 때(`_initialize_instance`) 다음 백그라운드 작업들이 함께 시작된다.
  1. **OTEL `TracerProvider` + `LangfuseSpanProcessor`**: 트레이스 span을 배치로 모아 export. 이건 OTEL의 `BatchSpanProcessor`를 상속하고 있고, OTEL 자체가 이미 `os.register_at_fork`를 등록해두고 있어서 **원래부터 fork-safe**했다.
  2. **`MediaUploadConsumer` 스레드(들)**: `media_upload_queue`를 소비하며 이미지/오디오 등 미디어를 업로드.
  3. **`ScoreIngestionConsumer` 스레드**: `score_ingestion_queue`를 소비하며 score/trace 업데이트 이벤트를 배치 전송.
  4. **`PromptCache`의 리프레시 컨슈머 스레드**: 프롬프트 캐시 갱신.
- 패턴은 전부 동일하다 — **큐에 넣는 쪽(producer, 메인 스레드)** 과 **큐를 소비해서 실제 HTTP 요청을 보내는 쪽(consumer, 별도 스레드)** 이 분리되어 있고, `flush()`는 `queue.join()`으로 "큐에 처리 중(unfinished)인 아이템이 0이 될 때까지" 블로킹 대기한다.

이 구조 자체는 흔한 producer-consumer 패턴이라 문제될 게 없다. 문제는 **프로세스가 fork될 때** 벌어진다.

## 2. 왜 하필 `gunicorn --preload`에서 터지는가

Gunicorn의 `--preload`는 애플리케이션 코드를 **마스터 프로세스에서 미리 import**한 뒤, 그 상태를 `os.fork()`로 복제해 워커 프로세스들을 만든다. 즉 순서가 이렇게 된다.

```
마스터 프로세스: import app → Langfuse() 싱글턴 생성 → 백그라운드 스레드들 시작
                        ↓ os.fork() × N
워커 프로세스 1, 2, 3 ...
```

여기서 POSIX 표준([`fork()` 문서](https://pubs.opengroup.org/onlinepubs/9699919799/functions/fork.html))의 핵심 함정이 있다.

> **`fork()`는 호출 시점의 메모리 상태(힙, 객체, 큐 안의 데이터, 락의 lock/unlock 상태, 소켓 파일 디스크립터)는 전부 복사한다. 하지만 스레드는 복사하지 않는다. 자식 프로세스에는 `fork()`를 호출한 그 스레드 하나만 남는다.**

그래서 워커(자식) 프로세스 입장에서는:
- `LangfuseResourceManager` 객체 자체는 메모리에 그대로 있다 (싱글턴 인스턴스, 큐 객체, `httpx.Client` 등 전부 "존재"는 한다).
- 하지만 그 큐를 소비하던 `MediaUploadConsumer`/`ScoreIngestionConsumer` **스레드는 자식에 없다**. 마스터 프로세스에만 살아있다.
- 워커에서 트레이스를 만들면 이벤트는 큐에 `put()`은 되지만, 아무도 그걸 꺼내서 처리(consume)하지 않는다.
- 이 상태에서 종료 시점에 `flush()` → `queue.join()`을 호출하면, "아직 처리 안 된 아이템이 있다"는 이유로 **영원히 리턴하지 않는다** → Gunicorn이 정한 워커 타임아웃에 걸려 `SIGABRT`로 강제 종료.

즉 두 가지 증상이 동시에 나타난다: **① 데이터가 에러 없이 조용히 사라짐(silent data loss)**, **② `flush()` 호출 시 워커가 멈추고 죽음**. span export는 OTEL이 이미 fork를 처리해줬기 때문에 멀쩡했고, 문제는 Langfuse SDK가 자체적으로 만든 나머지 스레드들이었다.

## 3. 해결 과정 : 7번의 커밋, 7번의 교훈

여기서부터가 이 PR의 진짜 내용이다. 처음엔 "스레드만 다시 띄우면 되겠지"였는데, 고치다 보니 계속 새로운 계층의 문제가 드러났다.

### Step 1 - `os.register_at_fork(after_in_child=...)`로 스레드 재시작

```python
if hasattr(os, "register_at_fork"):
    weak_reinit = weakref.WeakMethod(self._at_fork_reinit)
    os.register_at_fork(
        after_in_child=lambda: (m := weak_reinit()) and m()
    )
```

`os.register_at_fork`는 fork 발생 시 자식 프로세스에서 실행할 콜백을 등록하는 표준 API다. 여기서 바로 눈에 띄는 디테일이 `weakref.WeakMethod`다. 이유는: **`os.register_at_fork`는 등록 해제(unregister) API가 없다.** bound method(`self._at_fork_reinit`)를 그냥 넘기면 프로세스가 살아있는 한 그 콜백이 `self`에 대한 강한 참조를 영원히 붙잡고 있게 되어, 인스턴스가 더 이상 아무 데서도 참조되지 않아도 GC가 안 된다(메모리 누수). 약한 참조 + 월러스 연산자(`:=`)로 "콜백이 불릴 때 딱 한 번 약한 참조를 resolve"해서, 그 사이 GC가 인스턴스를 수거해버렸다면 조용히 no-op 되도록 만들었다.

### Step 2 - 예외 처리 추가

자식 프로세스에서 스레드를 새로 만들다가 실패하면(`OSError: can't start new thread` 같은 리소스 부족 상황) 그 예외가 `after_in_child` 콜백을 타고 올라가 **`os.fork()` 자체가 실패한 것처럼 워커 시작을 통째로 죽여버린다.** `try/except`로 감싸서 "텔레메트리 기능이 이번 워커에서만 비활성화되는" 수준으로 실패를 격리했다.

### Step 3 - 클래스 레벨 `RLock` 재생성 (데드락 발견)

```python
LangfuseResourceManager._lock = threading.RLock()
```

`_lock`은 싱글턴 생성(`__new__`) 시점에 잠그는 **클래스 레벨** `RLock`이다. 여기서 앞서 말한 "락의 lock 상태는 복사되지만 스레드는 복사되지 않는다"는 원리가 그대로 재현된다.

만약 **fork가 일어난 바로 그 순간**, 어떤 다른 스레드가 마침 이 락을 획득한 상태였다면? 자식 프로세스는 "잠겨 있는 락"을 메모리 그대로 물려받지만, 그 락을 풀어줘야 할 스레드는 자식에 존재하지 않는다. 결과적으로 그 락은 **자식 프로세스 안에서 영원히 풀리지 않고**, 이후 이 락을 잡으려는 모든 시도(예: 새로운 `LangfuseResourceManager` 생성)가 데드락에 빠진다. 표준적인 해법은 fork 직후 무조건 새 락 인스턴스로 교체하는 것이다.

### Step 4 - `httpx.Client` 재생성 (프로세스 안전성 문제)

이 부분은 메인테이너(wochinge)가 리뷰에서 먼저 짚어줬다 — httpx 문서는 **스레드 안전성은 약속하지만(그래서 싱글턴에서 공유해 쓰는 건 괜찮다) 프로세스 안전성은 어디에도 약속하지 않으며**, 이걸 프로세스 간에 복사해 쓰면 원인 파악이 어려운(obscure) 문제가 생길 수 있다는 것.

실제로 그렇다. `httpx.Client`는 내부에 커넥션 풀(열려 있는 TCP 소켓들의 집합, 즉 파일 디스크립터)을 들고 있다. `fork()`는 파일 디스크립터 테이블도 그대로 복사하기 때문에, **부모와 자식이 동일한 TCP 소켓을 공유**하게 된다. 한쪽 프로세스가 그 커넥션을 재사용하거나 닫는 순간, 다른 쪽은 죽은(또는 다른 요청에 재사용된) 소켓에 쓰기를 시도하게 되어 TLS 상태 불일치, connection reset 같은 알기 어려운 에러가 난다. 해법은 fork 후 자식에서 커넥션 풀을 아예 새로 만드는 것 — `httpx.Client`, `LangfuseAPI`, 내부 `LangfuseClient`(score ingestion용)를 전부 재생성했다.

### Step 5 - 사용자가 넘긴 커스텀 `httpx_client`는 건드리지 않기

Step 4에서는 사용자가 직접 넘긴 커스텀 `httpx_client`까지 포함해 **무조건 기본 클라이언트로 재생성**했다. 커스텀 transport/SSL/프록시 설정이 자식 프로세스로 이어지지 않는다는 건 알고 있었지만, 어차피 fork 후 싱글턴 내부의 클라이언트를 사용자가 직접 교체할 공개 API도 없으니 "커스텀 `httpx_client` + fork()는 아직 완전히 지원되지 않는 알려진 한계"로 문서화하는 쪽이 낫다고 판단했었다. 그런데 메인테이너가 정반대 방향을 제안했다. 요지는 이렇다(의역).

> *"httpx.Client에 대해서는 명시적으로 반대로 하고 싶다: 커스텀 httpx 클라이언트는 fork 시 재초기화하지 말고, 그냥 (fork로 넘어온) 복사본을 그대로 쓰자. 지금 구현대로라면 커스텀 클라이언트를 쓰는 사용자는 forking 환경에서 SDK를 아예 쓸 수 없다. 클라이언트를 그대로 두고 재초기화하지 않으면, 최소한 사용자가 재초기화를 직접 처리하거나 클라이언트를 다른 방식으로 프로세스 안전하게 만들 기회는 줄 수 있다."*

일리 있는 지적이라 반영했다. 최종적으로: **SDK가 자체 생성한 기본 클라이언트만 fork 후 재생성**하고, 사용자가 넘긴 커스텀 클라이언트는 fork로 복사된 상태 그대로 재사용한다. 프로세스 안전성 책임은 사용자에게 넘기고(필요하면 사용자가 자기 클라이언트에 대해 직접 `os.register_at_fork`를 걸 수 있음), docstring에 이 동작을 명시했다. **SDK가 통제할 수 없는 사용자 설정을 임의의 기본값으로 덮어쓰는 것보다, 한계를 명시하고 대응할 기회를 사용자 쪽에 남겨두는 게 더 안전한 API 설계**라는 판단이다. (근본적인 해법은 클라이언트 인스턴스 대신 `httpx_client_factory: Callable[[], httpx.Client]` 같은 팩토리를 받아서 fork 후 원래 설정 그대로 재생성하는 것이지만, 공개 API 변경이라 별도 논의로 남겨뒀다.)

### Step 6 - "무거운 작업을 지연시키면 되지 않을까" (틀린 가설)

메인테이너가 macOS에서 실제로 `gunicorn --preload`를 돌려보니 **세그폴트**가 재현됐다. 처음 가설은 "`after_in_child` 콜백 안에서 `httpx.Client` 재생성 같은 무거운 작업을 하는 게 문제 아닐까"였다. 그래서 무거운 초기화(HTTP 클라이언트 생성, 스레드 스폰)를 첫 사용 시점까지 지연시키는 lazy-init 커밋을 올렸다. 하지만 다시 테스트해보니 **세그폴트는 그대로 재현**됐다 — 무게가 문제가 아니었다. 이 lazy-init 커밋은 문제 해결에 기여하는 것 없이 복잡성만 더한다는 메인테이너 의견에 따라 이후 되돌렸다.

### Step 7 - 진짜 원인: macOS `getproxies()`가 fork 후 세그폴트를 낸다

메인테이너가 원인을 특정했다. `httpx.Client()`를 생성할 때 내부적으로 시스템 프록시 설정을 자동 감지하려고 `urllib.request.getproxies()`를 호출하는데, **macOS에서 이 함수는 Apple의 SystemConfiguration 프레임워크(CoreFoundation 계열 시스템 프레임워크)를 사용한다.**

macOS는 멀티스레드 프로세스가 `fork()`한 뒤 자식 프로세스에서 Objective-C 런타임이나 CoreFoundation API를 호출하는 것을 공식적으로 안전하지 않다고 문서화하고 있다. 메커니즘은 Step 3의 RLock 데드락과 본질적으로 동일하다 — fork 시점에 다른 스레드가 그 프레임워크 내부의 락을 들고 있었다면, 자식에서는 그 락을 풀 스레드가 없어 영원히 잠긴 채로 남는다. 다만 이번엔 Python 코드의 락이 아니라 **OS 프레임워크 내부의 락**이라 우리가 직접 고칠 수 없고, 결과도 데드락이 아니라 세그폴트로 나타났을 뿐이다. (같은 계열 문제로 Gunicorn 자체에도 이슈가 있다: [benoitc/gunicorn#2761](https://github.com/benoitc/gunicorn/issues/2761))

해법은 "안전하지 않은 API를 안전하게 만들기"가 아니라 **"애초에 그 API를 호출하지 않기"**였다.

```python
if sys.platform == "darwin":
    # urllib 프록시 탐지가 macOS SystemConfiguration API를 호출하는데,
    # 이건 fork 이후 호출하기에 안전하지 않다.
    os.environ["no_proxy"] = "*"
    os.environ["NO_PROXY"] = "*"
```

macOS에서 `urllib.request.getproxies()`는 내부적으로 `getproxies_environment() or getproxies_macosx_sysconf()` 구조로 동작한다. 프록시 관련 환경변수가 하나라도 설정되어 있으면 환경변수 쪽 결과가 채택되고, SystemConfiguration을 호출하는 `getproxies_macosx_sysconf()`까지는 아예 내려가지 않는다. 그래서 fork 직후 자식 프로세스에서만 이 환경변수를 걸어, 위험한 API 호출 자체를 원천 차단했다.

## 4. 병합 직전, 메인테이너의 마무리 두 커밋

여기까지가 내가 커밋한 7단계다. 병합 직전 메인테이너가 그 위에 두 개의 커밋을 더 얹었는데, 둘 다 짚어볼 가치가 있다.

**`ffd4a864` — 초기화 로직 통합**: 최초 생성 시의 클라이언트 초기화 코드와 fork 이후 재초기화 코드가 사실상 동일한 내용을 두 곳에 따로 들고 있었다(복붙 → 시간이 지나며 서로 다르게 진화할 위험). 

`_init_api_clients()`로 뽑아서 두 경로가 항상 같은 로직을 타도록 통합했다. 또한 Step 7의 `no_proxy` 강제 설정도 더 정교하게 다듬었다 - 사용자가 **이미 프록시 환경변수를 직접 설정해둔 경우**엔 건드리지 않도록 예외를 뒀다. <br>
그 경우엔 애초에 `getproxies()`가 SystemConfiguration까지 안 타고 환경변수에서 바로 답을 찾기 때문에 세그폴트 위험이 없고, 오히려 무조건 `no_proxy="*"`를 걸어버리면 사용자의 실제 프록시 설정을 프로세스 전역에서 꺼버리는 부작용이 생기기 때문이다.

**`e9a47c47` - 리뷰가 잡아낸 마지막 구멍**: 이게 이 PR에서 가장 미묘한 지점이고, 사람이 아니라 메인테이너가 병합 전에 돌린 자동 코드 리뷰(claude bot)가 잡아냈다. <br>
`LangfuseResourceManager`는 fork 후 `self.api`/`self.async_api`를 잘 재생성한다. 그런데 정작 사용자가 실제로 쓰는 공개 `Langfuse` 클라이언트(`client.py`)는 `__init__` 시점에 이렇게 **딱 한 번 스냅샷을 복사**해서 저장하고 있었다.

```python
self.api = self._resources.api
self.async_api = self._resources.async_api
```

fork 재초기화가 `self._resources`(리소스 매니저) 내부에서 아무리 잘 일어나도, 바깥의 `Langfuse.api`는 갱신되지 않는 **옛날 참조**(죽은 소켓을 든 `httpx.Client`를 물고 있는 API 클라이언트)를 계속 가리키고 있었다. <br>
사용자 코드는 거의 항상 `langfuse_client.api.prompts.get(...)`처럼 바깥 클라이언트를 통해 호출하지, 내부 `_resources`를 직접 건드리지 않는다 — 즉 지금까지의 모든 fork-safety 수정이 **실사용 경로에서는 무력화**되고 있었던 셈이다. 해법은 `api`/`async_api`를 프로퍼티로 바꿔서 접근할 때마다 `self._resources.api`를 통해 살아있는 참조를 가져오도록 한 것.

```python
@property
def api(self) -> LangfuseAPI:
    return self._resources.api
```

## 5. 왜 이 해법들이 맞는가 - 관통하는 하나의 원리

이 PR을 관통하는 원리는 하나다.

> **`fork()`는 "메모리 상태"는 복사하지만 "그 상태를 관리하던 실행 주체(스레드)"는 복사하지 않는다.**

이 어긋남에서 나오는 문제는 항상 두 갈래로 갈린다.

1. **아무도 처리를 안 하는 문제** — consumer 스레드 자체가 없어짐. 해법은 단순히 다시 띄우는 것 (Step 1, 2).
2. **상태는 있는데 그걸 관리하던 스레드가 없어서 영원히 회수되지 않는 문제** — 잠긴 채로 남는 `RLock`, 부모·자식이 공유하게 된 소켓 FD, macOS SystemConfiguration 내부 락(Step 3, 4, 7). 이런 건 재시작으로는 안 고쳐지고, **무조건 새로 만들거나(재생성) 애초에 그 상태 자체를 만들지 않게(getproxies 회피)** 해야 한다.

그리고 커스텀 `httpx_client` 처리(Step 5)는 이 원리와는 결이 다른, **API 설계 판단**이다 — SDK가 통제할 수 없는 사용자 설정을 임의로 덮어쓰는 것보다 한계를 인정하고 책임을 이양하는 편이 낫다는 것.

마지막 프로퍼티 위임 수정(`e9a47c47`)이 주는 교훈이 개인적으로 제일 크다: **fork-safety는 리소스를 "소유"하는 곳뿐 아니라, 그 리소스를 캐싱/스냅샷해서 들고 있는 모든 곳까지 추적해야 완전해진다.** 리소스 매니저 레벨에서 아무리 완벽하게 고쳐도, 그걸 가리키는 스냅샷 참조가 어딘가에 하나라도 남아 있으면 그 경로로는 버그가 그대로 재현된다.

---

**PR**: [langfuse/langfuse-python#1658 — fix(resource_manager): reinitialize consumer threads after os.fork()](https://github.com/langfuse/langfuse-python/pull/1658)