# [GopherCon UK 2021: Instrumenting GO Applications](https://www.youtube.com/watch?v=Xo7S5fQq2V8)

- Tracing:
  - "Trace a single request through the platform": 플랫폼을 통해 단일 요청을 추적하여 시스템에서 요청의 전체 경로를 이해하는 데 도움을 준다.
  - "Span code paths" - 코드 경로를 span으로 나누어 추적한다. (span: 작업의 시작 ~ 끝까지의 기간)
  - "Side effect visibility": 부작용의 가시성을 제공함으로써 예상치 못한 동작이나 영향을 파악하는 데 도움이 된다.
  - "Discover performance bottlenecks": 성능 병목 지점을 발견하는 데 사용된다.
  - span v.s. trace
    - 각 서비스 사이의 span은 각 서비스가 수행된 기간이다
    - 전체 흐름을 아우르는 trace는 전체 요청의 생명 주기를 나타낸다.
    - <img width="500" alt="image" src="https://github.com/user-attachments/assets/f15b9c93-c50f-4209-8db3-91e352a4aa9a">
  - 트레이싱 예시 코드
  - ```go
    package main

    import "time"
    
    func main() {
        span := StartSpan() // span 시작 

        /*
        // Do Some Work.
        ints := []int{1, 2, 3, 4}
        
        for _, i := range ints {
            time.Sleep(time.Second * i)
        }
        
        // Do Some More Work
        */
    
        span.End() // span 종료
    }
    ```
- Logging:
  - 디버깅을 돕기 위한 특정 서비스 정보를 제공한다.
  - "structured": parsing이 쉽도록 로그는 구조화되어야 한다 (ex. JSON)
    - github.com/rs/zerolog
    - github.com/uber-go/zap
    - 위 라이브러리는 Go에서 효율적인 구조화 로깅을 위해 주로 사용된다.
  - "context": 엔지니어가 문제를 진단할 수 있도록 충분한 컨텍스트를 제공해야 한다.
    - Trace/Span ID를 포함하여 로그 라인을 원래 요청과 연관지을 수 있어야 한다.
  - "stack traces": 오류 발생 시 스택 트레이스를 포함하는 것을 고려해야 한다.
    - github.com/pkg/errors.WithStack 를 사용하여 구현할 수 있다.
  - "propagate the logger": 가능한 경우 context.Context를 통해 로거를 전파해야 한다.
  - 로그 예시
  - ```json
    { //  HTTP 응답 코드, 요청 처리 시간, 서비스 이름, span ID, trace ID 등 다양한 정보를 포함하고 있어 문제 진단에 유용하다.
      "http": {
        "response": {
          "code": 200,
          "duration": 0.002382,
          "written": 0
        }
      },
      "level": "info",
      "message": "handled HTTP/1.1 request GET /",
      "service": "name-srv",
      "spanID": "7fc56dff77555b6a",
      "timestamp": "2021-10-19T15:07:40Z",
      "traceID": "57f6fcb2b3681cc8d1acfba431837b7c",
      "version": "2254bce5"
    }
    ```
- Metrics:
  - 애플리케이션 성능에 대한 인사이트를 제공
    - counters: 증가만 하는 값을 측정 (ex. error counts, HTTP 요청 입/출력 수, HTTP bytes in/out 등)
    - gauges: 증가하거나 감소할 수 있는 값을 측정 (ex. 현재 활성 사용자 수 등)
    - 메트릭스를 기반으로 한 알림 시스템을 구축할 수 있으며, 비즈니스 관련 지표(판매량, 사용자 등록 수 등)도 추적할 수 있다.
  - "Avoid log based metrics": 로그를 파싱하여 메트릭스를 생성하는 것은 비효율적일 수 있으므로 직접적인 메트릭스 수집을 권장한다.
  - 메트릭 시각화 예시: HTTP 200번대 응답의 지연 시간 - 대부분의 요청이 처리되는 시간과 극단적인 지연이 발생하는 시점을 쉽게 파악할 수 있다.
  - <img width="600" alt="image" src="https://github.com/user-attachments/assets/027b42ed-7599-4cfd-a939-363ea5437537">

## OpenTelemetry
애플리케이션 성능 모니터링(APM)과 관찰 가능성 분야를 표준화하고 단순화하는 것을 목표로 한다. 메트릭, 로그, 트레이스를 수집하기 위한 통일된 방식을 제공함.
1. OpenTelemetry의 Trace 시스템 구조와 사용 방법
- API: feature freeze
  - TraceProvider: API의 진입점으로, Tracer 인스턴스를 제공
  - Span을 생성하며, 새로운 Tracer 인스턴스는 항상 TraceProvider를 통해 생성되어야 한다.
  - Span: 작업을 추적하기 위한 API
