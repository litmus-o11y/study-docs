## chaoslib/litmus/pod-delete/lib/pod-delete.go 또는 experiments/generic/pod-delete/experiment/pod-delete.go
Kubernetes 환경에서 Pod 삭제 실험을 수행 / Kubernetes 환경에서 Pod 삭제 카오스 실험을 수행
1. 실험 관련
```go
span.SetAttributes(
    attribute.String("experiment.name", experimentsDetails.ExperimentName),
    attribute.Int64("experiment.chaos_duration", int64(experimentsDetails.ChaosDuration)),
    attribute.Float64("experiment.pods_affected_percentage", experimentsDetails.PodsAffectedPerc),
    attribute.String("experiment.sequence", experimentsDetails.Sequence),
)
```
2. 타겟 Pod 관련: 
```go
    // deriving the parent name of the target resources
		for _, pod := range targetPodList.Items {
      span.SetAttributes(
        attribute.String("target.pod.name", pod.Name),
        attribute.String("target.pod.namespace", pod.Namespace),
      )
    ...
		}
```
3. 타겟 애플리케이션 관련
```go
span.SetAttributes(
    attribute.String("target.app_ns", chaosDetails.AppDetail.Namespace),
    attribute.String("target.app_kind", chaosDetails.AppDetail.Kind),
    attribute.String("target.app_label", chaosDetails.AppDetail.Label),
)
```
4. 부모 리소스 관련: 
```go
for _, parent := range chaosDetails.ParentsResources {
    span.SetAttributes(
        attribute.String("parent.resource.name", parent.Name),
        attribute.String("parent.resource.kind", parent.Kind),
        attribute.String("parent.resource.namespace", parent.Namespace),
    )
    ...
}
```
5. 실험 진행 상태 관련 Event와 Attribute: 
```go
span.AddEvent("chaos_injection_started")
// Pod 삭제 후
span.AddEvent("pod_deleted", trace.WithAttributes(
    attribute.String("pod.name", pod.Name),
    attribute.String("pod.namespace", pod.Namespace),
))
// 실험 완료 후
span.AddEvent("chaos_injection_completed")
```
6. 실험 결과 관련
```go
span.SetAttributes(
    attribute.String("experiment.verdict", string(resultDetails.Verdict)),
)
```
8. 에러: line 
```go
if err != nil {
    span.RecordError(err)
    span.SetStatus(codes.Error, "Failed to delete pod")
}
```

## pkg/telemetry/otel.go
```go
res, err := resource.New(ctx,
    resource.WithAttributes(
        // 서비스 정보 관련
        semconv.ServiceNameKey.String(serviceName),
        semconv.ServiceVersionKey.String("1.0.0"), // 할 수 있다면 하드코딩으로 버전 추가
        // 컨테이너 및 쿠버네티스 관련
        semconv.ContainerIDKey.String(os.Getenv("CONTAINER_ID")),
        semconv.K8SNamespaceNameKey.String(os.Getenv("K8S_NAMESPACE")),
        semconv.K8SPodNameKey.String(os.Getenv("K8S_POD_NAME")),
        // 실험 관련
        attribute.String("experiment.type", "pod-delete"),
        attribute.String("chaos.platform", "litmus"),
    ),
)
```

- 참고: 동적으로 결정되는 값은 아래와 같이 런타임에 설정한다.
```go
func newTracerProvider(ctx context.Context, isExperiment bool, endpoint string, otlpAttrs map[string]string) (*trace.TracerProvider, error) {
    // ...

    attrs := []attribute.KeyValue{
        semconv.ServiceNameKey.String(serviceName),
        semconv.ServiceVersionKey.String("1.0.0"),
    }

    for key, value := range otlpAttrs {
        attrs = append(attrs, attribute.String(key, value))
    }

    res, err := resource.New(ctx,
        resource.WithAttributes(attrs...),
    )

    // ...
}
```

## bin/experiment/experiment.go
여러 카오스 실험을 실행하는 메인 함수
1. 실험 정보 관련
```go
span.SetAttributes(
    attribute.String("experiment.name", *experimentName),
    attribute.String("chaos.platform", "litmus"),
)
```
2. 클라이언트 정보 관련
```go
span.SetAttributes(
    attribute.String("kubernetes.config", os.Getenv("KUBECONFIG")),
)
```
3. 실행 환경 정보 관련
```go
hostname, _ := os.Hostname()
span.SetAttributes(
    attribute.String("host.name", hostname),
    attribute.String("os.type", runtime.GOOS),
    attribute.String("os.version", runtime.GOARCH),
)
```
4. 실험 선택 정보 관련
```go
span.SetAttributes(
    attribute.String("experiment.category", getCategoryForExperiment(*experimentName)),
)

func getCategoryForExperiment(name string) string {
    switch {
    case strings.HasPrefix(name, "pod-"):
        return "pod"
    case strings.HasPrefix(name, "node-"):
        return "node"
    case strings.HasPrefix(name, "aws-"):
        return "aws"
    // ... 기타 카테고리
    default:
        return "other"
    }
}
```
5. 텔레메트리 설정 정보 관련
```go
span.SetAttributes(
    attribute.Bool("telemetry.enabled", otelExporterEndpoint != ""),
    attribute.String("telemetry.exporter", otelExporterEndpoint),
)
```
