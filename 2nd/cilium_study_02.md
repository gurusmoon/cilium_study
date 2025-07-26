

# Cilium Study - Week 2

이 문서는 CloudNet@ 팀에서 진행한 Cilium Study 내용을 정리한 자료입니다. 본 스터디를 제공해주신 가시다님과 CloudNet@ 팀 모든 분들께 감사드립니다.

## 목차

### [실습 환경 구성](#1-실습-환경-구성)
- [x] 실습 환경 배포
- [x] 클러스터 점검
- [x] 네트워크 설정 검증

### [Cilium 관측 도구 구성](#2-cilium-관측-도구-구성)
- [x] Hubble 설치 및 설정
- [x] Prometheus & Grafana 구성
- [x] 메트릭스 수집 설정

### [네트워크 정책 실습](#3-네트워크-정책-실습)
- [x] L3/L4 정책 적용
- [x] L7 정책 구성
- [x] 트래픽 모니터링

## 1. 실습 환경 소개

> **실습 환경 구성 요약**
> - Vagrant로 구성하는 3노드 쿠버네티스 클러스터
> - Cilium CNI가 설치된 완전한 네트워킹 환경
> - 네트워크 정책 및 모니터링 실습을 위한 기반 환경

### 1.1 클러스터 구성

#### 컨트롤 플레인
- 노드명: `k8s-ctr`
- 역할: 클러스터 관리 및 제어
  
#### 워커 노드
- 노드명: `k8s-w1`, `k8s-w2`
- 역할: 워크로드 실행 및 네트워크 정책 적용

### 1.2 주요 컴포넌트

| 컴포넌트 | 버전 | 비고 |
|---------|------|------|
| **Kubernetes** | v1.33.2-1.1 | 클러스터 오케스트레이션 |
| **Container Runtime** | containerd v1.7.27-1 | 컨테이너 실행 환경 |
| **CNI** | Cilium | kube-proxy 대체 모드 |

---

## 2. 실습 환경 배포 🚀

### 2.1 배포 파일 구성

> **배포 자동화 개요**
> - Vagrant를 사용한 멀티노드 클러스터 프로비저닝
> - 쉘 스크립트를 통한 컴포넌트 자동 설치
> - Cilium CNI 및 관련 도구 설정 자동화

#### 주요 파일 구성

##### Vagrantfile
| 구성 요소 | 설명 |
|----------|------|
| 가상 머신 스펙 | CPU, 메모리, 디스크 정의 |
| 네트워크 설정 | 인터페이스 및 네트워크 구성 |
| 스크립트 연동 | 초기화 스크립트 실행 설정 |

##### 초기화 스크립트
| 스크립트 | 역할 | 실행 시점 |
|----------|------|-----------|
| `init_cfg.sh` | 기본 환경 구성 | 최초 실행 |
| `k8s-ctr.sh` | 컨트롤 플레인 초기화 | 마스터 노드 |
| `k8s-w.sh` | 워커 노드 조인 | 워커 노드 |

### 2.2 클러스터 배포

> **배포 절차**
> 1. 실습 디렉토리 생성
> 2. Vagrant 설정 파일 다운로드
> 3. 가상 머신 프로비저닝 실행

#### 배포 명령어
```bash
# 실습 환경 구성
mkdir cilium-lab && cd cilium-lab
curl -O https://raw.githubusercontent.com/gasida/vagrant-lab/refs/heads/main/cilium-study/2w/Vagrantfile
vagrant up
```

#### 배포 순서
1. 디렉토리 생성 및 이동
2. Vagrant 설정 파일 다운로드
3. 가상 머신 프로비저닝 시작
4. 자동화된 클러스터 구성


⸻

### 2.3 클러스터 초기 점검

#### 2.3.1 기본 점검 사항

| 점검 영역 | 확인 항목 | 방법 |
|-----------|-----------|------|
| 노드 상태 | 전체 노드 Ready | `kubectl get nodes` |
| 파드 상태 | 시스템 파드 Running | `kubectl get pods -A` |
| 네트워크 | CNI 동작 확인 | `cilium status` |

> **주요 검증 포인트**
> - 호스트 네임 해석 설정
> - 노드 간 SSH 연결성
> - 네트워크 인터페이스 상태
> - 쿠버네티스 클러스터 구성
> - 시스템 네트워크 설정

#### 2.3.2 기본 연결성 확인
```bash
# 호스트 파일 설정 확인
cat /etc/hosts
```

| 확인 사항 | 설명 | 기대 결과 |
|----------|------|-----------|
| 호스트 엔트리 | 노드별 IP 주소 매핑 | 모든 노드 등록 |
| 도메인 해석 | 클러스터 도메인 설정 | 내부 DNS 동작 |

#### 2.3.2 노드 연결성 검증
```bash
# 워커 노드 SSH 접속 테스트
sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-w1 hostname
sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-w2 hostname
```

#### 2.3.3 네트워크 구성 확인
```bash
# 네트워크 인터페이스 상태
ifconfig | grep -iEA1 'eth[0-9]:'
```
```

#### 2.3.4 클러스터 구성 검증

##### 기본 정보 확인
```bash
# 클러스터 기본 정보
kubectl cluster-info

# 네트워크 CIDR 설정
kubectl cluster-info dump | grep -m 2 -E "cluster-cidr|service-cluster-ip-range"
```

##### 설정 검증
```bash
# kubeadm 설정 확인
kubectl describe cm -n kube-system kubeadm-config
kubectl describe cm -n kube-system kubelet-config

# 노드 상태 확인
kubectl get node -o wide
```

##### 노드별 설정
```bash
# kubelet 인자 확인
cat /var/lib/kubelet/kubeadm-flags.env
for i in w1 w2 ; do
  echo ">> node : k8s-$i <<"
  sshpass -p 'vagrant' ssh vagrant@k8s-$i cat /var/lib/kubelet/kubeadm-flags.env
  echo
done
```

##### 네트워크 검증
```bash
# CIDR 및 IP 확인
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
kubectl get ciliumnode -o json | grep podCIDRs -A2
kubectl get pod -A -o wide

# iptables 설정
iptables-save
iptables -t nat -S
iptables -t filter -S
iptables -t mangle -S
```

| 검증 항목 | 확인 내용 | 방법 |
|----------|-----------|------|
| 클러스터 상태 | 컨트롤 플레인 동작 | `cluster-info` |
| 네트워크 설정 | CIDR 할당 | CIDR 범위 확인 |
| 노드 구성 | kubelet 설정 | 설정 파일 검사 |
| 네트워크 규칙 | iptables 상태 | 규칙 테이블 확인 |


⸻

### 2.4 Cilium 상태 점검

> **점검 범위**
> - Cilium 바이너리 설치 확인
> - 설정 및 상태 검증
> - 메트릭 및 모니터링 상태

#### 2.4.1 기본 설치 확인
```bash
# Cilium 바이너리 및 상태
which cilium
cilium status
cilium config view
kubectl get cm -n kube-system cilium-config -o json | jq
```

#### 2.4.2 상세 상태 검증
```bash
# 설정 및 메트릭
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg config
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg status --verbose
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg metrics list
```

#### 2.4.3 모니터링 구성
```bash
# 엔드포인트 확인
kubectl get ciliumendpoints -A