- SDK: experimental
  - TraceProvider: TraceProvider는 Shutdown과 ForceFlush 메소드를 구현해야한다.
    - Shutdown: 트레이싱 시스템을 정상적으로 종료하는 메소드
    - ForceFlush: 현재 처리 중인 모든 트레이스 데이터를 즉시 내보내도록 강제하는 메소드
  - Tracer: 새로운 인스턴스는 항상 TraceProvider를 통해 생성되어야 한다.
2. OpenTelemetry의 Metrics 시스템 구조와 사용 방법
- API: feature freeze
  - MeterProvider: API의 진입점으로, Meter 인스턴스를 제공
  - Meter: Instrument 인스턴스를 반환하는 역할
  - Instrument: 실제 측정을 수행(ex. Counter)
- SDK: experimental
  - MeterProvider: MeterProvider는 Shutdown과 ForceFlush 메소드를 구현해야 한다.
  - Meter: 새로운 인스턴스는 항상 MeterProvider를 통해 생성되어야 한다.
3. OpenTelemetry의 Logging 시스템 구조와 사용 방법: experimental
- 트레이스나 메트릭과는 다른 접근 방식을 취하고 있으며, "Clean Sheet Approach"를 사용한다.
- 로그는 해결해야 할 가장 큰 레거시 과제 중 하나
  - 대부분의 프로그래밍 언어들이 내장 로거를 가지고 있거나 잘 알려진 서드파티 라이브러리를 사용하고 있음
  - 기존 로깅 라이브러리의 한계: 기존 로깅 라이브러리들은 관찰 가능성(observability)과 약하게 연결되어 있다
  -> 기존 로깅 시스템과의 통합 및 개선이 필요하다.
4. Go SDKs
- github.com/open-telemetry/opentelemetry-go: 트레이싱, 메트릭스 등을 위한 핵심 API를 제공 (프로메테우스, 예거 support 모두 포함)
- github.com/open-telemetry/opentelemetry-go-contrib: 서드파티 계측(instrumentation)과 익스포터(exporters)를 제공
  - net/http를 위한 계측, Datadog용 exporter, B3 및 AWS propagators, Shopify Sarama (Kafka 라이브러리) 지원

## [Demo](https://github.com/banked/GopherConUK2021)
1. demo1: print out terminal & json formatted trace
2. demo2 -> exporter: jaeger(trace)
  ```go
  	// Create the exporter
  	exp, err := jaeger.New(jaeger.WithAgentEndpoint())
  	if err != nil {
  		panic(err)
  	}
  ```
3. demo3-> 2 HTTP server to talk each other
  - 참고: https://github.com/openzipkin/b3-propagation
  ```go
  var (
  	tracer     trace.Tracer
  	provider   *sdktrace.TracerProvider
  	propagator = b3.New(b3.WithInjectEncoding(b3.B3MultipleHeader)) // create a new propagator; send HTTP headers
  )
  ...
  func main() {
    //
  	srv := &http.Server{
  		Addr: "localhost:3000",
  		Handler: otelhttp.NewHandler( // handler is wrapped in otelhttp handler
        // 해당 http 헤더에서 발생하는 스팬/트레이스를 자동으로 추출하고 요청 컨텍스트를 업데이트하여 스팬을 갖도록 하며, 메트릭에 대한 자동 계측을 수행
  			handler(),
  			"http.server",
  			otelhttp.WithPropagators(propagator)),
  	}
    //
  }
  ```
4. demo4 -> zerolog를 사용하여 로그에 traceId/spanId를 주입
  - 참고: https://github.com/rs/zerolog
  - zerolog hook function: 로그 이벤트에 트레이싱 정보(traceID와 spanID)를 자동으로 추가
    - 프로덕션 환경에서는 일부 트레이스만 샘플링하고, 스테이징에서는 모든 트레이스를 샘플링하는 등의 설정이 가능하다. `IsRecording()`이 이러한 샘플링 설정을 반영함
    - 트레이스를 생성하는 경우 HTTP 헤더를 통해 특정 트레이스의 샘플링을 강제할 수도 있다. (프로덕션 환경에서 특정 문제를 디버깅할 때 유용함)
  ```go
  // SpanLogHook adds a gook into a zerolog.Logger if the span is
  // recording and has a valid TraceID and SpanID.
  func SpanLogHook(span trace.Span) zerolog.Hook { // zeroHook은 로그 이벤트가 생성될 때마다 호출된다.
  	return zerolog.HookFunc(func(event *zerolog.Event, _ zerolog.Level, _ string) {
  		if span.IsRecording() { // 현재 스팬이 샘플링되어 기록 중인지 확인
  			ctx := span.SpanContext()

        // 유효한 traceID와 spanID가 있는지 확인
  			if ctx.HasTraceID() { 
  				event.Str("traceID", ctx.TraceID().String()) // 조건이 충족되면 로그 이벤트에 추가
  			}
  			if ctx.HasSpanID() {
  				event.Str("spanID", ctx.SpanID().String())
  			}
  		}
  	})
  }
  ```
5. demo5 -> metrics sdk를 사용하는 경우

