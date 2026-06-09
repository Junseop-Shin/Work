# Go 동시성 — goroutine·channel·select·sync·context

> 2026-06-08 | Go, 동시성, goroutine, channel, context, CSP

## 한 줄 요약

Go 동시성은 **경량 실행 단위(goroutine)를 런타임이 소수의 OS 스레드에 다중화(M:N)** 하고, 상태를 락으로 지키는 대신 **채널로 값(소유권)을 건네 통신**하는 CSP 모델을 1급 문법으로 제공하는 것이다.

## 핵심 개념

### 왜 goroutine인가 — "통신으로 메모리를 공유하라"

전통적 동시성은 *공유 메모리 + 락(Mutex)*이다. 변수를 같이 쓰되 자물쇠로 보호한다. 문제는 (1) 언제 값이 준비됐는지 알기 어렵고 (2) 락 누락/순서 꼬임이 잦다는 것.

Go의 격언:
> "메모리를 공유해서 통신하지 말고, **통신해서 메모리를 공유하라.**"

즉 값을 채널로 **건네주는 행위 자체가 "이제 네 차례" 라는 동기화 신호**가 된다. 락 없이 안전해진다. (단, 채널이 유일한 방법은 아니다. 상태 보호엔 여전히 Mutex가 적합 — 아래 참고.)

### Goroutine — 스레드처럼 보이지만 스레드가 아니다

```go
go doWork()   // 이게 전부. 경량 실행 단위 생성.
```

| | OS 스레드 | Goroutine |
|---|---|---|
| 초기 스택 | ~1MB 고정 | **~2KB, 동적 증감** |
| 생성 비용 | 커널 호출, 무거움 | 유저 공간, 매우 쌈 |
| 스케줄링 | OS 커널 | **Go 런타임** |
| 현실적 개수 | 수천 | **수십만~수백만** |

#### G-M-P 스케줄러 (면접 포인트)

- **G** = goroutine, **M** = machine(OS 스레드), **P** = processor(논리 프로세서, `GOMAXPROCS`개)
- **M은 P를 하나 점유해야 G를 실행**할 수 있다. P마다 로컬 런큐(대기 goroutine 목록)를 가진다.
- **M:N 다중화**: 수많은 G를 적은 수의 M에 얹는다.
- **work-stealing**: 한 P의 큐가 비면 다른 P 큐에서 절반을 훔쳐와 부하를 분산한다.
- **핸드오프(handoff)**: G가 시스템콜로 블로킹되면 M에서 P를 떼어내 다른 M에 넘긴다 → 나머지 G는 계속 실행. 그래서 블로킹 I/O가 있어도 전체가 멈추지 않는다.

⚠️ 함정:
- `go`로 띄운 뒤 `main`이 끝나면 goroutine도 그냥 죽는다 → `WaitGroup`/채널로 대기해야 함.
- 루프 변수 캡처: Go 1.22 이전엔 반복마다 같은 변수를 캡처해 버그. 1.22+에서 반복마다 새 변수로 수정됨.

### Channel — 고루틴 간 타입 안전 파이프

```go
ch := make(chan int)      // 언버퍼드(동기)
ch := make(chan int, 5)   // 버퍼드(5칸)
ch <- 10                  // 송신
v := <-ch                 // 수신
v, ok := <-ch             // ok=false면 닫힌 채널
close(ch)
```

#### 언버퍼드 = "손에서 손으로 직거래"

칸이 없어 **두 고루틴이 만나야** 거래가 성립한다(랑데부). 먼저 온 쪽이 늦게 온 쪽을 기다린다.

```go
ch := make(chan int)
go func() { ch <- 42 }()   // 받는 사람 올 때까지 블로킹
v := <-ch                  // 악수! 동시에 전달
```

송수신이 **반드시 동시에** 일어나므로 "값이 전달된 순간 둘 다 이 지점을 통과했다"가 보장된다 = 강한 동기화.

#### 버퍼드 = "택배함 N칸"

- **송신**: 빈칸 있으면 넣고 진행(논블로킹), 꽉 차면 빌 때까지 대기.
- **수신**: 물건 있으면 꺼내고 진행, 비었으면 들어올 때까지 대기.

```go
ch := make(chan int, 2)
ch <- 1   // [1]   통과
ch <- 2   // [1,2] 통과
ch <- 3   // 꽉 참 → 누가 꺼낼 때까지 블로킹
```

> 언버퍼드 = 칸 0개짜리 버퍼드. 항상 직거래일 뿐 원리는 같다.

