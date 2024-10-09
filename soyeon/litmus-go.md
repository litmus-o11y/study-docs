```go
// pkg/telemetry/otel.go
package telemetry

import (
	"context"
	"errors"
	"github.com/go-logr/logr"
	"github.com/go-logr/stdr"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	"go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.25.0"
	codetrace "go.opentelemetry.io/otel/trace"
	"log"
	"os"
)

const (
	OTELExperimentJobServiceName       = "chaos_experiment_job"
	OTELExperimentJobHelperServiceName = "chaos_experiment_job_helper"
	OTELExporterOTLPEndpoint           = "OTEL_EXPORTER_OTLP_ENDPOINT"
)

type TracingLoggerWrapper struct {
	sink logr.LogSink
}

func (w *TracingLoggerWrapper) Init(info logr.RuntimeInfo) {
	w.sink.Init(info)
}

func (w *TracingLoggerWrapper) Enabled(level int) bool {
	return w.sink.Enabled(level)
}

func (w *TracingLoggerWrapper) Info(level int, msg string, keysAndValues ...interface{}) {
	ctx := context.Background()
	spanContext := codetrace.SpanFromContext(ctx).SpanContext()
	if spanContext.IsValid() {
		keysAndValues = append(keysAndValues,
			"trace_id", spanContext.TraceID().String(),
			"span_id", spanContext.SpanID().String())
	}
	w.sink.Info(level, msg, keysAndValues...)
}

func (w *TracingLoggerWrapper) Error(err error, msg string, keysAndValues ...interface{}) {
	ctx := context.Background()
	spanContext := codetrace.SpanFromContext(ctx).SpanContext()
	if spanContext.IsValid() {
		keysAndValues = append(keysAndValues,
			"traceId", spanContext.TraceID().String(),
			"spanId", spanContext.SpanID().String())
	}
	w.sink.Error(err, msg, keysAndValues...)
}

func (w *TracingLoggerWrapper) WithValues(keysAndValues ...interface{}) logr.LogSink {
	return &TracingLoggerWrapper{sink: w.sink.WithValues(keysAndValues...)}
}

func (w *TracingLoggerWrapper) WithName(name string) logr.LogSink {
	return &TracingLoggerWrapper{sink: w.sink.WithName(name)}
}

func InitOTelSDK(ctx context.Context, isExperiment bool, endpoint string) (shutdown func(context.Context) error, err error) {
	var shutdownFuncs []func(context.Context) error

	shutdown = func(ctx context.Context) error {
		var err error
		for _, fn := range shutdownFuncs {
			err = errors.Join(err, fn(ctx))
		}
		shutdownFuncs = nil
		return err
	}

	handleErr := func(inErr error) {
		err = errors.Join(inErr, shutdown(ctx))
	}

	tracerProvider, err := newTracerProvider(ctx, isExperiment, endpoint)
	if err != nil {
		handleErr(err)
		return
	}

	prop := newPropagator()
	otel.SetTextMapPropagator(prop)

	shutdownFuncs = append(shutdownFuncs, tracerProvider.Shutdown)
	otel.SetTracerProvider(tracerProvider)

	// TODO: need to add metrics & logging provider
	stdrLogger := stdr.NewWithOptions(log.New(os.Stdout, "", log.LstdFlags|log.Lshortfile), stdr.Options{LogCaller: stdr.All})
	wrappedLogger := logr.New(&TracingLoggerWrapper{
		sink: stdrLogger.GetSink(),
	})
	otel.SetLogger(wrappedLogger)

	return
}

func newPropagator() propagation.TextMapPropagator {
	return propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{},
		propagation.Baggage{},
	)
}

func newTracerProvider(ctx context.Context, isExperiment bool, endpoint string) (*trace.TracerProvider, error) {
	serviceName := OTELExperimentJobHelperServiceName
	if isExperiment {
		serviceName = OTELExperimentJobServiceName
	}
	res, err := resource.New(ctx,
		resource.WithAttributes(
			semconv.ServiceNameKey.String(serviceName),
		),
	)
	traceExporter, err := otlptrace.New(
		ctx,
		otlptracegrpc.NewClient(
			// TODO: add secure option
			otlptracegrpc.WithInsecure(),
			otlptracegrpc.WithEndpoint(endpoint),
		),
	)
	if err != nil {
		return nil, err
	}

	batchSpanProcessor := trace.NewBatchSpanProcessor(traceExporter)
	tracerProvider := trace.NewTracerProvider(
		trace.WithSampler(trace.AlwaysSample()),
		trace.WithResource(res),
		trace.WithSpanProcessor(batchSpanProcessor),
	)

	return tracerProvider, nil
}

```

