---
layout: post
title: "single-millis-counter"
slug: uvcounter
category: essay
---

## Golang low latency application

### Latency & Throughput Optimization

다른 제품의 기반이 되는 플랫폼은 편하게 부담없이 사용하는게 매우 중요하다. 우리가 사용하는 제품도 동일하다. 텍스트 에디터에서 100ms의 지연이 1분마다 발생한다면 어떨까? 웹 페이지를 이동할 때 어떤 페이지는 첫 화면이 1초가 넘게 걸린다면 어떨까? 사용하는데 전혀 문제는 없지만 기분이 매우 안좋아진다. 이렇듯 latency는 사용성에 매우 중요한 요소이다.

tail latency는 전체 요청 중 상위 0.01%에 해당하는 지연 시간(p99.99)과 같이 드물게 발생하지만, 사용자 경험에 큰 영향을 줄 수 있다.

### h2load test

성능 테스트를 할 때 tail latency는 다양한 원인으로 늘어날 수 있다.

흔히 하는 실수는 성능 테스트를 하는 노드가 분리되어있지 않은 경우이다. 머신 한대를 받아서 성능 테스트를 해본다고 하자. go process가 http 서버 형태로 떠있고, 같은 노드에서 h2load를 사용해서 localhost로 요청을 보내는 경우 정확한 측정이 가능할까??

당연히 아니다. latency는 대부분의 경우 queue에서 대기하는 경우에 발생한다. 우리는 이 테스트에서 cpu 자원을 공유하고 있고, 같은 cpu core를 공유하기에 경쟁이 생길 수 있다. run queue에 여러 프로세스가 대기하고 있다면 다른 process는 cpu 자원을 할당받지 못한 시간동안 지연이 발생한다.

이러한 지연은 profiling으로는 빠르게 찾기 어려울 수 있는데, 어떠한 syscall, 혹은 함수에서 느려지는 것을 발견하면 이게 단순히 지연인 것인지 자원 경합인 것인지 process레벨에서는 확인하지 못하기 때문이다.

나는 간단하게 run queue를 모니터링해보는 것을 추천하는데, 내가 사랑하는 sysstat도구중 하나인 sar를 사용하는 방법이다. `sar -q 1`를 실행하면 cpu의 queue를 1초마다 출력해준다. 만약 노드의 cpu가 모든 코어에서 100%를 사용하는 현상을 발견했다면 이러한 도구를 사용해보자. (모든 core가 일하고 있는 것은 `mpstat 1 -P ALL`으로 확인할 수 있다.)

run queue는 다른 방식으로도 모니터링할 수 있는데, 매우 강력한 eBPF를 사용하면 좋다. runqlat와 같은 도구는 run queue자체보다는 작업이 얼마나 대기하고 실행되는지에 대한 메트릭을 볼 수 있다. queue에 차는 것 보다 이 방법이 더 정확하지만, 사용하는 것에 제약이 있다는 것이 단점이다.


### go trace with pprof

가장 쉽게 할 수 있는 선택은 pprof를 사용한 trace분석이다. trace를 수집한 다음 어떤 goroutine이 얼마나 걸렸는지 확인할 수 있다.

- What is Go Execution Tracer
    
    perf, profiling, heap dumper의 경우에는 aggregate된 데이터만 보여주거나, 현재 상태를 보여주는데 주력합니다. go를 사용하는 대부분의 목적은 낮은 tail latency를 위하는 것이기에 이러한 정보는 크게 도움이 되지 않는 경우가 있습니다.
    
    **execution tracer에서는 goroutine이 어디서 만들어지고, 어디에서 기다리고, 어떻게 흘러가는지에 대한 정보를 제공합니다. go runtime에서 이러한 latency를 위한 정보들을 거의 100% 수집함.**
    
    [design doc](https://docs.google.com/document/d/1FP5apqzBgr7ahCCgFO-yoVhk4YZrNIDNf9RybngBc14/edit?tab=t.0)
    
- async profiler랑 다른점? tracing?
    
    profiling은 보통 cpu나 자원을 어디에서 많이 사용했는지를 추적하는것과 다르게 goroutine의 상태 전이, blocking, 의존 관계에 초점을 맞추어서 분석할 수 있습니다. async profiler의 wall clock과도 비슷하지만, aggregate되지 않고 실제 goroutine id를 보면서 분석할 수 있어요.
    
- pprof의 오버헤드는 괜찮을까?
    
    https://github.com/golang/go/issues/65208 를 찾아보면 overhead는 문서화 되지는 않았지만, 10년전에는 약 5%의 오버헤드로 알려짐.
    
    https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap/ 트위치의 블로그에서는 프로덕션에서 매번 호출해도 문제 없을 정도의 오버헤드라고 함.
    
    https://blog.felixge.de/reducing-gos-execution-tracer-overhead-with-frame-pointer-unwinding/1.21에서는 profiling하는데 사용되는 cpu가 1~2%정도라고 함.
    
- 어떻게 사용할까?
    
    pprof 서버를 따로 띄운다음에 `/debug/pprof/trace?seconds=N` 을 호출하면 N초 동안 실행된 trace가 남는다. (wall clock profiling과 동일함)
    
    이렇게 만들어진 file을 갖고 `go tool trace FILE` 이렇게 하면 web ui가 열리면서 timeline에 따라서 볼 수 있다. 
    
    [gotraceui](https://gotraceui.dev/)라는 도구를 사용할 수도 있다. (개인적으로 둘 다 같이 사용하는것 추천)
    
- 근데 손으로 떠야하면 latency 이슈가 있을때를 어떻게 예측해서 뜨지?
    
    latency가 늘어나는 것을 보고 tracer를 뜬다고 하더라도 이미 지나간 뒤다. 그래서 latency가 늘어나는 요청이 언제일지 예측할 수 없다면 계속 직접 뜨고있어야하는데, 한달에 한번 있는거라면? 거의 불가능하기때문에 새로운 기법을 제안한다.
    
    go 1.23부터는 tracer의 cpu 사용이 1~2%로 매우 적기때문에 항상 trace를 하고있으면서 고정된 ring buffer에 trace를 저장하고있는다. latency가 늘어나는 요청이 탐지되면 메모리에 있던 trace를 file에 write하는 것으로 사람의 개입없이 문제가 있는 요청을 정확히 탐지 가능하다.
    
    https://go.dev/blog/execution-traces-2024
