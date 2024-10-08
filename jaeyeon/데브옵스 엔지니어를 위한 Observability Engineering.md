# 정형화된 이벤트: 관찰 가능성의 기본 구성 요소

- 관찰 가능한 시스템:  새로운 것이는 그렇지 않은 것이든 관계없이 시스템의 상태를 얼마나 잘 설명하고 이해할 수 있는지에 대한 척도
    - 상위 수준에서 집계된 성능 데이터부터 개별 요청에서 사용된 원천 데이터까지 시스템을 반복적으로 탐색할 수 있어야 한다.
- 원격 측정 정보
    - 임의로 데이터를 여러 디멘션을 바탕으로 나누고 쪼갤 필요가 있다.
    - 가장 낮은 논리적인 수준의 개별 정보에 이르기까지 최대한 자세하게 수집되어야 한다. (수집 당시 문맥까지)
    - 서비스별 혹은 개별 요청별 수준으로 수집
- 정형화된 이벤트
    - 어느 특정 요청이 서비스와 상호 작용하는 동안 일어난 모든 것의 기록
    

## 메트릭을 기본 구성 요소로 사용하기 어려운 이유

- 메트릭: 시스템 상태를 나타내기 위해 수집한 스칼라값 + 그룹화와 검색을 도와줄 태그
- 미리 집계된 측정치라는 태생적인 한계 존재
- 메트릭과 다르게 이벤트는 접근 특정 시점에 일어난 일에 대한 스냅숏
    - 세부한 정보, 다양한 관점을 보여줄 수 있다.

## 기존 로그(비정형)를 기본 구성 요소로 사용하기 어려운 이유

- 비정형화된 로그
    - 쉽게 검색할 수 있는 방식으로 조직화되지 않고, 자신만의 포멧으로 저장
    - 거대한 텍스트 덩어리, 사람이 읽기는 좋음, 기계가 처리하긴 어렵다
- 정형화된 로그
    - 정형화된 이벤트처럼 보일 수 있도록 디자인되면 관찰 가능성의 기본 구성 요소로 유용하게 쓰일 수 있다.
    - 정형화를 잘 하자!!

## 디버깅 시 유용한 이벤트 속성

- 높은 카디널리티, 높은 디멘셔널리티
- 예상 가능한 필드뿐만 아니라 임의의 필드 값도 저장할 수 있도록 이벤트의 범위가 넓어야 한다.

# 이벤트를 추적으로 연결하기

## 분산 추척

- 추적: 프로그램 실행 중 발생할 수 있는 문제를 진단하기 위해 다양한 정보를 기록하는 기초적인 소프트웨어 디버깅 기술
- 분산 추적: 애플리케이션을 구성하는 다양한 서비스가 처리하는 단일 요청 혹은 추적의 진행 상황을 따라가는 방법
- 옵저빌리티 시스템에서 추적은 **상호 연결된 일련의 이벤트 집합**

## 추적을 구성하는 컴포넌트

- 병목 현상이 생기는 지점을 빠르게 파악하기 위해서는 추적 정보를 워터폴 형식으로 추적 시각화를 해보는 것이 좋다
- 스팬: 각 세부 단계, 하나의 추적 단위
    - 하나의 RPC 단위 뿐만 아니라 상호 연관된 시스템의 모든 개별 이벤트에 개념 적용 가능
- 루트 스팬: 최상위 스팬
    - 루트 스팬에 포함된 스팬은 또 다른 루트 스팬이 될 수 있고, 하위 스팬을 가질 수도 있다.
    - 부모, 자식 관계
- 필수 데이터
    - 추적 ID: 루트 스팬에 의해 생성되고 각 요청 처리 단계를 통해 전파
    - 스팬 ID: 생성된 개별 스팬에 대한 고유 식펼자
    - 부모 ID: 스팬 간의 종속 관계를 정의하기 위해 필요. 루트 스팬은 값을 가지고 있지 않음
    - 타임스탬프
    - 지속 시간
- 추가적인 데이터
    - 서비스 이름 : 분석을 진행하는 동안 해당 작업이 어디에서 일어났는지 알려줄 수 있는 서비스 이름
    - 스팬 이름

## 이벤트를 추적으로 연결하기

- 추적을 일반적이지 않는 방식으로 사용하는 경우를 살펴보면, 대부분 태생적으로 분산 환경의 작업이 아님에도 불구하고 여러 가지 이유로 인해 작업을 스팬으로 나눠야 할 필요가 있는 경우들이다.
    - CPU리소스를 많이 쓰는 수행 작업
    - 해결법: 문제 코드 블록을 하나의 개별 스팬으로 감싸서 더 자세한 워터폴 시각화를 확보

# OpenTelemertry를 이용한 계측

- 옵저빌리티의 진정한 힘은 실제 비즈니스 로직의 동작을 디버깅할 수 있도록 **사용자 정의 속성을 활용한 문맥** 정보가 추가되었을 때 발휘된다.
- Telemetry : 원격 측정법

## 계측이란?