#### close 규칙

- 닫힌 채널 **수신** → 남은 값 다 받고, 이후엔 `zero value + ok=false` 무한 반환.
- 닫힌 채널 **송신** → **panic**. 재close → panic.
- **관례: 송신자가 닫는다. 수신자는 닫지 않는다.** (수신자가 닫으면 다른 송신자가 panic)
- `for v := range ch` → 닫힐 때까지 받다가 자동 종료.

⚠️ **nil 채널**: 송수신 모두 영원히 블로킹. select에서 case를 동적으로 끄는 트릭으로 의도적 활용.

#### happens-before 보장

채널엔 메모리 가시성 규칙이 있다:
> 송신자가 보내기 전에 한 모든 쓰기는, 수신자가 받은 후에 전부 보인다.

```go
data = 42; ch <- struct{}{}   // 쓰기 후 신호
// ── 다른 고루틴 ──
<-ch; use(data)               // data는 반드시 42
```

그래서 채널로 값을 넘기면 그 값과 그 전 작업들이 함께 동기화된다.

### select — 채널들의 switch

```go
select {
case v := <-ch1:   use(v)
case ch2 <- x:     sent()
case <-time.After(2 * time.Second): timeout()  // 타임아웃
default:           nonblocking()               // 준비된 게 없을 때
}
```

- 준비된 채널 중 **하나** 실행. 여러 개 준비되면 **랜덤 선택**(기아 방지).
- `default` 있으면 논블로킹, 없으면 준비될 때까지 블로킹.
- 취소/타임아웃/멀티플렉싱의 기본 도구.

### sync — "메모리 공유 + 락" 진영

| 도구 | 용도 |
|------|------|
| `sync.Mutex` | 상호 배제. 한 번에 한 고루틴만 임계구역 진입 |
| `sync.RWMutex` | 읽기 다수 동시 / 쓰기 독점. 읽기 압도적인 캐시류 |
| `sync.WaitGroup` | 여러 고루틴이 다 끝날 때까지 대기 |
| `sync.Once` | 초기화 등 딱 한 번만 실행 보장 |
| `sync/atomic` | 락 없는 원자 연산(카운터/플래그) |

```go
var mu sync.Mutex
mu.Lock(); count++; mu.Unlock()    // 보통 defer mu.Unlock()

var wg sync.WaitGroup
for i := 0; i < 3; i++ {
    wg.Add(1)                       // 띄우기 전에 Add
    go func() { defer wg.Done(); work() }()
}
wg.Wait()

atomic.AddInt64(&count, 1)          // 단순 카운터의 정답
```

#### Mutex vs Channel — 언제 뭘?

| 상황 | 선택 |
|------|------|
| 공유 **상태** 보호(카운터·캐시·맵) | **Mutex** (또는 atomic) |
| 데이터 **전달 / 작업 분배 / 신호** | **Channel** |
| 단순 숫자/플래그 | **atomic** |
| 고루틴 종료 대기 | **WaitGroup** |

격언: **"상태를 지키려면 Mutex, 데이터를 옮기려면 Channel."** 맵 하나 보호하는 데 채널은 과하다.

### context — 취소·마감·요청값을 트리 아래로 전파

**일반적인 "컨텍스트(실행 환경/DI 컨테이너)" 개념이 아니다.** Go의 `context`는 인터페이스 메서드가 딱 4개로, 용도가 **2가지로 좁혀진** 특수 목적 도구다.

```go
type Context interface {
    Done() <-chan struct{}        // ┐
    Deadline() (time.Time, bool)  // ├ 일①: 취소·마감 전파
    Err() error                   // ┘
    Value(key any) any            // ─ 일②: 요청 스코프 값
}
```

#### 일① 취소/마감 전파

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()                 // 반드시 호출(누수 방지)
go worker(ctx)
cancel()                       // → ctx.Done() 채널이 close → 모든 worker 감지