# 트래픽 모니터링
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg monitor
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg monitor -v
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg monitor -v -v
```

| 검증 항목 | 명령어 | 확인 내용 |
|----------|--------|-----------|
| 바이너리 | `which cilium` | 설치 위치 |
| 기본 상태 | `cilium status` | 동작 상태 |
| 상세 설정 | `cilium config view` | 설정 내용 |
| 메트릭 | `cilium-dbg metrics` | 성능 지표 |

#### 2.4.4 고급 모니터링 기능

##### 엔드포인트 상세 분석
```bash
# 특정 엔드포인트 모니터링
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- \
  cilium-dbg monitor --related-to=<id>

# 드롭된 패킷 확인
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- \
  cilium-dbg monitor --type drop

# 16진수 패킷 덤프
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- \
  cilium-dbg monitor -v -v --hex
```

#### 2.4.5 L7 트래픽 모니터링

> **L7 모니터링 개요**
> - HTTP 트래픽 실시간 분석
> - 애플리케이션 레벨 정책 검증
> - 요청/응답 상세 확인

```bash
# L7 트래픽 모니터링
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- \
  cilium-dbg monitor -v --type l7
```

| 모니터링 유형 | 용도 | 명령어 옵션 |
|--------------|------|-------------|
| 엔드포인트 | 특정 ID 트래픽 | `--related-to` |
| 드롭 패킷 | 차단된 트래픽 | `--type drop` |
| 패킷 덤프 | 상세 패킷 분석 | `--hex` |
| L7 트래픽 | HTTP 분석 | `--type l7` |

## 3. Cilium 관측 도구 구성

### 3.1 Hubble 설치 및 설정

> **Hubble 플랫폼 개요**
> - Cilium의 네트워크 관측 도구
> - 실시간 트래픽 모니터링
> - 트래픽 플로우 시각화/분석
> - 보안 정책 검증/디버깅

#### 3.1.1 사전 환경 점검

| 점검 항목 | 확인 내용 | 중요도 |
|----------|-----------|---------|
| Cilium 상태 | 구성 및 동작 상태 | 필수 |
| 인증서/Secret | 보안 컴포넌트 | 필수 |
| 포트 상태 | 서비스 포트 할당 | 필수 |

> **주의 사항**
> - 모든 노드의 Cilium Agent 정상 동작 확인
> - 필요한 포트 미사용 상태 확인
> - 인증서 갱신 주기 확인

```bash
```bash
#### 3.1.2 설치 절차

```bash
# 1. Hubble UI 활성화
microk8s kubectl patch configmap -n kube-system cilium-config \
  --type merge \
  --patch '{"data":{"hubble-ui":"{\"enabled\":true}"}}'

# 2. 포드 상태 확인
microk8s kubectl get pods -n kube-system
```

#### 3.1.3 UI 접속 설정

| 설정 | 값 | 설명 |
|------|-----|------|
| 포트 포워딩 | `kubectl port-forward -n kube-system svc/hubble-ui 12000:80` | UI 접속용 포트 설정 |
| 접속 URL | `http://localhost:12000` | 웹 브라우저 접속 주소 |

> **접속 팁**
> - 브라우저 개발자 도구로 연결 상태 확인
> - UI 로드 실패 시 포드 로그 확인
> - 방화벽 설정 점검

### 3.2 Prometheus & Grafana 구성

#### 3.2.1 모니터링 스택 개요

| 컴포넌트 | 역할 | 주요 기능 |
|---------|------|-----------|
| Prometheus | 메트릭 수집 | 시계열 데이터 저장/조회 |
| Grafana | 데이터 시각화 | 대시보드 및 알람 관리 |
| AlertManager | 알림 관리 | 알림 규칙 및 라우팅 |

> **구성 특징**
> - 클러스터 전반의 메트릭 수집
> - Cilium 성능 지표 모니터링
> - 커스텀 대시보드 지원
#### 3.2.2 설치 절차

##### A. Prometheus 설치

```bash
# 1. Helm 저장소 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 2. Prometheus 설치
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

##### B. Grafana 구성

| 설정 | 값 | 설명 |
|------|-----|------|
| 서비스 타입 | LoadBalancer/NodePort | 외부 접속 설정 |
| 기본 포트 | 3000 | Grafana UI 포트 |
| 기본 계정 | admin/prom-operator | 초기 로그인 정보 |

```bash
# Grafana 서비스 확인
kubectl get svc -n monitoring prometheus-grafana
```

#### 3.2.3 메트릭 수집 설정

##### A. ServiceMonitor 설정

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cilium-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      k8s-app: cilium
  namespaceSelector:
    matchNames:
    - kube-system
  endpoints:
  - port: metrics
    interval: 10s
```

##### B. 대시보드 임포트

| 대시보드 ID | 설명 | 주요 메트릭 |
|------------|------|------------|
| 13537 | 기본 Hubble | 트래픽 플로우 |
| 13538 | Cilium Operator | 운영 지표 |
| 13539 | DNS/HTTP 지표 | L7 프로토콜 |


#### 3.2.4 알람 설정

##### A. 기본 알람 규칙

| 알람 | 조건 | 심각도 |
|------|------|--------|
| HighErrorRate | 에러율 > 10% | Critical |
| PodRestarts | 재시작 > 5회 | Warning |
| NetworkLatency | 지연 > 1s | Warning |

##### B. 알람 설정 예시

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cilium-alerts
  namespace: monitoring
spec:
  groups:
  - name: cilium.rules
    rules:
    - alert: CiliumHighErrorRate
      expr: rate(cilium_drop_count_total[5m]) > 0.1
      for: 5m
      labels:
        severity: critical
      annotations:
        description: "Cilium drop rate is high"
