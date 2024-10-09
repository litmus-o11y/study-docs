### 모니터링의 새로운 미래 관측 가능성 
metric은 일정 시간동안 측정된 데이터를 집계하고 수치화하며, 전체적인 시스템의 상태를 보고하는 데 유용하다. 일반적으로 히스토그램 또는 게이지 등 차트를 사용해 시각적으로 표현한다. 가능하면 태그를 사용하여 메트릭에 상황 정보를 포함해야 한다. 
- ex) 큐의 대기 메세지 개수, 사용중인 CPU/Memory 크기, 서비스에서 초당 처리하는 개수 등
- 구글의 골든 시그널:
    - 지연(응답시간): 잠재적인 장애를 예측할 수 있는 신호로, 서비스 요청 ~ 완료까지 거리는 시간을 측정한다.
    - 에러: 단순히 에러의 리스트를 출력하는 것 뿐만 아니라, 각 에러는 traceId와 연관하여 trace를 보고 보다 상세한 근본 원인을 분석할 수 있도록 한다. 또한 이상 탐지, 인프라, 애플리케이션에서 발생하는 다양한 유형의 에러를 구분하여 출력하도록 해야한다.
    - 트래픽: 발생하는 요청의 양을 뜻한다. 시스템의 유형, 초당 요청의 수, 네트워크 입출력 등에 따라 다양한 트래픽이 있을 수 있다.
    - 포화: 리소스가 현재 얼마나 채워졌는지/얼마나 가득 채울 수 있는지를 나타낸다. 이를 통해 향후 트래픽이 증가했을 경우 이용 가능한 자원 총 사용률을 예측할 수 있다. CPU/Memory/Network와 같이 제약이 큰 리소스에 주로 적용한다.
- 집계 유형
  - counter: 모니터링하는 이벤트의 누적 개수/크기를 표현한다. 일반적으로 rate 함수와 함께 이벤트를 추적하는 데 사용한다. 주의할 점은 증감할 수 있는 메트릭을 나타내는 데 카운터를 사용해서는 안된다. ex) 초당 요청 개수, 초당 처리 개수
  - gauge: 카운터와 달리 증감하는 임의의 값을 나타내는 메트릭으로, 현재 상태를 표현한다. ex) DB에 연결된 커넥션 개수, 현재 동작하는 스레드 개수, 사용률
  - summary: 측정한 이벤트의 합계와 카운드를 모두 제공한다. 단, 집계가 필요하거나 범위/분포에 대해 측정하는 경우는 히스토그램을 권장한다. ex) response size, 대기시간
  - histogram: 시간 경과에 따른 추세와 데이터가 단일 범주에서 어떻게 분포되어있는지 나타낸다. 분위수를 사용사여 히스토그램을 주로 분석한다.
- 메트릭 명명규칙: 메트릭 이름을 지을 때는 정의된 표준을 지켜야 한다. 서비스 전체에 명명규칙을 유지하는 것이 중요하다. 메트릭의 이름을 짓는 방법 중 하나는 수집하고자 하는 메트릭의 서비스 이름, 메서드 이름, 메트릭의 유형을 사용하는 것이다.

### OpenTelemetry Metrics
1. MeterProvider 생성: 애플리케이션이나 패키지 이름으로 meter를 생성하여 메트릭을 생성하고 기록하도록 한다.
```go
import "go.opentelemetry.io/otel"

var Meter = otel.Meter("app_or_package_name")
```
2. Counter: "test.my_counter"라는 이름의 카운터를 생성하고, 단위`1`와 설명`Just a test counter`을 설정한다. Add 메서드를 사용해 카운터 값을 증가시키며, 속성(attribute)을 추가할 수 있다.
```go
counter, _ := Meter.Int64Counter(
    "test.my_counter",
    metric.WithUnit("1"),
    metric.WithDescription("Just a test counter"),
)

counter.Add(ctx, 1, metric.WithAttributes(attribute.String("foo", "bar")))
counter.Add(ctx, 10, metric.WithAttributes(attribute.String("hello", "world")))
```
3. UpDownCounter: 증가하거나 감소할 수 있는 값을 측정한다.
```go
counter, _ := Meter.Int64UpDownCounter(
    "some.prefix.up_down_counter",
    metric.WithUnit("1"),
    metric.WithDescription("TODO"),
)

// 무작위로 카운터를 1 증가시키거나 1 감소
if rand.Float64() >= 0.5 {
    counter.Add(ctx, +1)
} else {
    counter.Add(ctx, -1)
}
```
4. Histogram: 기록된 값들의 분포를 생성한다.
```go
durRecorder, _ := meter.Int64Histogram(
    "some.prefix.histogram",
    metric.WithUnit("microseconds"),
    metric.WithDescription("TODO"),
)

dur := time.Duration(rand.NormFloat64()*5000000) * time.Microsecond // 무작위 지속 시간을 생성 후
durRecorder.Record(ctx, dur.Microseconds()) // 히스토그램에 기록
// -> 총 기록 수, 평균/최소/최대 값, 히트맵/백분위수 등을 얻을 수 있다.
```
5. CounterObserver: 감소하지 않는 누적값을 측정한다.
```go
counter, _ := meter.Int64ObservableCounter(
    "some.prefix.counter_observer",
    metric.WithUnit("1"),
    metric.WithDescription("TODO"),
)

var number int64
if _, err := meter.RegisterCallback( // 콜백 함수를 등록하여 주기적으로 카운터 값을 관찰
    func(ctx context.Context, o metric.Observer) error { 
        number++
        o.ObserveInt64(counter, number)
        return nil
    },
); err != nil {
    panic(err)
}
```
6. UpDownCounterObserver
```go
counter, err := meter.Int64ObservableUpDownCounter(
    "some.prefix.up_down_counter_async",
    metric.WithUnit("1"),
    metric.WithDescription("TODO"),
)

if _, err := meter.RegisterCallback(
    func(ctx context.Context, o metric.Observer) error {
        num := runtime.NumGoroutine() // 현재 실행 중인 고루틴의 수를 주기적으로 관찰
        o.ObserveInt64(counter, int64(num))
        return nil
    },
    counter,
); err != nil {
    panic(err)
}
```
7. GaugeObserver:
```go
gauge, _ := meter.Float64ObservableGauge(
    "some.prefix.gauge_observer",
    metric.WithUnit("1"),
    metric.WithDescription("TODO"),
)

if _, err := meter.RegisterCallback(
    func(ctx context.Context, o metric.Observer) error {
        o.ObserveFloat64(gauge, rand.Float64()) // 무작위 float64 값을 게이지에 기록하고, 주기적으로 관찰
        return nil
    },
    gauge,
); err != nil {
    panic(err)
}
```

참고
- https://uptrace.dev/opentelemetry/metrics.html
- https://uptrace.dev/opentelemetry/go-metrics.html
