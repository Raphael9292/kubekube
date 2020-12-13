# 목표
prometheus를 사용하여 모니터링을 한다.
- Datadog 제거


### [Kubernetes monitoring architecture](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/monitoring_architecture.md#kubernetes-monitoring-architecture)

- #### Metrics 분류

    - 시스템 메트릭: 시스템 메트릭은 노드나 컨테이너의 CPU, 메모리 사용량 같은 일반적인 시스템 관련 메트릭들입니다. 시스템 메트릭은 다시 코어 메트릭(core metrics)과 비코어메트릭(non-core metrics)으로 나뉩니다. 코어 메트릭은 쿠버네티스 내부 컴포넌트들이 사용하는 메트릭들입니다. 현재 클러스터내의 가용자원이 얼마나 되는지 파악하는데도 사용하고 내장된 오토스케일링등에서도 사용합니다. 대시보드에서도 사용하고 “kubectl top” 같은 명령어에서 사용하는 메트릭 데이터도 여기의 값입니다. CPU/메모리 사용량, 포드/컨테이너의 디스크 사용량등이 코어 메트릭입니다. 비코어메트릭은 쿠버네티스가 직접 사용하지 않는 다른 시스템 메트릭들을 의미합니다.

    - 서비스 메트릭: 서비스 메트릭은 애플리케이션을 모니터링 하는데 필요한 메트릭들입니다. 서비스 메트릭은 다시 2가지로 나눠서 생각할 수 있습니다. 쿠버네티스 인프라용 컨테이너들에서 나오는 메트릭들과 사용자의 애플리케이션에서 나오는 메트릭입니다. 쿠버네티스 인프라컨테이너들에서 나오는 메트릭은 클러스터를 관리할 때 참고해서 사용할 수 있습니다. 사용자 애플리케이션의 메트릭들은 웹서버의 응답속도(response time)에 대한 값이라던지 시간당 HTTP 500에러가 몇건이나 나타나는지등의 서비스 관련된 정보를 파악할 수 있는 메트릭 들입니다. 서비스 메트릭은 커스텀 메트릭으로 사용되서 HPA에서 사용할 수 있습니다.
  
- #### 파이프라인 분류
    - 코어 메트릭 파이프라인: 코어 메트릭 파이프라인은 쿠버네티스에서 관련 구성요소들을 직접 관리하는 파이프 라인이고 말 그대로 핵심 요소들에 대한 모니터링을 담당합니다. kubelet, metrics-server, metric API 등으로 구성되어 있습니다. 여기서 제공하는 메트릭들은 시스템 컴포넌트들에서 사용합니다. 주로 스케줄러나  HPA(Horizontal Pod Autoscaling, 수평적 포드 오토스케일링)등의 기초 자료로 활용됩니다. 별도의 외부 서드파티 모니터링 시스템과 연계되지 않고 독립적으로 운영됩니다.
      코어 메트릭 파이프파인은 코어 시스템 메트릭들을 수집합니다. kubelet에 내장된 cAdvisor를 통해서 노드/포드/컨테이너의 사용량 정보를 수집합니다. 메트릭서버(metrics-server)가 이 정보들을 kubelet에서 가져와서 메모리에 저장하고 있습니다. 메모리에 저장하기 때문에 저장 용량의 한계가 있기 때문이 짧은 기간의 데이터만 보관하고 있게 됩니다. 그리고 이렇게 저장하고 메트릭 정보들을 마스터 메트릭 API를 통해서 다른 시스템 컴포넌트들이 조회할 수 있게 됩니다.   
         
    - 모니터링 파이프라인: 모니터링 파이프라인은 기본 메트릭 뿐만아니라 다양한 여러가지 메트릭을 수집합니다. 여기서 수집한 메트릭들은 쿠버네티스 시스템이 사용하기보다는 클러스터의 사용자들이 필요한 모니터링을 하는데 사용합니다. 쿠버네티스는 모니터링파이프라인 쪽은 직접 관리하지 않습니다. 이런 부분까지 쿠버네티스에서 직접 개발/유지 하려면 부담이 크기 때문에 외부 모니터링 시스템을 연계해서 이용하도록 가이드 하고 있습니다.
      모니터링 파이프라인에서는 시스템 메트릭과 서비스 메트릭 둘 다 수집할 수 있습니다. 코어 메트릭 파이프라인과는 분리되어 있기 때문에 원하는대로 모니터링 솔루션을 선택해서 사용하면 됩니다. 모니터링 파이프 라인으로 사용가능한 조합은 다음 처럼 여러가지가 있습니다.
      - Metrics Server가 제공하지 않는 모니터링 데이터를 수집하기 위해서는 Custom Metrics를 API로서 등록하고 이를 Prometheus에 저장해야 합니다. 예를 들어, 애플리케이션 레벨에서의 HTTP 요청 횟수 등을 수집하고 사용하기 위해서는 해당 데이터를 Export하는 별도의 로직을 애플리케이션에서 구현한 뒤, 이를 Prometheus에 저장하는 방식으로 동작합니다. 저장된 HTTP 요청 횟수 데이터는 Prometheus Adapter에 의해 쿼리되어 쿠버네티스의 Custom Metrics API (custom.metrics.k8s.io) 로서 HPA에 제공됩니다.



  물론 지금은 힙스터가 너무 무거워서 사장되고 `metrics-server` 를 사용하는게 맞음. (코어 메트릭 파이프라인)

### Components
- [Node Exporter](https://prometheus.io/docs/guides/node-exporter/)
  - 하드웨어 및 커널 관련 메트릭을 수집
  
- [Alert Manager](https://prometheus.io/docs/alerting/latest/alertmanager/)
  - 프로메테우스에서 다른 미들웨어로 보내는 역할
  
- [Prometheus Adapter](https://github.com/helm/charts/tree/master/stable/prometheus-adapter)
    - Prometheus Adapter는 Custom Metrics를 별도의 API 서비스로서 Aggregation Layer에 등록한 뒤, 원하는 모니터링 데이터를 Prometheus로부터 PromQL의 형식을 통해 가져와 Custom Metrics API에 제공하는 역할을 담당합니다. 즉 HPA가 Custom Metrics를 사용하기 위해서는 반드시 Prometheus Adapter가 필요.
