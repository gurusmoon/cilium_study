

# Cilium Study - Week 2

## 목차

1. [실습 환경 구성](#1-실습-환경-구성)
   - 실습 환경 배포
   - 클러스터 점검
   - 네트워크 설정 검증

2. [Cilium 관측 도구 구성](#2-cilium-관측-도구-구성)
   - Hubble 설치 및 설정
   - Prometheus & Grafana 구성
   - 메트릭스 수집 설정

3. [네트워크 정책 실습](#3-네트워크-정책-실습)
   - L3/L4 정책 적용
   - L7 정책 구성
   - 트래픽 모니터링

## 1. 실습 환경 소개

> **실습 환경 구성 요약**
> - Vagrant로 구성하는 3노드 쿠버네티스 클러스터
> - Cilium CNI가 설치된 완전한 네트워킹 환경
> - 네트워크 정책 및 모니터링 실습을 위한 기반 환경

### 1.1 클러스터 구성
- **컨트롤 플레인**
  - 노드명: `k8s-ctr`
  - 역할: 클러스터 관리 및 제어
  
- **워커 노드**
  - `k8s-w1`, `k8s-w2`
  - 역할: 워크로드 실행 및 네트워크 정책 적용

### 1.2 주요 컴포넌트
- **Kubernetes**: v1.33.2-1.1
- **Container Runtime**: containerd v1.7.27-1
- **CNI**: Cilium (kube-proxy 대체 모드)

---

## 2. 실습 환경 배포

### 2.1 배포 파일 구성

> **배포 자동화 개요**
> - Vagrant를 사용한 멀티노드 클러스터 프로비저닝
> - 쉘 스크립트를 통한 컴포넌트 자동 설치
> - Cilium CNI 및 관련 도구 설정 자동화

#### 주요 파일 구성
1. **Vagrantfile**
   - 가상 머신 스펙 정의
   - 네트워크 인터페이스 구성
   - 초기화 스크립트 연결

2. **초기화 스크립트**
   - `init_cfg.sh`: 기본 환경 구성
   - `k8s-ctr.sh`: 컨트롤 플레인 초기화
   - `k8s-w.sh`: 워커 노드 조인

### 2.2 클러스터 배포

> **배포 절차**
> 1. 실습 디렉토리 생성
> 2. Vagrant 설정 파일 다운로드
> 3. 가상 머신 프로비저닝 실행

```bash
# 실습 환경 구성
mkdir cilium-lab && cd cilium-lab
curl -O https://raw.githubusercontent.com/gasida/vagrant-lab/refs/heads/main/cilium-study/2w/Vagrantfile
vagrant up


⸻

### 2.3 클러스터 초기 점검

> **검증 항목**
> - 호스트 네임 해석 설정
> - 노드 간 SSH 연결성
> - 네트워크 인터페이스 상태
> - 쿠버네티스 클러스터 구성
> - 시스템 네트워크 설정

#### 2.3.1 기본 연결성 확인
```bash
# 호스트 파일 설정 확인
cat /etc/hosts

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

#### 2.3.4 클러스터 구성 검증
```bash
# 클러스터 기본 정보
kubectl cluster-info

# 네트워크 CIDR 설정
kubectl cluster-info dump | grep -m 2 -E "cluster-cidr|service-cluster-ip-range"

# kubeadm 설정 확인
kubectl describe cm -n kube-system kubeadm-config
kubectl describe cm -n kube-system kubelet-config

# 5. 노드 정보 조회 (상태, INTERNAL-IP)
kubectl get node -o wide

# 6. kubelet 인자 확인
cat /var/lib/kubelet/kubeadm-flags.env
for i in w1 w2 ; do
  echo ">> node : k8s-$i <<"
  sshpass -p 'vagrant' ssh vagrant@k8s-$i cat /var/lib/kubelet/kubeadm-flags.env
  echo
done

# 7. 파드 CIDR 및 IP 확인
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
kubectl get ciliumnode -o json | grep podCIDRs -A2
kubectl get pod -A -o wide

# 8. iptables 설정 확인
iptables-save
iptables -t nat -S
iptables -t filter -S
iptables -t mangle -S


⸻

k8s-ctr – Cilium 설치 정보 확인

Cilium 바이너리 설치 여부, 설정(ConfigMap), 상태, 메트릭, 엔드포인트 등을 상세히 점검합니다.

# 1. cilium 바이너리 및 기본 상태 확인
which cilium
cilium status
cilium config view
kubectl get cm -n kube-system cilium-config -o json | jq

# 2. 상세 상태 및 메트릭
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg config
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg status --verbose
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg metrics list

# 3. 엔드포인트 정보
kubectl get ciliumendpoints -A

# 4. 트래픽 모니터링
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg monitor
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg monitor -v
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg monitor -v -v

# 5. 엔드포인트별 상세 모니터링 및 드롭 확인
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg monitor --related-to=<id>
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg monitor --type drop
kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg monitor -v -v --hex


⸻

Layer 7 모니터링

L7(HTTP 등 애플리케이션 레벨) 트래픽을 실시간으로 확인하여 정책 적용 결과를 점검합니다.

kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg monitor -v --type l7

## 3. Cilium 관측 도구 구성

### 3.1 Hubble 설치 및 설정

> **Hubble 개요**
> - Cilium의 네트워크 관측(Observability) 플랫폼
> - 실시간 네트워크 트래픽 모니터링
> - 트래픽 플로우 시각화 및 분석
> - 보안 정책 검증 및 디버깅

#### 3.1.1 설치 전 환경 점검

> **필수 확인 사항**
> - Cilium 구성 상태 확인
> - Hubble 관련 인증서 및 Secret 점검
> - 시스템 포트 상태 기록

```bash
```bash
# Cilium 상태 점검
cilium status
cilium config view | grep -i hubble
kubectl get cm -n kube-system cilium-config -o json | jq

# 인증서 및 Secret 확인
kubectl get secret -n kube-system | grep -iE 'cilium-ca|hubble'

# 포트 상태 기록 (설치 전)
ss -tnlp | grep -iE 'cilium|hubble' | tee before.txt


⸻

#### 3.1.2 Hubble 설치 옵션

> **설치 방식**
> - Helm 차트: 전체 기능 설치
> - Cilium CLI: 기본 기능 빠른 설치

##### A. Helm 차트 설치
```bash
# Hubble 전체 기능 설치
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.ui.service.type=NodePort \
  --set hubble.ui.service.nodePort=31234 \
  --set hubble.export.static.enabled=true \
  --set hubble.export.static.filePath=/var/run/cilium/hubble/events.log \
  --set prometheus.enabled=true \
  --set operator.prometheus.enabled=true \
  --set hubble.metrics.enableOpenMetrics=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}"
```

##### B. Cilium CLI 설치
```bash
# 기본 기능만 활성화
cilium hubble enable

# UI 포함 활성화
cilium hubble enable --ui
```


⸻

⚙️ 설치 후 검증

설치가 완료된 후, Relay 상태와 ConfigMap, Secret, 포트 열림 상태 등을 다시 확인합니다.

# Relay 상태 확인
cilium status
# → “Hubble Relay: OK” 메시지 확인

# Hubble 설정 반영 확인
cilium config view | grep -i hubble
kubectl get cm -n kube-system cilium-config -o json | grep -i hubble

# Secret 확인
kubectl get secret -n kube-system | grep -iE 'cilium-ca|hubble'

# 포트(4244) 열림 여부 확인 (설치 후)
ss -tnlp | grep -iE 'cilium|hubble' | tee after.txt
vi -d before.txt after.txt

# 각 워커 노드에서도 4244 포트 확인
for i in w1 w2 ; do
  echo ">> node : k8s-$i <<"
  sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo ss -tnlp | grep 4244
  echo
done


⸻

#### 3.1.4 Relay 구성 검증

> **Relay 서비스**
> - 각 노드의 Hubble 데이터를 중앙 집계
> - 포트 4245로 gRPC 서비스 제공
> - 클러스터 전체 네트워크 흐름 수집

```bash
# Relay Pod 상태
kubectl get pod -n kube-system -l k8s-app=hubble-relay
kubectl describe pod -n kube-system -l k8s-app=hubble-relay

# 서비스 및 Endpoint 확인
kubectl get svc,ep -n kube-system hubble-relay
# NAME                     ENDPOINTS           AGE
# endpoints/hubble-relay   172.20.1.202:4245   7m54s

# Relay 설정 검증
kubectl describe cm -n kube-system hubble-relay-config

# 주요 설정 예시
cluster-name: default
peer-service: "hubble-peer.kube-system.svc.cluster.local.:443"
listen-address: :4245
```


⸻

#### 3.1.5 Peer 서비스 검증

> **Peer 서비스**
> - 각 노드별 Hubble API 제공
> - 포트 4244로 노드 데이터 수집
> - Relay 서비스와 연동

```bash
# Peer 서비스 상태
kubectl get svc,ep -n kube-system hubble-peer
# NAME                  TYPE        CLUSTER-IP    PORT(S)   AGE
# service/hubble-peer   ClusterIP   10.96.145.0   443/TCP   5h55m

# Endpoint 검증
# 노드별 4244 포트 연결:
# 192.168.10.100:4244,192.168.10.101:4244,192.168.10.102:4244
```


⸻

#### 3.1.6 UI 서비스 검증

> **Hubble UI**
> - 네트워크 흐름 시각화 도구
> - NGINX 기반 웹 인터페이스
> - NodePort로 외부 접근 제공

```bash
# UI 컴포넌트 검증
kubectl describe pod -n kube-system -l k8s-app=hubble-ui
kubectl describe cm -n kube-system hubble-ui-nginx

# 서비스 상태 확인
kubectl get svc,ep -n kube-system hubble-ui
# NAME                TYPE       CLUSTER-IP    PORT(S)        AGE
# service/hubble-ui   NodePort   10.96.66.67   80:31234/TCP   17m
# endpoints/hubble-ui   172.20.2.70:8081   17m
```

#### 3.1.7 UI 접속 설정

> **접속 방법**
> 1. 노드 IP 확인 (eth1 인터페이스)
> 2. NodePort 서비스로 접속 (포트 31234)

```bash
# UI 접속 주소 확인
NODEIP=$(ip -4 addr show eth1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
echo -e "Hubble UI 주소 → http://$NODEIP:31234"
```

### 3.2 Hubble Client 구성

> **Client 설정 개요**
> - 로컬 시스템에서 Hubble API 접근
> - 포트 포워딩을 통한 Relay 연결
> - CLI 도구를 통한 모니터링

#### 3.2.1 Client 연결 설정

```bash
# Relay 포트 포워딩
cilium hubble port-forward&
# Hubble Relay is available at 127.0.0.1:4245

# 포트 상태 확인
ss -tnlp | grep 4245

# API 연결 검증
hubble status
# Healthcheck (via localhost:4245): Ok
# Current/Max Flows: 12,285/12,285 (100.00%)
# Flows/s: 41.20
```

#### 3.2.2 Client 설정 관리

```bash
# 서버 설정 확인
hubble config view 
# port-forward-port: "4245"
# server: localhost:4245

# 서버 옵션 확인
hubble help status | grep 'server string'
# --server string    Address of a Hubble server

# 트래픽 흐름 모니터링
kubectl get ciliumendpoints.cilium.io -n kube-system 
hubble observe
hubble observe -f
```


⸻

### 3.3 운영 편의성 설정

> **Cilium Agent 접근 단축키**
> - 노드별 Pod 접근 alias 설정
> - Cilium 명령어 실행 단축키
> - BPF 도구 접근 단축키

```bash
# Pod 환경변수 설정
export CILIUMPOD0=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-ctr -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD1=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w1  -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD2=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w2  -o jsonpath='{.items[0].metadata.name}')
echo $CILIUMPOD0 $CILIUMPOD1 $CILIUMPOD2

# Cilium 명령어 단축키
alias c0="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- cilium"
alias c1="kubectl exec -it $CILIUMPOD1 -n kube-system -c cilium-agent -- cilium"
alias c2="kubectl exec -it $CILIUMPOD2 -n kube-system -c cilium-agent -- cilium"

# BPF 도구 단축키
alias c0bpf="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- bpftool"
alias c1bpf="kubectl exec -it $CILIUMPOD1 -n kube-system -c cilium-agent -- bpftool"
alias c2bpf="kubectl exec -it $CILIUMPOD2 -n kube-system -c cilium-agent -- bpftool"

# 단축키 사용 예시
c0 endpoint list
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


#### 3.4.3 IP 및 Identity 관리
```bash
# IP 주소 관리
c0 ip list
c0 ip list -n    # IDENTITY 정보 포함

# Identity 관리
c0 identity list           # 전체 identity 조회
c0 identity list --endpoints  # 엔드포인트별 ID

# 엔드포인트 관리
c0 endpoint config <id>    # 설정 확인/변경
c0 endpoint get <id>       # 상세 정보
c0 endpoint log <id>       # 로그 확인

# BPF 파일시스템
c0 bpf fs show            # 마운트 정보
tree /sys/fs/bpf          # 디렉토리 구조
```


#### 3.4.4 로드밸런서 및 NAT 관리
```bash
# 로드밸런서 서비스
c0 service list
c1 service list
c2 service list

# BPF 로드밸런서
c0 bpf lb list
c1 bpf lb list --revnat    # Reverse NAT 항목

# 커넥션 트래킹
c0 bpf ct list global      # 전체 연결 조회
c0 bpf ct flush           # 연결 정보 초기화

# NAT 매핑
c0 bpf nat list           # NAT 항목 조회
c0 bpf nat flush          # NAT 항목 초기화

# IP 캐시
c0 bpf ipcache list       # IP-Identity 매핑
```#### 3.4.5 시스템 모니터링
```bash
# cgroup 관리
c0 cgroups list          # cgroup 메타데이터

# BPF 맵 관리
c0 map list              # 전체 맵 목록
c1 map list --verbose    # 상세 정보

# 맵 이벤트 모니터링
c1 map events cilium_lb4_services_v2
c1 map events cilium_lb4_reverse_nat
c1 map events cilium_lxc
c1 map events cilium_ipcache

# 메트릭스 및 정책
c1 metrics list          # 전체 메트릭스
c0 bpf policy get --all  # 정책 맵 조회

# 시스템 상태
c0 statedb dump         # StateDB 내용
c0 shell -- db/show devices  # 디바이스 정보
```>)

## 4. Star Wars 데모 실습

> **실습 개요**
> - 스타워즈 테마 데모 애플리케이션 배포
> - Cilium 네트워크 정책 적용
> - Hubble UI로 트래픽 관찰

### 4.1 데모 환경 구성

#### 4.1.1 애플리케이션 배포
```bash
# 데모 리소스 생성
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/http-sw-app.yaml

# 리소스 상태 확인
kubectl get pod --show-labels
NAME                         READY   STATUS    RESTARTS   AGE    LABELS
deathstar-8c4c77fb7-9klws   1/1     Running   0          29s    app.kubernetes.io/name=deathstar,class=deathstar,org=empire
deathstar-8c4c77fb7-kkwds   1/1     Running   0          29s    app.kubernetes.io/name=deathstar,class=deathstar,org=empire
tiefighter                  1/1     Running   0          29s    app.kubernetes.io/name=tiefighter,class=tiefighter,org=empire
xwing                       1/1     Running   0          29s    app.kubernetes.io/name=xwing,class=xwing,org=alliance

# 서비스 구성 확인
kubectl get deploy,svc,ep deathstar
```

> **배포된 컴포넌트**
> - Deathstar: 2개의 복제본을 가진 deployment
> - Tiefighter: empire 조직 소속 pod
> - Xwing: alliance 조직 소속 pod

위 명령을 통해 Deathstar 서비스가 2개의 파드로 로드밸런싱되며, Tiefighter/Xwing 파드가 생성됩니다.

⸻

#### 4.1.2 Cilium 리소스 검증

> **검증 항목**
> - Endpoint와 Identity 리소스 상태
> - 정책 설정 상태 (기본: Disabled)
> - 레이블 기반 식별자 확인

```bash
# Cilium 리소스 조회
kubectl get ciliumendpoints.cilium.io -A
kubectl get ciliumidentities.cilium.io

# 엔드포인트 상태 확인
kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium endpoint list

# 노드별 상태 확인
c0 endpoint list
c1 endpoint list
c2 endpoint list    # 정책 미적용 상태
```

> **엔드포인트 예시**
```
ENDPOINT   POLICY (ingress)   POLICY (egress)   IDENTITY   LABELS
1579       Disabled          Disabled          318        k8s:app.kubernetes.io/name=deathstar
                                                         k8s:class=deathstar
                                                         k8s:org=empire
```

### 4.2 트래픽 모니터링

> **초기 상태**
> - 모든 Pod 간 통신 허용
> - 레이블 기반 제한 없음
> - 전체 트래픽 모니터링 가능

#### 4.2.1 Identity 설정
```bash
# Pod별 Identity 확인
c1 endpoint list | grep -iE 'xwing|tiefighter|deathstar'
c2 endpoint list | grep -iE 'xwing|tiefighter|deathstar'

# Identity 환경변수 설정
XWINGID=17141
TIEFIGHTERID=56716
DEATHSTARID=8113
```

#### 4.2.2 모니터링 설정
```bash
# Cilium Agent 모니터링
c0 monitor -v -v    # 컨트롤 플레인
c1 monitor -v -v    # 워커 노드 1
c2 monitor -v -v    # 워커 노드 2

# Hubble 트래픽 관찰
hubble observe -f   # 전체 트래픽
hubble observe -f --from-identity $XWINGID          # X-wing 트래픽
hubble observe -f --protocol tcp --from-identity $DEATHSTARID  # Deathstar TCP
```

#### 4.2.3 접근 테스트

> **테스트 시나리오**
> - X-wing에서 Deathstar 접근
> - TIE Fighter에서 Deathstar 접근
> - 트래픽 플로우 모니터링

```bash
# X-wing 접근 테스트
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

# 연속 요청 테스트
while true; do
  kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
  sleep 5
done

# TIE Fighter 접근 테스트
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

# 연속 요청 테스트
while true; do
  kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
  sleep 5
done

# 트래픽 모니터링
hubble observe -f --protocol tcp --from-identity $TIEFIGHTERID
hubble observe -f --protocol tcp --from-identity $DEATHSTARID
```


⸻

### 4.3 네트워크 정책 적용

> **정책 개요**
> - L3/L4 레벨 접근 제어
> - 레이블 기반 트래픽 필터링
> - Empire 조직 내부 통신만 허용

#### 4.3.1 정책 정의
```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L3-L4 policy to restrict deathstar access to empire ships only"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
```
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP

# 정책 적용 및 확인
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/sw_l3_l4_policy.yaml
kubectl get cnp
kubectl get cnp -o json | jq

# 드롭된 패킷 모니터링
hubble observe -f --type drop

정책 적용 후, 다시 xwing(org=alliance)에서 요청을 시도하면 연결조차 되지 않아야 합니다.

# 호출 시도 1: Xwing에서 요청 (타임아웃 또는 연결 실패)
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing --connect-timeout 2

# 모니터링: Deathstar → Xwing 간 TCP 트래픽
hubble observe -f --protocol tcp --from-identity $DEATHSTARID

#### 4.3.2 정책 검증

> **정책 적용 결과**
> - Alliance 소속 X-wing 접근 차단
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

### 5.3 Dynamic Exporter 구성

> **Dynamic Exporter 특징**
> - 실시간 구성 변경 지원
> - 다중 필터 설정 가능
> - 별도 파일로 로그 분리

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
## 6. 메트릭 수집 테스트

### 6.1 테스트 애플리케이션 구성

> **구성 요소**
> - Webpod: 샘플 웹 애플리케이션
> - 분산 배포: Anti-affinity 설정
> - 서비스 노출: ClusterIP

#### 6.1.1 웹 애플리케이션 배포
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

#### 6.1.2 테스트 클라이언트 배포

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

#### 6.1.3 배포 검증

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
### 6.2 모니터링 도구 구성

> **구성 요소**
> - Prometheus: 메트릭 수집 및 저장
> - Grafana: 시각화 대시보드
> - 사전 구성: Cilium/Hubble 대시보드

#### 6.2.1 도구 설치
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

5. NodePort를 통한 외부 접속

호스트 PC에서 Prometheus 및 Grafana UI에 접근할 수 있도록 NodePort로 Service 타입을 변경합니다.

#
kubectl get svc -n cilium-monitoring
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
grafana      ClusterIP   10.96.212.137   <none>        3000/TCP   6m36s
prometheus   ClusterIP   10.96.240.147   <none>        9090/TCP   6m36s

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

위 과정을 통해 브라우저에서 http://<노드_IP>:30001 및 http://<노드_IP>:30002 로 Prometheus와 Grafana UI에 접근할 수 있습니다.