```

#### 3.2.5 메트릭 검증

##### A. Prometheus 타겟 확인
```bash
# 타겟 상태 확인
kubectl port-forward svc/prometheus-operated 9090:9090 -n monitoring
```

##### B. 주요 메트릭

| 메트릭 | 설명 | PromQL |
|--------|------|--------|
| 패킷 드롭 | 차단된 패킷 수 | `rate(cilium_drop_count_total[5m])` |
| 연결 상태 | 활성 연결 수 | `cilium_connection_active_total` |
| 정책 평가 | 정책 평가 횟수 | `cilium_policy_verdict_total` |

> **모니터링 팁**
> - 대시보드 필터링으로 특정 네임스페이스 관찰
> - 알람 임계치는 환경에 맞게 조정
> - 히스토리컬 데이터 보존 기간 설정

### 3.3 Hubble 고급 설정

#### 3.3.1 설치 옵션 비교

| 설치 방식 | 특징 | 사용 사례 |
|-----------|-------|-----------|
| Helm 차트 | 전체 기능 설치 | 프로덕션 환경 |
| Cilium CLI | 기본 기능 설치 | 테스트/개발 환경 |

#### 3.3.2 Helm 차트 고급 설정
```bash
# Hubble 전체 기능 설치
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.ui.service.type=NodePort \
  --set hubble.ui.service.nodePort=31234 \
  --set hubble.export.static.enabled=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,icmp,http}"
```

| 설정 옵션 | 설명 | 기본값 |
|-----------|------|---------|
| hubble.enabled | Hubble 활성화 | false |
| relay.enabled | Relay 서비스 | false |
| ui.service.type | 서비스 유형 | ClusterIP |
| metrics.enabled | 수집 메트릭 | [] |

#### 3.3.3 CLI 기반 설치

```bash
# 1. 기본 설치
cilium hubble enable

# 2. UI 포함 설치
cilium hubble enable --ui
```


#### 3.3.4 설치 검증

##### A. 상태 확인

# Relay 상태 확인
cilium status
# → “Hubble Relay: OK” 메시지 확인

# Hubble 설정 반영 확인
cilium config view | grep -i hubble
kubectl get cm -n kube-system cilium-config -o json | grep -i hubble

# Secret 확인
kubectl get secret -n kube-system | grep -iE 'cilium-ca|hubble'

### 3.4 네트워크 정책 실습

#### 3.4.1 L3/L4 정책 개요

| 정책 수준 | 제어 대상 | 예시 |
|----------|-----------|------|
| L3 | IP 주소/CIDR | 특정 네트워크 접근 제어 |
| L4 | 포트/프로토콜 | 서비스 포트 접근 제어 |

> **정책 특징**
> - IP 기반 네트워크 분리
> - 포트 레벨 접근 제어
> - 프로토콜 기반 필터링
> - 레이블 기반 선택자 지원

#### 3.4.2 기본 정책 설정

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l3-l4-policy
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
```

| 정책 요소 | 설명 | 예시 |
|----------|------|------|
| endpointSelector | 정책 적용 대상 | app=backend |
| fromEndpoints | 허용할 출발지 | app=frontend |
| toPorts | 허용할 포트 | 8080/TCP |

#### 3.4.3 정책 검증 방법

```bash
# 1. 정책 상태 확인
kubectl get cnp

# 2. 정책 상세 확인
kubectl describe cnp l3-l4-policy
```


### 3.5 L7 정책 구성 🌐

#### 3.5.1 L7 정책 개요 📝

| 특징 | 설명 | 예시 |
|------|------|------|
| 프로토콜 인식 | HTTP/DNS/gRPC 등 | 메서드/경로 필터링 |
| 세부 제어 | 요청/응답 제어 | 헤더/쿠키 검사 |
| 보안 강화 | 애플리케이션 보호 | API 접근 제한 |

#### 3.5.2 HTTP 정책 예시 ⚡️

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l7-policy
spec:
  endpointSelector:
    matchLabels:
      app: api-server
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: client
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: GET
          path: "/api/v1/products"
#### 3.5.3 정책 구성 요소 📋

