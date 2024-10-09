## Attribute
OpenTelemetry에서 메트릭이나 트레이스에 추가 정보를 제공하는 키-값 쌍으로, 이를 통해 메트릭이나 트레이스에 컨텍스트를 부여하고 더 세분화된 분석을 가능하게 한다.
- 추가 정보 제공: 메트릭이나 트레이스에 대한 부가적인 정보를 제공한다
- 분류와 필터링: 데이터를 분류하고 필터링하는 데 사용된다
- 차원 추가: 메트릭에 추가적인 차원을 부여한다
- 상세한 분석: 더 깊이 있는 분석을 가능하도록 한다

1. 메트릭(Metrics)에서의 Attribute: 메트릭 데이터에 추가적인 차원을 제공한다.
- ex) 서버 이름, 환경(개발/스테이징/프로덕션), 고객 ID 등
```go
counter.Add(ctx, 1, // HTTP 요청을 측정하는 카운터
    metric.WithAttributes(
        attribute.String("method", "GET"), // "method": HTTP 메서드
        attribute.String("path", "/users"), // "path": 요청 경로
        attribute.Int("status_code", 200) // "status_code": 응답 상태 코드
    )
)
```
- 이러한 attribute를 사용하면: 특정 경로에 대한 요청 수 확인/HTTP 메서드별 요청 분포 분석/상태 코드별 요청 수 파악 등이 가능해진다.
2. 트레이스(Traces)에서의 Attribute: 스팬(Span)에 추가 정보를 제공하며, 트레이스의 컨텍스트를 더 잘 이해할 수 있게 해준다.
- ex) HTTP 요청의 URL, 메서드, 상태 코드, 데이터베이스 쿼리 등
```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/trace"
)

func handleRequest(ctx context.Context, req *http.Request) { // HTTP 요청을 처리하는 함수에서 trace span에 attribute 추가
    tracer := otel.Tracer("example-tracer")
    ctx, span := tracer.Start(ctx, "handle-request")
    defer span.End()

    // Attribute 추가
    span.SetAttributes(
        attribute.String("http.method", req.Method),
        attribute.String("http.url", req.URL.String()),
        attribute.String("http.user_agent", req.UserAgent()),
    )

    // 요청 처리 로직...
}
```
- 각 요청의 처리 과정을 추적할 때 HTTP 메서드, URL, 사용자 에이전트 등의 추가 정보를 함께 볼 수 있다.
```go
import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/trace"
)

func performDatabaseQuery(ctx context.Context, query string) {
    tracer := otel.Tracer("database-operations")
    ctx, span := tracer.Start(ctx, "execute-query")
    defer span.End()

    // Span에 Attribute 추가
    span.SetAttributes(
        attribute.String("db.system", "postgresql"),
        attribute.String("db.name", "users"),
        attribute.String("db.statement", query),
    )

    // 데이터베이스 쿼리 실행 로직...
}
```
- 데이터베이스 쿼리 실행을 추적하는 Span에 데이터베이스 시스템, 이름, 실행된 쿼리 등의 Attribute를 추가할 수도 있음
3. 로그(Logs)에서의 Attribute: 로그 이벤트에 구조화된 데이터를 추가하고, 로그 검색과 분석을 용이하게 한다.
- ex) 사용자 ID, 세션 ID, 에러 코드 등
```go
import (
    "go.uber.org/zap"
    "go.opentelemetry.io/otel/attribute"
)

func logUserActivity(logger *zap.Logger, userID string, action string) {
    // Attribute를 zap.Field로 변환
    fields := []zap.Field{
        zap.String("user.id", userID),
        zap.String("action", action),
        zap.Int64("timestamp", time.Now().Unix()),
    }

    // Attribute와 함께 로그 기록
    logger.Info("User activity", fields...)
}

func main() {
    logger, _ := zap.NewProduction()
    defer logger.Sync()

    logUserActivity(logger, "user123", "login")
}
```
- 사용자 활동을 로깅할 때 userID, action, timestamp 등의 Attribute를 포함시키도록 한다.

## Event
일반적으로 span의 수명 동안 발생한 중요한 순간을 기록하는 데 사용한다.
```go
import (
    "go.opentelemetry.io/otel/trace"
    "go.opentelemetry.io/otel/attribute"
)

func processOrder(ctx context.Context, orderID string) {
    tracer := otel.Tracer("order-processor")
    ctx, span := tracer.Start(ctx, "process-order")
    defer span.End()

    // 주문 처리 로직...

    // Event 추가
    span.AddEvent("order-received", trace.WithAttributes(
        attribute.String("order.id", orderID),
        attribute.Int("items.count", 3),
    ))

    // 더 많은 처리 로직...

    span.AddEvent("order-shipped")
}
```

## Error
span 실행 중 발생한 오류를 기록하는 데 사용한다.
```go
import (
    "go.opentelemetry.io/otel/codes"
)

func fetchUserData(ctx context.Context, userID string) (*UserData, error) {
    tracer := otel.Tracer("user-service")
    ctx, span := tracer.Start(ctx, "fetch-user-data")
    defer span.End()

    userData, err := db.FetchUser(ctx, userID)
    if err != nil { // 데이터베이스에서 사용자 데이터를 가져오는 중 오류가 발생하면
        span.RecordError(err) // 오류 기록
        span.SetStatus(codes.Error, "Failed to fetch user data") // SetStatus를 사용하여 스팬의 상태를 "Error"로 설정
        return nil, err
    }

    return userData, nil
}
```