```go
// chaoslib/litmus/pod-delete/lib/pod-delete.go
package lib

import (
	"context"
	"fmt"
	"github.com/go-logr/logr"
	"strconv"
	"strings"
	"time"

	"github.com/litmuschaos/litmus-go/pkg/cerrors"
	"github.com/litmuschaos/litmus-go/pkg/clients"
	"github.com/litmuschaos/litmus-go/pkg/events"
	experimentTypes "github.com/litmuschaos/litmus-go/pkg/generic/pod-delete/types"
	"github.com/litmuschaos/litmus-go/pkg/log"
	"github.com/litmuschaos/litmus-go/pkg/probe"
	"github.com/litmuschaos/litmus-go/pkg/status"
	"github.com/litmuschaos/litmus-go/pkg/telemetry"
	"github.com/litmuschaos/litmus-go/pkg/types"
	"github.com/litmuschaos/litmus-go/pkg/utils/common"
	"github.com/litmuschaos/litmus-go/pkg/workloads"
	"github.com/palantir/stacktrace"
	"github.com/sirupsen/logrus"
	"go.opentelemetry.io/otel"
	v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

var logger logr.Logger

func init() {
	logger = logr.New(&telemetry.TracingLoggerWrapper{})
}

// PreparePodDelete contains the preparation steps before chaos injection
func PreparePodDelete(ctx context.Context, experimentsDetails *experimentTypes.ExperimentDetails, clients clients.ClientSets, resultDetails *types.ResultDetails, eventsDetails *types.EventDetails, chaosDetails *types.ChaosDetails) error {
	ctx, span := otel.Tracer(telemetry.TracerName).Start(ctx, "InjectPodDeleteChaos")
	defer span.End()
	logger.Info(fmt.Sprintf("test start!!!!!!!!!!!!!!!!!!!!!"))

	//Waiting for the ramp time before chaos injection
	if experimentsDetails.RampTime != 0 {
		log.Infof("[Ramp]: Waiting for the %vs ramp time before injecting chaos", experimentsDetails.RampTime)
		common.WaitForDuration(experimentsDetails.RampTime)
	}

	//set up the tunables if provided in range
	SetChaosTunables(experimentsDetails)

	log.InfoWithValues("[Info]: The chaos tunables are:", logrus.Fields{
		"PodsAffectedPerc": experimentsDetails.PodsAffectedPerc,
		"Sequence":         experimentsDetails.Sequence,
	})

	switch strings.ToLower(experimentsDetails.Sequence) {
	case "serial":
		if err := injectChaosInSerialMode(ctx, experimentsDetails, clients, chaosDetails, eventsDetails, resultDetails); err != nil {
			return stacktrace.Propagate(err, "could not run chaos in serial mode")
		}
	case "parallel":
		if err := injectChaosInParallelMode(ctx, experimentsDetails, clients, chaosDetails, eventsDetails, resultDetails); err != nil {
			return stacktrace.Propagate(err, "could not run chaos in parallel mode")
		}
	default:
		return cerrors.Error{ErrorCode: cerrors.ErrorTypeGeneric, Reason: fmt.Sprintf("'%s' sequence is not supported", experimentsDetails.Sequence)}
	}

	//Waiting for the ramp time after chaos injection
	if experimentsDetails.RampTime != 0 {
		log.Infof("[Ramp]: Waiting for the %vs ramp time after injecting chaos", experimentsDetails.RampTime)
		common.WaitForDuration(experimentsDetails.RampTime)
	}
	return nil
}

// injectChaosInSerialMode delete the target application pods serial mode(one by one)
func injectChaosInSerialMode(ctx context.Context, experimentsDetails *experimentTypes.ExperimentDetails, clients clients.ClientSets, chaosDetails *types.ChaosDetails, eventsDetails *types.EventDetails, resultDetails *types.ResultDetails) error {

	// run the probes during chaos
	if len(resultDetails.ProbeDetails) != 0 {
		if err := probe.RunProbes(ctx, chaosDetails, clients, resultDetails, "DuringChaos", eventsDetails); err != nil {
			return err
		}
	}

	GracePeriod := int64(0)
	//ChaosStartTimeStamp contains the start timestamp, when the chaos injection begin
	ChaosStartTimeStamp := time.Now()
	duration := int(time.Since(ChaosStartTimeStamp).Seconds())

	for duration < experimentsDetails.ChaosDuration {
		// Get the target pod details for the chaos execution
		// if the target pod is not defined it will derive the random target pod list using pod affected percentage
		if experimentsDetails.TargetPods == "" && chaosDetails.AppDetail == nil {
			return cerrors.Error{ErrorCode: cerrors.ErrorTypeTargetSelection, Reason: "provide one of the appLabel or TARGET_PODS"}
		}

		targetPodList, err := common.GetTargetPods(experimentsDetails.NodeLabel, experimentsDetails.TargetPods, experimentsDetails.PodsAffectedPerc, clients, chaosDetails)
		if err != nil {
			return stacktrace.Propagate(err, "could not get target pods")
		}

		// deriving the parent name of the target resources
		for _, pod := range targetPodList.Items {
			kind, parentName, err := workloads.GetPodOwnerTypeAndName(&pod, clients.DynamicClient)
			if err != nil {
				return stacktrace.Propagate(err, "could not get pod owner name and kind")
			}
			common.SetParentName(parentName, kind, pod.Namespace, chaosDetails)
		}
		for _, target := range chaosDetails.ParentsResources {
			common.SetTargets(target.Name, "targeted", target.Kind, chaosDetails)
		}

		if experimentsDetails.EngineName != "" {
			msg := "Injecting " + experimentsDetails.ExperimentName + " chaos on application pod"
			types.SetEngineEventAttributes(eventsDetails, types.ChaosInject, msg, "Normal", chaosDetails)
			events.GenerateEvents(eventsDetails, clients, chaosDetails, "ChaosEngine")
		}

		//Deleting the application pod
		for _, pod := range targetPodList.Items {

			log.InfoWithValues("[Info]: Killing the following pods", logrus.Fields{
				"PodName": pod.Name})

			if experimentsDetails.Force {
				err = clients.KubeClient.CoreV1().Pods(pod.Namespace).Delete(context.Background(), pod.Name, v1.DeleteOptions{GracePeriodSeconds: &GracePeriod})
			} else {
				err = clients.KubeClient.CoreV1().Pods(pod.Namespace).Delete(context.Background(), pod.Name, v1.DeleteOptions{})
			}
			if err != nil {
				return cerrors.Error{ErrorCode: cerrors.ErrorTypeChaosInject, Target: fmt.Sprintf("{podName: %s, namespace: %s}", pod.Name, pod.Namespace), Reason: fmt.Sprintf("failed to delete the target pod: %s", err.Error())}
			}

			switch chaosDetails.Randomness {
			case true:
				if err := common.RandomInterval(experimentsDetails.ChaosInterval); err != nil {
					return stacktrace.Propagate(err, "could not get random chaos interval")
				}
			default:
				//Waiting for the chaos interval after chaos injection
				if experimentsDetails.ChaosInterval != "" {
					log.Infof("[Wait]: Wait for the chaos interval %vs", experimentsDetails.ChaosInterval)
					waitTime, _ := strconv.Atoi(experimentsDetails.ChaosInterval)
					common.WaitForDuration(waitTime)
				}
			}

			//Verify the status of pod after the chaos injection
			log.Info("[Status]: Verification for the recreation of application pod")
			for _, parent := range chaosDetails.ParentsResources {
				target := types.AppDetails{
					Names:     []string{parent.Name},
					Kind:      parent.Kind,
					Namespace: parent.Namespace,
				}
				if err = status.CheckUnTerminatedPodStatusesByWorkloadName(target, experimentsDetails.Timeout, experimentsDetails.Delay, clients); err != nil {
					return stacktrace.Propagate(err, "could not check pod statuses by workload names")
				}
			}

			duration = int(time.Since(ChaosStartTimeStamp).Seconds())
		}

	}
	log.Infof("[Completion]: %v chaos is done", experimentsDetails.ExperimentName)

	return nil

}

// injectChaosInParallelMode delete the target application pods in parallel mode (all at once)
func injectChaosInParallelMode(ctx context.Context, experimentsDetails *experimentTypes.ExperimentDetails, clients clients.ClientSets, chaosDetails *types.ChaosDetails, eventsDetails *types.EventDetails, resultDetails *types.ResultDetails) error {

	// run the probes during chaos
	if len(resultDetails.ProbeDetails) != 0 {
		if err := probe.RunProbes(ctx, chaosDetails, clients, resultDetails, "DuringChaos", eventsDetails); err != nil {
			return err
		}
	}

	GracePeriod := int64(0)
	//ChaosStartTimeStamp contains the start timestamp, when the chaos injection begin
	ChaosStartTimeStamp := time.Now()
	duration := int(time.Since(ChaosStartTimeStamp).Seconds())

	for duration < experimentsDetails.ChaosDuration {
		// Get the target pod details for the chaos execution
		// if the target pod is not defined it will derive the random target pod list using pod affected percentage
		if experimentsDetails.TargetPods == "" && chaosDetails.AppDetail == nil {
			return cerrors.Error{ErrorCode: cerrors.ErrorTypeTargetSelection, Reason: "please provide one of the appLabel or TARGET_PODS"}
		}
		targetPodList, err := common.GetTargetPods(experimentsDetails.NodeLabel, experimentsDetails.TargetPods, experimentsDetails.PodsAffectedPerc, clients, chaosDetails)
		if err != nil {
			return stacktrace.Propagate(err, "could not get target pods")
		}

		// deriving the parent name of the target resources
		for _, pod := range targetPodList.Items {
			kind, parentName, err := workloads.GetPodOwnerTypeAndName(&pod, clients.DynamicClient)
			if err != nil {
				return stacktrace.Propagate(err, "could not get pod owner name and kind")
			}
			common.SetParentName(parentName, kind, pod.Namespace, chaosDetails)
		}
		for _, target := range chaosDetails.ParentsResources {
			common.SetTargets(target.Name, "targeted", target.Kind, chaosDetails)
		}

		if experimentsDetails.EngineName != "" {
			msg := "Injecting " + experimentsDetails.ExperimentName + " chaos on application pod"
			types.SetEngineEventAttributes(eventsDetails, types.ChaosInject, msg, "Normal", chaosDetails)
			events.GenerateEvents(eventsDetails, clients, chaosDetails, "ChaosEngine")
		}

		//Deleting the application pod
		for _, pod := range targetPodList.Items {

			log.InfoWithValues("[Info]: Killing the following pods", logrus.Fields{
				"PodName": pod.Name})

			if experimentsDetails.Force {
				err = clients.KubeClient.CoreV1().Pods(pod.Namespace).Delete(context.Background(), pod.Name, v1.DeleteOptions{GracePeriodSeconds: &GracePeriod})
			} else {
				err = clients.KubeClient.CoreV1().Pods(pod.Namespace).Delete(context.Background(), pod.Name, v1.DeleteOptions{})
			}
			if err != nil {
				return cerrors.Error{ErrorCode: cerrors.ErrorTypeChaosInject, Target: fmt.Sprintf("{podName: %s, namespace: %s}", pod.Name, pod.Namespace), Reason: fmt.Sprintf("failed to delete the target pod: %s", err.Error())}
			}
		}

		switch chaosDetails.Randomness {
		case true:
			if err := common.RandomInterval(experimentsDetails.ChaosInterval); err != nil {
				return stacktrace.Propagate(err, "could not get random chaos interval")
			}
		default:
			//Waiting for the chaos interval after chaos injection
			if experimentsDetails.ChaosInterval != "" {
				log.Infof("[Wait]: Wait for the chaos interval %vs", experimentsDetails.ChaosInterval)
				waitTime, _ := strconv.Atoi(experimentsDetails.ChaosInterval)
				common.WaitForDuration(waitTime)
			}
		}

		//Verify the status of pod after the chaos injection
		log.Info("[Status]: Verification for the recreation of application pod")
		for _, parent := range chaosDetails.ParentsResources {
			target := types.AppDetails{
				Names:     []string{parent.Name},
				Kind:      parent.Kind,
				Namespace: parent.Namespace,
			}
			if err = status.CheckUnTerminatedPodStatusesByWorkloadName(target, experimentsDetails.Timeout, experimentsDetails.Delay, clients); err != nil {
				return stacktrace.Propagate(err, "could not check pod statuses by workload names")
			}
		}
		duration = int(time.Since(ChaosStartTimeStamp).Seconds())
	}

	log.Infof("[Completion]: %v chaos is done", experimentsDetails.ExperimentName)

	return nil
}

// SetChaosTunables will setup a random value within a given range of values
// If the value is not provided in range it'll setup the initial provided value.
func SetChaosTunables(experimentsDetails *experimentTypes.ExperimentDetails) {
	experimentsDetails.PodsAffectedPerc = common.ValidateRange(experimentsDetails.PodsAffectedPerc)
	experimentsDetails.Sequence = common.GetRandomSequence(experimentsDetails.Sequence)
}

```