| 구성 | 설명 | 예시 |
|------|------|------|
| 프로토콜 | L7 프로토콜 지정 | HTTP/DNS/Kafka |
| 메서드 | 허용할 HTTP 메서드 | GET/POST/PUT |
| 경로 | URL 패턴 매칭 | /api/v1/* |
| 헤더 | HTTP 헤더 규칙 | Authorization |

#### 3.5.4 검증 및 모니터링 🔍

```bash
# 1. 정책 적용 확인
kubectl get cnp l7-policy

# 2. 트래픽 모니터링
cilium monitor --type l7
```


### 3.6 트래픽 모니터링 🔍

#### 3.6.1 모니터링 도구 📊

| 도구 | 용도 | 특징 |
|------|------|------|
| Hubble UI | 시각화 | 플로우 그래프 |
| cilium monitor | 실시간 분석 | 상세 로그 |
| Prometheus | 메트릭 수집 | 시계열 데이터 |

#### 3.6.2 모니터링 명령어 ⌨️

```bash
# 1. 일반 트래픽 모니터링
cilium monitor

# 2. L7 트래픽 모니터링
cilium monitor --type l7

# 3. 특정 엔드포인트 모니터링
cilium monitor --related-to ENDPOINT_ID

# 4. 드롭된 패킷 모니터링
cilium monitor --type drop
```

#### 3.6.3 모니터링 분석 팁 💡

| 상황 | 확인 사항 | 해결 방법 |
|------|-----------|-----------|
| 연결 거부 | 정책 매칭 여부 | 레이블/정책 검토 |
| L7 오류 | 프로토콜 설정 | 정책 규칙 확인 |
| 성능 저하 | 메트릭 추이 | 리소스 조정 |

> 🔍 **디버깅 체크리스트**
> - 엔드포인트 상태 확인
> - 정책 적용 상태 검증
> - 로그 레벨 조정
> - 메트릭 대시보드 활용

⸻

이것으로 Cilium Study Week 2 문서를 마칩니다. 
더 자세한 내용은 [Cilium 공식 문서](https://docs.cilium.io)를 참조하세요.

### 3.7 Hubble Client 구성 🖥️

#### 3.7.1 Client 설정 개요 📝

| 구성 요소 | 설명 | 용도 |
|----------|------|------|
| API 접근 | 로컬 Hubble API | 상태 조회/제어 |
| Relay 연결 | 포트 포워딩 | 클러스터 연결 |
| CLI 도구 | 명령행 인터페이스 | 모니터링/분석 |

#### 3.7.2 Client 연결 설정 ⚡️

```bash
# 1. Relay 연결 설정
cilium hubble port-forward&

# 2. 연결 상태 확인
ss -tnlp | grep 4245

# 3. API 상태 검증
hubble status
```

| 상태 정보 | 설명 | 예시 값 |
|----------|------|---------|
| Healthcheck | API 연결 상태 | Ok |
| Flows | 현재/최대 플로우 | 12,285/12,285 |
| Rate | 초당 플로우 수 | 41.20/s |

#### 3.7.3 Client 설정 관리 ⚙️

```bash
# 1. 설정 확인
hubble config view 

# 2. 트래픽 모니터링
hubble observe -f
```

| 명령어 | 용도 | 옵션 |
|--------|------|-------|
| config view | 설정 확인 | 서버/포트 설정 |
| observe | 트래픽 관찰 | -f: 실시간 모드 |
| status | 상태 확인 | --server: 서버 지정 |


⸻

### 3.8 고급 운영 관리 🔧

#### 3.8.1 운영 환경 설정 ⚙️

| 구성 요소 | 설명 | 용도 |
|----------|------|------|
| 환경 변수 | Pod 식별자 | 노드별 접근 |
| Alias | 명령어 단축키 | 빠른 실행 |
| BPF 도구 | 저수준 접근 | 상세 분석 |

```bash
# 1. Pod 환경변수 설정
export CILIUMPOD0=$(kubectl get -l k8s-app=cilium pods -n kube-system \
  --field-selector spec.nodeName=k8s-ctr -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD1=$(kubectl get -l k8s-app=cilium pods -n kube-system \
  --field-selector spec.nodeName=k8s-w1 -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD2=$(kubectl get -l k8s-app=cilium pods -n kube-system \
  --field-selector spec.nodeName=k8s-w2 -o jsonpath='{.items[0].metadata.name}')

# 2. Cilium 명령어 단축키
alias c0="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- cilium"
alias c1="kubectl exec -it $CILIUMPOD1 -n kube-system -c cilium-agent -- cilium"
alias c2="kubectl exec -it $CILIUMPOD2 -n kube-system -c cilium-agent -- cilium"

# 3. BPF 도구 단축키
alias c0bpf="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- bpftool"
alias c1bpf="kubectl exec -it $CILIUMPOD1 -n kube-system -c cilium-agent -- bpftool"
alias c2bpf="kubectl exec -it $CILIUMPOD2 -n kube-system -c cilium-agent -- bpftool"
```
c0 endpoint list -o json
c1 endpoint list
c2 endpoint list

c1 endpoint get %3Cid%3E
c1 endpoint log <id>

## Enable debugging output on the cilium-dbg monitor for this endpoint
c1 endpoint config <id> Debug=true


# monitor
c1 monitor
c1 monitor -v
c1 monitor -v -v

## Filter for only the events related to endpoint
c1 monitor --related-to=<id>

## Show notifications only for dropped packet events
c1 monitor --type drop

## Don’t dissect packet payload, display payload in hex information
c1 monitor -v -v --hex

## Layer7
c1 monitor -v --type l7


#### 3.8.6 IP 및 Identity 관리 🔑

| 관리 항목 | 명령어 | 설명 |
|----------|--------|------|
| IP 주소 | `c0 ip list` | IP 목록 조회 |
| Identity | `c0 identity list` | 식별자 관리 |
| 엔드포인트 | `c0 endpoint get` | 상세 정보 |
| BPF | `c0 bpf fs show` | 파일시스템 |

```bash
# 1. IP 관리
c0 ip list
c0 ip list -n    # Identity 포함

# 2. Identity 관리
c0 identity list           # 전체 조회
c0 identity list --endpoints  # 엔드포인트별

# 3. 엔드포인트 관리
c0 endpoint config <id>    # 설정
c0 endpoint get <id>       # 정보
c0 endpoint log <id>       # 로그
```


#### 3.8.7 로드밸런서 및 NAT 관리 🔄

##### A. 서비스 관리
| 구성 요소 | 명령어 | 설명 |
|----------|--------|------|
| 서비스 목록 | `c0 service list` | LB 서비스 조회 |
| BPF LB | `c0 bpf lb list` | BPF 로드밸런서 |
| NAT | `c0 bpf nat list` | NAT 매핑 조회 |
| 연결 추적 | `c0 bpf ct list` | 커넥션 트래킹 |

```bash
# 1. 서비스 상태
c0 service list
c1 service list

# 2. BPF 구성
c0 bpf lb list
c0 bpf lb list --revnat  # Reverse NAT

# 3. 네트워크 매핑
c0 bpf nat list    # NAT 항목
c0 bpf ct list global  # 연결 추적
```

> 💡 **관리 팁**
> - 정기적인 상태 점검
> - 연결 추적 테이블 관리
> - NAT 항목 모니터링
#### 3.8.8 시스템 모니터링 📊

##### A. 시스템 구성 요소
| 구성 요소 | 명령어 | 설명 |
|----------|--------|------|
| cgroups | `c0 cgroups list` | 리소스 제어 |
| BPF 맵 | `c0 map list` | 맵 관리 |
| 메트릭 | `c1 metrics list` | 성능 지표 |
| 정책 | `c0 bpf policy get` | 정책 상태 |

##### B. 모니터링 명령어
```bash
# 1. 리소스 관리
c0 cgroups list          # cgroup
c0 map list --verbose    # BPF 맵

# 2. 이벤트 모니터링
c1 map events cilium_lb4_services_v2
c1 map events cilium_ipcache

# 3. 시스템 상태
c1 metrics list          # 메트릭
c0 statedb dump         # 상태 DB
```

> 💡 **모니터링 팁**
> - 주요 메트릭 정기 확인
> - 이벤트 로그 분석
> - 시스템 상태 추적

## 4. Star Wars 데모 실습 🚀

### 4.1 데모 개요 📋

| 구성 요소 | 설명 | 목적 |
|----------|------|------|
| 애플리케이션 | Star Wars 테마 앱 | 정책 테스트 |
| 네트워크 정책 | Cilium CNP | 접근 제어 |
| 모니터링 | Hubble UI | 트래픽 관찰 |

### 4.2 환경 구성 🛠️

#### 4.2.1 애플리케이션 배포 📦

```bash
# 1. 데모 리소스 생성
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/http-sw-app.yaml

# 2. Pod 상태 확인
kubectl get pod --show-labels

# 3. 서비스 구성 확인
kubectl get deploy,svc,ep deathstar
```

##### A. 배포된 컴포넌트

| 컴포넌트 | 유형 | 레이블 |
|----------|------|--------|
| Deathstar | Deployment | org=empire,class=deathstar |
| Tiefighter | Pod | org=empire,class=tiefighter |
| Xwing | Pod | org=alliance,class=xwing |
```

##### B. 구성 특징

| 특징 | 설명 |
|------|------|
| 로드밸런싱 | Deathstar 2개 Pod |
| 조직 구분 | Empire vs Alliance |
| 기본 정책 | 전체 통신 허용 |

#### 4.2.2 Cilium 리소스 검증 🔍

| 검증 항목 | 확인 내용 | 방법 |
|----------|-----------|------|
| Endpoint | 리소스 상태 | `kubectl get ciliumendpoints` |
| Identity | 식별자 매핑 | `kubectl get ciliumidentities` |
| 정책 상태 | 기본 Disabled | `cilium endpoint list` |

```bash
# 1. Cilium 리소스 조회
kubectl get ciliumendpoints.cilium.io -A
kubectl get ciliumidentities.cilium.io

# 2. 노드별 상태 확인
c0 endpoint list
c1 endpoint list
c2 endpoint list
```

##### A. 엔드포인트 상태 예시

| 필드 | 값 | 설명 |
|------|-----|------|
| ENDPOINT | 1579 | 엔드포인트 ID |
| POLICY | Disabled | 정책 미적용 상태 |
| IDENTITY | 318 | 식별자 번호 |
| LABELS | k8s:org=empire | 조직 레이블 |

### 4.3 트래픽 모니터링 👀

#### 4.3.1 초기 상태 확인 📊

| 항목 | 상태 | 비고 |
|------|------|------|
| 통신 제어 | 전체 허용 | 정책 미적용 |
| 레이블 기반 | 제한 없음 | 모든 레이블 허용 |
| 모니터링 | 활성화 | 전체 트래픽 관찰 |

#### 4.3.2 Identity 설정 🏷️

```bash
# 1. Pod별 Identity 확인
c1 endpoint list | grep -iE 'xwing|tiefighter|deathstar'

# 2. Identity 환경변수 설정
XWINGID=17141
TIEFIGHTERID=56716
DEATHSTARID=8113
```

| Pod | Identity | 조직 |
|-----|----------|------|
| X-wing | 17141 | Alliance |
| TIE Fighter | 56716 | Empire |
| Deathstar | 8113 | Empire |

#### 4.3.3 트래픽 모니터링 🔍

##### A. 모니터링 설정
```bash
# 1. Cilium Agent 모니터링
c0 monitor -v -v    # 컨트롤 플레인
c1 monitor -v -v    # 워커 노드 1

# 2. Hubble 트래픽 관찰
hubble observe -f --from-identity $XWINGID          # X-wing
hubble observe -f --protocol tcp --from-identity $DEATHSTARID  # Deathstar
```

##### B. 모니터링 옵션

| 옵션 | 설명 | 예시 |
|------|------|------|
| -f | 실시간 관찰 | `observe -f` |
| --from-identity | 출발지 필터 | `$XWINGID` |
| --protocol | 프로토콜 필터 | `tcp` |

#### 4.3.4 접근 테스트 ⚡️

| 시나리오 | 설명 | 예상 결과 |
|----------|------|-----------|
| X-wing | Alliance 접근 | 거부 |
| TIE Fighter | Empire 접근 | 허용 |

```bash
# 1. X-wing 접근 테스트
kubectl exec xwing -- curl -s -XPOST \
  deathstar.default.svc.cluster.local/v1/request-landing

# 2. 연속 요청 테스트
while true; do
  kubectl exec xwing -- curl -s -XPOST \
    deathstar.default.svc.cluster.local/v1/request-landing
  sleep 5
done

# 3. TIE Fighter 테스트
kubectl exec tiefighter -- curl -s -XPOST \
  deathstar.default.svc.cluster.local/v1/request-landing

# 4. 트래픽 모니터링
hubble observe -f --protocol tcp --from-identity $TIEFIGHTERID
```

### 4.4 네트워크 정책 적용 🔒

#### 4.4.1 정책 개요

| 구성 요소 | 설명 | 예시 |
|----------|------|------|
| 제어 레벨 | L3/L4 정책 | TCP/80 |
| 필터링 기준 | 레이블 기반 | org=empire |
| 허용 범위 | 조직 내부 | Empire 통신만 |

#### 4.4.2 정책 정의 📝

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "empire-rule"
spec:
  description: "L3-L4 policy for Empire access control"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```

#### 4.4.3 정책 적용 ✨

```bash
# 1. 정책 적용
kubectl apply -f sw_l3_l4_policy.yaml

# 2. 정책 상태 확인
kubectl get cnp

# 3. 트래픽 검증
hubble observe -f --type policy-verdict
```

| 구성 요소 | 설명 | 값 |
|----------|------|-----|
| Selector | 대상 선택 | org=empire,class=deathstar |
| 허용 출발지 | Empire 소속 | org=empire |
| 허용 포트 | HTTP | TCP/80 |

⸻

## 5. 정리 및 결론 📝

### 5.1 학습 내용 요약 
- Cilium CNI 구성 및 관리
- Hubble을 통한 관측성 확보
- 네트워크 정책 실습

### 5.2 다음 단계
- 고급 네트워크 정책 구성
- 서비스 메시 통합
- 보안 모니터링 강화

> 💡 **참고 자료**
> - [Cilium 문서](https://docs.cilium.io)
> - [Hubble 가이드](https://docs.cilium.io/en/stable/gettingstarted/hubble/)
> - [정책 레퍼런스](https://docs.cilium.io/en/stable/policy/)
#### 4.4.4 정책 검증 및 모니터링 🔍

##### A. 정책 상태 확인
```bash
# 1. 정책 조회
kubectl get cnp
kubectl get cnp -o json | jq

# 2. 트래픽 모니터링
hubble observe -f --type drop
```

##### B. 접근 테스트

| 테스트 | 명령어 | 예상 결과 |
|--------|--------|-----------|
| X-wing 접근 | `kubectl exec xwing -- curl ...` | 연결 거부 |
| 패킷 확인 | `hubble observe -f --type drop` | 드롭 기록 |
| TCP 모니터링 | `hubble observe --protocol tcp` | 정책 적용 |

## 5. 참고 자료 및 결론 📚

### 5.1 주요 문서 📖

| 자료 | 설명 | URL |
|------|------|-----|
| 공식 문서 | Cilium 메인 문서 | [링크](https://docs.cilium.io) |
| Hubble | 관측성 가이드 | [링크](https://docs.cilium.io/en/stable/gettingstarted/hubble/) |
| 정책 | 네트워크 정책 | [링크](https://docs.cilium.io/en/stable/policy/) |

### 5.2 학습 자원 🎓

> 💡 **추가 학습 경로**
> - Cilium GitHub 저장소 탐색
> - 공식 블로그 글 학습
> - Hubble 실습 튜토리얼
> - eBPF 기술 문서 학습
> - Empire 소속 TIE Fighter 접근 허용
> - 정책 상태 Enabled 확인

```bash
# TIE Fighter 접근 테스트
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

# 정책 활성화 상태 확인
c0 endpoint list
c1 endpoint list
c2 endpoint list
```

> **엔드포인트 상태 예시**
```
ENDPOINT   POLICY (ingress)   POLICY (egress)   IDENTITY   LABELS
1579       Enabled           Disabled          318        k8s:app.kubernetes.io/name=deathstar
                                                         k8s:class=deathstar
                                                         k8s:org=empire

# 정책 상세 확인
kubectl describe cnp rule1

위 출력에서 Enabled로 표시된 것을 통해 deathstar 파드의 ingress에 정책이 정상 적용되었음을 확인할 수 있습니다.
kc describe cnp rule1 명령으로 정책의 endpointSelector, ingress.fromEndpoints, toPorts 등의 상세 설정을 검토하세요.

⸻

### 4.4 L7 정책 구성

> **L7 처리 구조**
> - Envoy 프록시 기반 HTTP/gRPC 정책 처리
> - 노드별 cilium-envoy DaemonSet 배포
> - BPF와 Envoy 간 통합 구성

#### 4.4.1 Envoy 컴포넌트 검증
```bash
# DaemonSet 상태
kubectl get ds -n kube-system
NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
cilium         3         3         3       3            3           kubernetes.io/os=linux
cilium-envoy   3         3         3       3            3           kubernetes.io/os=linux

# Envoy Pod 배포 상태
kubectl get pod -n kube-system -l k8s-app=cilium-envoy -owide

# 구성 정보 확인
kubectl describe ds -n kube-system cilium-envoy
```

> **주요 마운트 포인트**
```
Mounts:
  /sys/fs/bpf: BPF 맵 (rw)
  /var/run/cilium/envoy/: Envoy 설정 (ro)
  /var/run/cilium/envoy/artifacts: Envoy 아티팩트 (ro)
  /var/run/cilium/envoy/sockets: Envoy 소켓 (rw)
```

#### 4.4.2 Envoy 연결 검증

> **검증 항목**
> - Envoy-Agent 소켓 연결
> - XDS 설정 상태
> - 관리자 인터페이스

```bash
# 소켓 연결 상태
kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- ss -xnp | grep -i -envoy

# 연결된 소켓
/var/run/cilium/envoy/sockets/xds.sock     # XDS 설정
/var/run/cilium/envoy/sockets/admin.sock   # 관리자 인터페이스

# Envoy 설정 확인
kubectl describe cm -n kube-system cilium-envoy-config
```

> **주요 설정**
```json
{
  "admin": {
    "address": {
      "pipe": {
        "path": "/var/run/cilium/envoy/sockets/admin.sock"
      }
    }
  }
}
```

bootstrap-config.json에서 Envoy Bootstrap 설정(관리 소켓 경로 등)의 세부 내용을 검토합니다.

### 4.5 L7 정책 테스트

> **정책 목표**
> - HTTP 메서드와 경로 기반 제어
> - `/v1/request-landing`만 POST 허용
> - 기타 엔드포인트 차단

#### 4.5.1 기존 제한사항 확인

> **L3/L4 정책의 한계**
> - HTTP 경로 구분 불가
> - 메서드 기반 제어 불가
> - 상세 애플리케이션 로직 제어 불가

```bash
# L3/L4 모니터링
hubble observe -f --protocol tcp --from-identity $DEATHSTARID

# 취약점 테스트
kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
```


⸻

#### 4.5.2 L7 정책 정의

> **정책 세부사항**
> - Empire 조직 내부 통신 허용
> - POST 메서드만 허용
> - `/v1/request-landing` 경로만 허용

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L7 policy to restrict access to specific HTTP call"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "POST"
          path: "/v1/request-landing"
```

```bash
# 정책 적용
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/sw_l3_l4_l7_policy.yaml

# 정책 검증
kubectl get cnp
kubectl describe cnp
c0 policy get
```


⸻

#### 4.5.3 HTTP 트래픽 모니터링

> **모니터링 항목**
> - HTTP 요청/응답 흐름
> - 정책 적용 결과
> - 응답 시간 측정

```bash
# HTTP 트래픽 모니터링
hubble observe -f --pod deathstar --protocol http

# 출력 예시
# default/tiefighter:59020 -> default/deathstar:80
#   http-request FORWARDED POST /v1/request-landing
#   http-response FORWARDED 200 (6ms)

# 허용된 요청 테스트
kubectl exec tiefighter -- curl -s -XPOST \
  deathstar.default.svc.cluster.local/v1/request-landing
```


⸻

#### 4.5.4 차단 정책 검증

> **검증 포인트**
> - 차단된 요청 모니터링
> - 정책 적용 결과 확인
> - 상세 로그 분석

```bash
# 차단된 트래픽 모니터링
hubble observe -f --pod deathstar --verdict DROPPED

# 출력 예시
# default/tiefighter:44734 -> default/deathstar:80
#   http-request DROPPED PUT /v1/exhaust-port

# L7 상세 모니터링
c1 monitor -v --type l7

# 출력 예시
# Request: tiefighter -> deathstar
#   verdict: Denied
#   method: PUT
#   path: /v1/exhaust-port
#   status: 403
```

> **정책 동작 확인**
> - POST /v1/request-landing: 허용(200 OK)
> - PUT /v1/exhaust-port: 차단(403 Forbidden)

#### 4.5.5 종합 테스트

> **테스트 시나리오**
> - 차단된 엔드포인트 재시도
> - X-wing 트래픽 모니터링
> - 정책 적용 결과 검증

```bash
# 차단 정책 테스트
kubectl exec tiefighter -- curl -s -XPUT \
  deathstar.default.svc.cluster.local/v1/exhaust-port
# 결과: Access denied

# X-wing 트래픽 모니터링
hubble observe -f --pod xwing

# 다양한 시나리오 테스트 수행
# - Empire 소속 vs Alliance 소속
# - 허용된 경로 vs 차단된 경로
# - GET/POST/PUT 메서드 차이
```
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing --connect-timeout 2


⸻

### 4.6 환경 정리

> **정리 항목**
> - 데모 애플리케이션 제거
> - 네트워크 정책 삭제
> - 리소스 상태 확인

```bash
# 애플리케이션 삭제
kubectl delete -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/http-sw-app.yaml

# 정책 삭제
kubectl delete cnp rule1

# 상태 확인
kubectl get cnp
```


⸻

## 5. Hubble Exporter 설정

### 5.1 기본 구성

> **Exporter 기능**
> - 트래픽 흐름 로그 저장
> - 로그 회전 및 크기 관리
> - 필터 및 마스킹 지원

#### 5.1.1 Exporter 활성화
```bash
# Helm 설정 (이미 적용됨)
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
  --set hubble.enabled=true \
  --set hubble.export.static.enabled=true \
  --set hubble.export.static.filePath=/var/run/cilium/hubble/events.log

# 구성 적용 확인
kubectl -n kube-system rollout status ds/cilium
```

#### 5.1.2 설정 검증
```bash
# 설정 조회
kubectl get cm -n kube-system cilium-config -o json | grep hubble-export
cilium config view | grep hubble-export

# 주요 설정 항목
hubble-export-allowlist                # 허용 리스트
hubble-export-denylist                 # 차단 리스트
hubble-export-fieldmask               # 필드 마스킹
hubble-export-file-max-backups        # 보관 파일 수 (기본값: 5)
hubble-export-file-max-size-mb        # 로그 회전 크기 (기본값: 10MB)
hubble-export-file-path               # 로그 파일 경로

# 로그 저장 확인
kubectl -n kube-system exec ds/cilium -- tail -f /var/run/cilium/hubble/events.log
kubectl -n kube-system exec ds/cilium -- sh -c 'tail -f /var/run/cilium/hubble/events.log' | jq
```

### 5.2 성능 최적화

> **최적화 옵션**
> - 트래픽 필터링 설정
> - 데이터 마스킹 규칙
> - 저장소 관리 정책

#### 5.2.1 필터 설정
```yaml
hubble.export.static.allowList:  # 허용 리스트
  - type: "cilium"
    match: "l7-http"
    
hubble.export.static.denyList:   # 차단 리스트
  - type: "kubernetes"
    match: "health-check"
```

#### 5.2.2 데이터 마스킹
```yaml
hubble.export.static.fieldMask:  # 필드 마스킹
  - source.identity
  - destination.identity
  - source.namespace
  - destination.namespace
```

> Field mask는 [flow proto](https://github.com/cilium/cilium/blob/main/api/v1/flow/flow.proto) 정의의 필드 이름을 사용합니다.

---

## Static Exporter: Filters & Field Masks

> `hubble observe --print-raw-filters` 명령을 활용해 필요한 필터 조건을 생성할 수 있습니다.

```bash
# You can use hubble CLI to generated required filters (see Specifying Raw Flow Filters for more examples).
# For example, to filter flows with verdict DENIED or ERROR, run:
hubble observe --verdict DROPPED --verdict ERROR --print-raw-filters
allowlist:
- '{"verdict":["DROPPED","ERROR"]}'

위 예시는 verdict가 DROPPED 또는 ERROR인 플로우만 로깅하도록 하는 허용 리스트입니다.

# To keep all information except pod labels:
hubble-export-fieldmask: time source.identity source.namespace source.pod_name destination.identity destination.namespace destination.pod_name source_service destination_service l4 IP ethernet l7 Type node_name is_reply event_type verdict Summary

# To keep only timestamp, verdict, ports, IP addresses, node name, pod name, and namespace:
hubble-export-fieldmask: time source.namespace source.pod_name destination.namespace destination.pod_name l4 IP node_name is_reply verdict

첫 번째 필드 마스크는 pod 라벨을 제외한 모든 정보를,
두 번째는 최소한의 정보(타임스탬프, 포드/네임스페이스, 포트, IP, 노드명, 응답 여부, verdict)만 유지합니다.

⸻

Static Exporter 구성 방법

방법 1: ConfigMap 직접 패치

# 설정 방안 1 : Then paste the output to hubble-export-allowlist in cilium-config Config Map:
kubectl -n kube-system patch cm cilium-config --patch-file=/dev/stdin <<-EOF
data:
  hubble-export-allowlist: '{"verdict":["DROPPED","ERROR"]}'
  hubble-export-denylist: '{"source_pod":["kube-system/"]},{"destination_pod":["kube-system/"]}'
EOF

cilium-config ConfigMap에 직접 허용/차단 리스트를 추가합니다.

⸻

방법 2: Helm 업그레이드

# 설정 방안 2 : helm 업그레이드
helm upgrade cilium cilium/cilium --version 1.17.6 \
   --set hubble.enabled=true \
   --set hubble.export.static.enabled=true \
   --set hubble.export.static.filePath=/var/run/cilium/hubble/events.log \
   --set hubble.export.static.allowList[0]='{"verdict":["DROPPED","ERROR"]}' \
   --set hubble.export.static.denyList[0]='{"source_pod":["kube-system/"]}' \
   --set hubble.export.static.denyList[1]='{"destination_pod":["kube-system/"]}' \
   --set "hubble.export.static.fieldMask={time,source.namespace,source.pod_name,destination.namespace,destination.pod_name,l4,IP,node_name,is_reply,verdict,drop_reason_desc}"

Helm 차트를 통해 static exporter의 allowList, denyList, fieldMask를 한 번에 구성합니다.

# 확인
cilium config view | grep hubble-export
hubble-export-allowlist                           {"verdict":["DENIED","ERROR"]}
...

kubectl -n kube-system exec ds/cilium -- tail -f /var/run/cilium/hubble/events.log
{"flow":{"time":"2023-08-21T12:12:13.517394084Z","verdict":"DROPPED","IP":{"source":"fe80::64d8:8aff:fe72:fc14","destination":"ff02::2","ipVersion":"IPv6"},"l4":{"ICMPv6":{"type":133}},"source":{},"destination":{},"node_name":"kind-kind/kind-worker","drop_reason_desc":"INVALID_SOURCE_IP"},"node_name":"kind-kind/kind-worker","time":"2023-08-21T12:12:13.517394084Z"}
{"flow":{"time":"2023-08-21T12:12:18.510175415Z","verdict":"DROPPED","IP":{"source":"10.244.1.60","destination":"10.244.1.5","ipVersion":"IPv4"},"l4":{"TCP":{"source_port":44916,"destination_port":80,"flags":{"SYN":true}}},"source":{"namespace":"default","pod_name":"xwing"},"destination":{"namespace":"default","pod_name":"deathstar-7848d6c4d5-th9v2"},"node_name":"kind-kind/kind-worker","drop_reason_desc":"POLICY_DENIED"},"node_name":"kind-kind/kind-worker","time":"2023-08-21T12:12:18.510175415Z"}

설정이 반영된 후 Hubble flows 로그에 POLICY_DENIED 등의 이벤트만 기록되는 예시입니다.

⸻

### 5.3 Dynamic Exporter 구성 🔄

#### 5.3.1 Dynamic Exporter 특징 📊

| 기능 | 설명 | 장점 |
|------|------|------|
| 실시간 구성 | 동적 설정 변경 | 무중단 운영 |
| 다중 필터 | 복수 필터 지원 | 상세한 제어 |
| 로그 분리 | 개별 파일 저장 | 효율적 관리 |

#### 5.3.1 활성화 설정
```bash
# Exporter 전환
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
  --set hubble.enabled=true \
  --set hubble.export.static.enabled=false \
  --set hubble.export.dynamic.enabled=true

# 적용 확인
kubectl -n kube-system rollout status ds/cilium
```

#### 5.3.2 필터 구성
```bash
# 시스템 로그 필터 예시
helm upgrade cilium cilium/cilium --version 1.17.6 \
  --set hubble.enabled=true \
  --set hubble.export.dynamic.enabled=true \
  --set hubble.export.dynamic.config.content[0].name=system \
  --set hubble.export.dynamic.config.content[0].filePath=/var/run/cilium/hubble/events-system.log \
  --set hubble.export.dynamic.config.content[0].includeFilters[0].source_pod[0]='kube_system/' \
  --set hubble.export.dynamic.config.content[0].includeFilters[1].destination_pod[0]='kube_system/'
```

> **주요 기능**
> - 다중 필터 세트 관리
> - 실시간 구성 변경
> - Static Exporter와 동일한 기능 지원
> - Pod 재시작 없이 설정 적용
## 6. 메트릭 수집 테스트 📊

### 6.1 테스트 환경 구성 🔧

#### 6.1.1 구성 요소 개요

| 구성 요소 | 설정 | 목적 |
|----------|------|------|
| Webpod | 샘플 웹 앱 | 트래픽 생성 |
| 분산 배포 | Anti-affinity | 가용성 확보 |
| 서비스 | ClusterIP | 내부 접근 |

#### 6.1.2 웹 애플리케이션 배포
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webpod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webpod
  template:
    metadata:
      labels:
        app: webpod
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - sample-app
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: webpod
        image: traefik/whoami
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: webpod
  labels:
    app: webpod
spec:
  selector:
    app: webpod
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```


⸻

#### 6.1.3 테스트 클라이언트 배포

> **클라이언트 구성**
> - 용도: HTTP 요청 테스트
> - 위치: k8s-ctr 노드
> - 도구: netshoot 이미지

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-pod
  labels:
    app: curl
spec:
  nodeName: k8s-ctr
  containers:
  - name: curl
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
```


⸻

#### 6.1.4 배포 검증

> **검증 항목**
> - 리소스 상태 확인
> - 엔드포인트 등록 확인
> - 통신 테스트 수행

```bash
# 리소스 상태 확인
kubectl get deploy,svc,ep webpod -owide
kubectl get endpointslices -l app=webpod

# Cilium 엔드포인트 확인
kubectl get ciliumendpoints
kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- \
  cilium-dbg endpoint list

# 통신 테스트
kubectl exec -it curl-pod -- curl webpod | grep Hostname

# 연속 테스트
kubectl exec -it curl-pod -- sh -c \
  'while true; do curl -s webpod | grep Hostname; sleep 1; done'
```

> **검증 포인트**
> - Deployment/Service/Endpoint 상태
> - Cilium 엔드포인트 등록
> - 로드밸런싱 동작 확인
### 6.2 모니터링 도구 구성 📈

#### 6.2.1 구성 요소 개요

| 도구 | 용도 | 기능 |
|------|------|------|
| Prometheus | 메트릭 수집 | 시계열 데이터 저장 |
| Grafana | 시각화 | 대시보드 제공 |
| 대시보드 | 사전 구성 | Cilium/Hubble 통합 |

#### 6.2.2 도구 설치
```bash
# 모니터링 스택 배포
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/kubernetes/addons/prometheus/monitoring-example.yaml

# 리소스 확인
kubectl get deploy,pod,svc,ep -n cilium-monitoring

# ConfigMap 확인
kubectl get cm -n cilium-monitoring
```

> **생성된 ConfigMap**
```
grafana-cilium-dashboard           # Cilium 기본 대시보드
grafana-cilium-operator-dashboard  # Operator 대시보드
grafana-config                     # Grafana 기본 설정
grafana-hubble-dashboard          # Hubble 대시보드
grafana-hubble-l7-http-metrics    # HTTP 메트릭 대시보드
prometheus                        # Prometheus 설정
```

#### 6.2.2 설정 검증
```bash
# Prometheus 설정
kubectl describe cm -n cilium-monitoring prometheus

# Grafana 설정
kubectl describe cm -n cilium-monitoring grafana-config

# 대시보드 설정
kubectl describe cm -n cilium-monitoring grafana-cilium-dashboard
kubectl describe cm -n cilium-monitoring grafana-hubble-dashboard
```

위 명령으로 cilium-monitoring 네임스페이스에 Prometheus, Grafana, 그리고 관련 ConfigMap들이 생성 및 주입되었음을 확인합니다.

⸻

### 6.3 메트릭 수집 활성화

> **메트릭 구성**
> - Cilium 메트릭: 9962 포트
> - Operator 메트릭: 9963 포트
> - Hubble 메트릭: 9965 포트

#### 6.3.1 메트릭 설정
```bash
# 메트릭 활성화 (이미 설정됨)
helm install cilium cilium/cilium --version 1.17.6 \
  --namespace kube-system \
  --set prometheus.enabled=true \
  --set operator.prometheus.enabled=true \
  --set hubble.enabled=true \
  --set hubble.metrics.enableOpenMetrics=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}"
```

#### 6.3.2 포트 검증
```bash
# 컨트롤 플레인 포트 확인
ss -tnlp | grep -E '9962|9963|9965'

# 출력 예시
# 9963: Cilium-operator 메트릭
# 9962: Cilium-agent 메트릭
# 9965: Hubble 메트릭

# 워커 노드 포트 확인
for i in w1 w2 ; do
  echo ">> node : k8s-$i <<"
  sshpass -p 'vagrant' ssh vagrant@k8s-$i \
    sudo ss -tnlp | grep -E '9962|9963|9965'
done
```
  echo
done

위 확인을 통해 각 노드에서 Cilium 컴포넌트의 메트릭 포트가 정상적으로 열렸음을 검증합니다.

⸻

### 6.4 외부 접속 설정

#### 6.3.1 NodePort 구성

| 서비스 | 포트 | 접근 URL |
|--------|------|----------|
| Prometheus | 30001 | http://<노드_IP>:30001 |
| Grafana | 30002 | http://<노드_IP>:30002 |

```bash
# 1. 서비스 확인
kubectl get svc -n cilium-monitoring

# 2. NodePort 변경
kubectl patch svc -n cilium-monitoring prometheus \
  -p '{"spec": {"type": "NodePort", "ports": [{"port": 9090, "nodePort": 30001}]}}'
kubectl patch svc -n cilium-monitoring grafana \
  -p '{"spec": {"type": "NodePort", "ports": [{"port": 3000, "nodePort": 30002}]}}'

# NodePort 설정
kubectl patch svc -n cilium-monitoring prometheus -p '{"spec": {"type": "NodePort", "ports": [{"port": 9090, "targetPort": 9090, "nodePort": 30001}]}}'
kubectl patch svc -n cilium-monitoring grafana -p '{"spec": {"type": "NodePort", "ports": [{"port": 3000, "targetPort": 3000, "nodePort": 30002}]}}'

# 설정 확인
kubectl get svc -n cilium-monitoring
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
grafana      NodePort   10.96.212.137   <none>        3000:30002/TCP   14m
prometheus   NodePort   10.96.240.147   <none>        9090:30001/TCP   14m

# 접속 주소 예시 (노드 eth1 IP 사용)s
echo "http://192.168.10.100:30001"  # Prometheus
echo "http://192.168.10.100:30002"  # Grafana

#### 6.3.2 접속 검증 ✅

```bash
# 접속 URL 확인
NODEIP=$(ip -4 addr show eth1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
echo "Prometheus: http://$NODEIP:30001"
echo "Grafana: http://$NODEIP:30002"
```

