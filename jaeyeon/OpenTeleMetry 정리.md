# Components

- A [specification](https://opentelemetry.io/docs/specs/otel) for all components
    - 오픈텔리메트리에서 정의한 스팩
    - 오픈텔리메트리 프로젝트의 overview와 중요한 용어들이 정리되어있다.
- A standard [protocol](https://opentelemetry.io/docs/specs/otlp/) that defines the shape of telemetry data
    - OTLP 프로토콜: telemetry 데이터를 여러 노드끼리 주고 받을 수 있게 하는 프로토콜(규약)
        - grpc, http 모두 지원, 페이로드는 Protocol Buffers를 사용
- [Semantic conventions](https://opentelemetry.io/docs/specs/semconv/) that define a standard naming scheme for common telemetry data types
    - telemetry 데이터들의 네이밍, 시멘틱 컨벤션 모음
- APIs that define how to generate telemetry data
- [Language SDKs](https://opentelemetry.io/docs/languages) that implement the specification, APIs, and export of telemetry data
    - 오픈텔리메트리 스팩을 구현한 SDK 이를 활용하여 쉽게 telemetry를 만들고 보낼 수 있다.
- A [library ecosystem](https://opentelemetry.io/ecosystem/registry) that implements instrumentation for common libraries and frameworks
    - 오픈텔리메트리를 활용한 여러 라이브러리
- Automatic instrumentation components that generate telemetry data without requiring code changes
    - Automatic instrumentation components를 활용하면 코드 수정 없이 telemetry 데이터를 생성할 수 있다. (JAVA Agent)
- The [OpenTelemetry Collector](https://opentelemetry.io/docs/collector), a proxy that receives, processes, and exports telemetry data
    - telemetry를 받고, 프로세싱해서, backend 시스템에 export하는 Collector 모듈
- Various other tools, such as the [OpenTelemetry Operator for Kubernetes](https://opentelemetry.io/docs/kubernetes/operator/), [OpenTelemetry Helm Charts](https://opentelemetry.io/docs/kubernetes/helm/), and [community assets for FaaS](https://opentelemetry.io/docs/faas/)

# **Understanding distributed tracing**

### Log

- timestamped 메시지
- 어떠한 요청이나 트랜잭션 단위가 아니고 개발자가 시스템을 이해할 수 있게 보이는 용도
- 정보가 부족함
- 따라서 span 에 속하거나 trace에 연관되면 좋다

### Span

- 어떠한 작업의 단위
    - 그 작업의 단위동안 어떠한 일이 일어났는지 상세하게 트래킹함
- 이름, 시간 데이터, 정형화된 로그 메시지, 기타 메타데이터(Attribute)를 가지게 된다.

### Trace

- 요청 Path를 기록
- 여러 서비스에 전파 (마이크로서비스)
- 하나 또는 그 이상의 span으로 이루어져 있다.
    - 첫번째 span은 root span
    - span들끼리 부모 자식 관계를 가진다
- Trace는 분산 시스템을 이해하고 디버깅하는데 필요

## Context Propagation

- Context propagation을 통해 Signal들은 서로 엮일 수 있다.
- 프로세스 및 네트워크 경계를 넘어  분산된 서비스들 간에 시스템에 대한 인과 관계 정보를 추적할 수 있게 해준다.
- context: 보내고 받는 서비스에 signal을 연관시키려 하는 객체
- propagation: 컨텍스트를 프로세스, 서비스 간 움직여주는 메카니즘.
    - Context Object를 직렬화, 역직렬화 한다.
    - 기본 propagator는 [W3C TraceContext](https://www.w3.org/TR/trace-context/) 표준을 구현

# Signal

- **Signal (시그널):** 시그널은 특정한 **유형**의 텔레메트리 데이터를 나타냅니다. 주로 메트릭, 로그, 트레이스, 배기지와 같은 텔레메트리 데이터의 범주를 의미
- **Telemetry (텔레메트리):** 시스템의 상태, 성능, 동작 등을 모니터링하기 위해 수집된 데이터 텔레메트리 자체는 전체적인 개념

## Trace

- trace는 어플리케이션에서 요청이 거치는 전체 경로를 이해하는데 필요
- Trace  Provider: tracer를 생성하는 팩토리. 대부분의 앱에서 한번만 초기화. SDK에서 제공
- Tracer: 서비스 내에서 span을 생성해준다.
- Trace Exporter: trace를 consumer(otel-collecter 등등)에게 전송해준다.

## Spans

- 어떠한 작업의 단위, Trace들을 구성하는 요소

### 구성 요소

- Name
- Parent span ID (empty for root spans)
    - 부모 자식 구조를 가진다.
        - 해당 Span이 부모 Span id 여부를 가지고 판단
- Start and End Timestamps
- [Span Context](https://opentelemetry.io/docs/concepts/signals/traces/#span-context)
    - 각 스팬에 있는 불변 객체로, 다음을 포함
        - Trace ID
        - Span ID
        - Trace Flag
            - 트레이스에 대한 정보를 담은 이진 플래그
        - Trace State
            - 벤더별 트레이스 정보를 전달할 수 있는 키-값 쌍의 리스트
- [Attributes](https://opentelemetry.io/docs/concepts/signals/traces/#attributes)
    - 메타데이터를 포함하는 키-값 쌍
    - 각 Span에 대한 추가적인 정보를 포함
- [Span Events](https://opentelemetry.io/docs/concepts/signals/traces/#span-events)
    - Span에 구조화된 로그 메시지 (또는 주석). 일반적으로 스팬의 기간 동안 중요한 당일 시점을 나타낸다.
    - Span의 Attribute로 둘지 Event로 둘지는 해당 요소가 TimeStamp를 가지는지에 따라 달라진다.
- [Span Links](https://opentelemetry.io/docs/concepts/signals/traces/#span-links)
    - 서로 다른 트레이스의 스팬을 인과적으로 연결하여 관련성을 나타내는 기능
- [Span Status](https://opentelemetry.io/docs/concepts/signals/traces/#span-status)
    - 각 Span의 상태
        - Unset: 기본값으로, 스팬이 추적하는 작업이 오류 없이 성공적으로 완료되었음을 의미합니다.
        - Error**:** 스팬이 추적하는 작업에서 오류가 발생했음을 의미합니다. (HTTP 500 에러)
        - Ok: 개발자가 명시적으로 오류 없는 상태로 표시함. 일반적으로 성공은 Unset이나, OK는 스팬에 대해 “성공적” 외에 다른 해석을 하지 않도록 하고자 하는 경우에 사용
- Span Kind: Span에 대한 종류
    - Client: HTTP 요청 또는 데이터베이스 호출과 같은 동기식 원격 호출을
    - Server: HTTP 요청 수신 또는 원격 프로시저 호출과 같은 동기식 수신 원격 호출을
    - Internal: 프로세스 경계를 넘지 않는 작업을 나타내며, 함수 호출이나 Express 미들웨어 같은 내부 작업에 사용
    - Producer: 나중에 비동기적으로 처리될 작업의 생성을 나타내며, 원격 작업일 수도 있고 로컬 작업일 수도 있다.
    - Consumer: 프로듀서가 생성한 작업을 처리하는 것을 나타내며, 이는 프로듀서 스팬이 끝난 후에 시작될 수 있다.

## Metrics

- 런타임에 수집된 측정값
- 측정이 이루어진 순간을 metric event라고 한다.

### 구성 요소

- 이름
- 종류
    - Counter: 시간이 지남에 따라 증가하는 값
    - Asynchronous Counter: Counter와 동일하지만 Export할 때 한 번만 수집된는 값 (즉 주기적으로 수집하는 값이 아님)
    - UpDownCounter: 증가하거나 감소하는 값
    - Asynchronous UpDounCounter
    - Gauge: 읽을 때 현재 값 (일반적인 값)
    - Histogram: Client단의 집계 값 (예시: 1초 미만으로 처리된 요청 수)

### View

- 메트릭을 커스터마이징할 수 있는 기능