func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():     // Done()은 그냥 채널! (close되면 즉시 수신)
            return             // ctx.Err(): Canceled / DeadlineExceeded
        case job := <-jobs:
            process(job)
        }
    }
}
```

생성 함수: `Background()`(뿌리), `WithCancel`, `WithTimeout`, `WithDeadline`, `WithValue`.
**핵심 성질 = 트리**: 부모를 취소하면 자식들도 전부 취소된다(위→아래 단방향 전파). 요청 하나에 딸린 고루틴 수십 개를 루트 cancel 하나로 정리.

#### 일② 요청 스코프 값

```go
ctx := context.WithValue(r.Context(), userKey{}, currentUser)
user := ctx.Value(userKey{}).(*User)   // 한참 아래 핸들러에서 꺼냄
```

용도: 하나의 요청이 흐르는 동안 호출 스택을 따라다녀야 하는 메타데이터(traceID, 인증 유저, requestID). 구조적으로 React Context와 닮았지만(트리 아래로 값 전파) 용도가 "요청 부가 데이터"로 좁혀져 있다.

⚠️ 함정:
- **일반 파라미터를 Value에 숨기지 마라** — 진짜 필요한 값은 명시적 인자로. Value는 타입 안전성이 없다(`any`).
- **DI 컨테이너로 쓰지 마라** — DB 커넥션·로거를 ctx에 넣는 건 안티패턴.
- 키는 충돌 방지를 위해 **비공개 커스텀 타입**(`type userKey struct{}`).
- `cancel`은 무조건 `defer cancel()`.

### 실전 패턴

```go
// worker pool — 동시성 = 워커(고루틴) 수, 채널은 운반 수단일 뿐
jobs := make(chan int, 100); results := make(chan int, 100)
for w := 0; w < 3; w++ {          // 동시 처리 3
    go func() {
        for j := range jobs { results <- j * 2 }   // 각 job은 한 워커만 가져감
    }()
}
for i := 0; i < 100; i++ { jobs <- i }
close(jobs)
```

- **Fan-out / Fan-in**: 입력 채널 하나를 여러 워커가 나눠 처리(fan-out), 여러 결과를 채널 하나로 합침(fan-in, WaitGroup으로 다 끝나면 close).
- **Pipeline**: 각 단계가 입력 채널 → 출력 채널. 리눅스 파이프 `|`처럼 단계들이 동시에 흐름. 취소는 context로 일괄.

> 패턴들의 공통 재료: **goroutine(일꾼) + channel(운반) + WaitGroup/context(수명관리).**

## 핵심 질의응답

**Q. 고루틴은 변수를 어떻게 공유하나? 채널로만?**
A. 고루틴은 같은 주소공간이라 변수를 직접 공유한다. 공유 방법은 둘 — (1) 변수+Mutex(메모리 공유), (2) Channel(통신). 채널이 권장이지 유일은 아니다.

**Q. 채널은 int만 보내나?**
A. 아니다. 타입 붙은 파이프라 `chan User`, `chan *Job`, `chan error`, `chan chan int` 등 뭐든. 단 포인터/슬라이스를 보내면 알맹이는 여전히 공유되어 레이스 가능.

**Q. 채널 개수만큼 동시 처리되나?**
A. 아니다. 동시 처리량은 **goroutine 수**가 정한다. 채널(버퍼)은 "운반/적재량"일 뿐. 워커 3개면 채널 버퍼가 100이어도 동시 처리는 3.

**Q. 인덱스를 채널로 +1 해서 다음 고루틴이 쓰면 동기화되나?**
A. happens-before 보장으로 동기화 자체는 맞다. 다만 단순 카운터는 atomic/Mutex가 적합하고, 채널은 "작업 분배 큐"로 쓰는 게 제 용도다(각 값을 한 고루틴만 가져감).

**Q. context는 취소만 하나?**
A. 아니다. 두 가지 — 취소/마감 전파(Done/Deadline/Err)와 요청 스코프 값(Value). 일반적인 "실행 환경" 컨텍스트가 아니라 이 2개로 좁혀진 도구다.

## 주의사항 / 자주 하는 실수

- **닫힌 채널에 송신 → panic.** 송신자가 닫고, 수신자는 닫지 않는다.
- **`main` 종료 시 goroutine 즉사** — WaitGroup/채널로 대기.
- **`cancel()` 누락** — 타이머/고루틴 누수. `defer cancel()` 습관화.
- **채널 데드락** — 언버퍼드에 받는 쪽 없이 송신하면 영원히 블로킹.
- **Mutex로 충분한 걸 채널로** — 단순 상태 보호엔 Mutex가 더 단순/빠름.

## 참고

- [Go 기본](Go-기본.md) — 타입/구조체/인터페이스/에러
- [언어별 동시성 비교](언어별-동시성-비교.md) — Node/Python/Java/Go/C# + 리소스 단위 비교
- [Go Memory Model](https://go.dev/ref/mem)
- [Go Concurrency Patterns (Rob Pike)](https://go.dev/blog/pipelines)