- 어플리케이션을 계측하고 중앙 로깅이나 모니터링 솔루션으로 시스템의 상태 정보를 보내는 것은 소프트웨어 산업의 일반적인 관행
- 수 십년동안 개발자는 로그, 메트릭, 근래에는 추적과 프로파일 등 원격 측정 데이터의 여러 가지 카테고리를 정의해왔음
- 다양한 정보를 수용할 수 있는 이벤트는 적은 양의 정보를 가진 모호한 블롭이 아니라 여러 정형화된 데이터 필드로 구성된 특별한 종류의 로그
    - 추적은 서로 다른 이벤트를 연결하기 위한 내장 필드와 여러 개의 광범위한 이벤트로 구성
- 애플리케이션 계측에 대한 접근 방법은 다분히 독점적

## 오픈 계측 표준

- 옵저빌리티 커뮤니티는 공급 업체 락인 문제에서 벗어나기 위해 다양한 오픈소스 프로젝트를 만들었다.
- OpenTelemetry는 OpenTracing과 OpenCencus의 차세대 주요 버전
    - 추적, 메트릭, 로그뿐만 아니라 애플리케이션의 원격 측정 데이터를 포착하고 준비된 백엔드 시스템으로 전송
    - 애플리케이션 계측을 위한 단일 오픈소스 표준
    - OTel을 이용하여 한번 계측한 애플리케이션 코드 데이터는 백엔드 시스템의 종류와 관계없이 전송할 수 있다.

## 코드를 이용한 계측

- OTel은 다양한 언어로 작성된 계측 코드와 공통 메시지 표준, 컬렉터 에이전트를 제공
- 분산 추적을 채택하는 과정에서 가장 큰 숙제는 이미 알고 있는 시스템 지식과 분산 추적을 통해 얻은 정보를 연결해줄 수 있는 유용한 데이터를 얻어야 한다는 점이다.

### OpenTelemetry 개념에 관한 요약

- OTel은 다양한 원격 측정 데이터에 대한 API를 한데 모아, 이들 사이에 문맥(context)이 공톡적으로 분산 전파될 수 있도록 관리한다.
- API: 개발자가 계측의 세부적인 구현을 신경쓰지 않더라도, 코드를 계측 할 수 있도록 해주는 OTel 라이브러리 규격
- SDK: OTel을 구현하는 컴포넌트. 상태를 추적하고 전송할 데이터를 정리
- Tracer(추적기): 프로세스 내의 활성(active) 스팬을 추적하는 SDK의 컴포넌트. 현재 스팬에 속성, 이벤트를 추가하고 추적 완료 시 스팬을 종료
- Meter(미터): 프로세스에 대해 보고 가능한 메트릭을 추적하는 SDK의 컴포넌트. 현재 메트릭에 값을 추가하거나 주기적으로 값을 추출
- Context Propagation(문맥 전파): 현재 들어오는 요청 헤더에 포함된 특정 문맥(W3C TraceContext, B3M)을 역직렬화하는 SDK의 필수적인 기능. 프로세스 내에서 현재 요청의 문맥이 무엇인지 추적하고, 추적 정보를 새로운 서비스로 전달하기 위해 직렬화
- Exporter: 메모리에 적재된 OTel 객체를 지정된 저장소로 전달 가능하도록 적절한 형식으로 변환하는 SDK 플러그인.
    - 저장소 예시
        - 표준 출력
        - 예거, 허니컴, 라이트스탭
    - OpenTelemetry 컬렉터로 전송 시 사용하는 OTLP 전송 프로토콜을 이용

### 자동 계측 시작하기

- OTel의 자동 계측은 사용자가 첫 번째 계측값을 얻는 데 소요되는 시간을 최소화해준다.
- OTel은 gRPC, HTTP 및 데이터베이스와 캐시에 대한 호출의 추적 스팬을 자동으로 생성
- 자동 계측을 통해 요청의 속성과 타이밍 정보를 제공받기 위해서는 사용 중인 프레임워크가 각 요청 송신 전후에 OTel을 호출해야 한다.
    - 공통 프레임워크는 OTel이 호출할 수 있는 래퍼, 인터셉터,  혹은 미들웨어를 제공해야 한다.
- Go에서는 자동 계측을 위한 설정도 명시적이여야 한다.
    - 자바나 닷넷같은 경우는 런타임에 별도로 동작하는 OpenTelemetry 에이전트를 연결해 자동 계측을 수행할 수 있다.
    - 라우터에서 기존의 핸들러 함수를 전달하기 전에 otelhttp 패키지를 이용해 새로운 핸들러 함수를 생성한 다음 전달

### 사용자 정의 계측 추가하기

- 자동 계측은 특정 비즈니스 로직에 사용자 정의 계측을 적용할 수 있는 토대가 된다.
- 사용자 정의 계측은 관찰 가능성 주도 개발을 할 수 있는 여건을 만들어 준다.
- 코드에 사용자 정의 계측을 추가하면 특정 코드 실행 경로에 대해 비즈니스 로직을 포함한 전체 문맥을 제공받을 수 있다.
