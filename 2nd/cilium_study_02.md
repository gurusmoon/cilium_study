

아래와 같이 원본 구성(Config)은 그대로 유지하면서, 각 섹션별로 설명을 추가해 보기 좋게 재정리했습니다.

## 2주차 실습 환경 개요

> 2주차 실습에서는 Vagrant 기반의 3노드(k8s-ctr, k8s-w1, k8s-w2) 클러스터를 구성하고, Cilium CNI를 설치한 뒤 초기 상태를 점검합니다.

- **가상 머신 구성**  
  - 컨트롤러: `k8s-ctr`  
  - 워커 1: `k8s-w1`  
  - 워커 2: `k8s-w2`

- **초기 프로비저닝**  
  - `kubeadm init` (컨트롤러 노드)  
  - `kubeadm join` (워커 노드)

- **Cilium CNI**  
  - CNI 플러그인으로 Cilium 설치 상태로 배포 완료

---

## 실습 환경 배포 파일 작성

> Vagrant와 스크립트를 이용해 클러스터를 자동으로 프로비저닝합니다.

- **Vagrantfile**  
  - 가상머신 정의  
  - 부팅 시 초기 프로비저닝(`init_cfg.sh`, `k8s-ctr.sh`, `k8s-w.sh`) 설정

- **init_cfg.sh**  
  - 인자(args)에 따라 Kubernetes 및 Cilium 설치

- **k8s-ctr.sh**  
  - `kubeadm init` 실행  
  - Cilium CNI 설치  
  - 편의성 설정(alias `k`, `kc` 등)

- **k8s-w.sh**  
  - `kubeadm join` 실행

```bash
mkdir cilium-lab && cd cilium-lab
curl -O https://raw.githubusercontent.com/gasida/vagrant-lab/refs/heads/main/cilium-study/2w/Vagrantfile
vagrant up


⸻

실습 환경 배포 및 초기 점검

클러스터가 정상적으로 생성되었는지 네트워크, 노드, 파드, iptables 상태 등을 종합적으로 확인합니다.

# 1. /etc/hosts 확인
cat /etc/hosts

# 2. 워커 노드 접속 테스트
sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-w1 hostname
sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-w2 hostname

# 3. 네트워크 인터페이스 확인
ifconfig | grep -iEA1 'eth[0-9]:'

# 4. 클러스터 정보 확인
kubectl cluster-info
kubectl cluster-info dump | grep -m 2 -E "cluster-cidr|service-cluster-ip-range"
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

## 1. Network Observability with Hubble

### Hubble 소개
Cilium이 제공하는 네트워크 관측(Observability) 플랫폼으로,  
- **네트워크 트래픽 가시성** 제공  
- **실시간 플로우 모니터링** 기능 제공  

---

### 📋 설치 전 확인
설치에 앞서 기존 Cilium 및 Hubble 관련 리소스와 포트 상태를 점검합니다.

```bash
# 1) Cilium 상태 및 설정 확인
cilium status
cilium config view | grep -i hubble
kubectl get cm -n kube-system cilium-config -o json | jq

# 2) 기존 Secret 확인
kubectl get secret -n kube-system | grep -iE 'cilium-ca|hubble'

# 3) 포트(4244 등) 열림 여부 확인 (설치 전)
ss -tnlp | grep -iE 'cilium|hubble' | tee before.txt


⸻

🚀 Hubble 설치

설치방안 1: Helm 차트로 완전 설치

Helm을 통해 Hubble, Relay, UI, Prometheus exporter 등을 한 번에 구성합니다.

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

설치방안 2: cilium CLI로 간단 활성화

Hubble만 간단히 활성화하거나, UI까지 포함해서 활성화할 수 있습니다.

# Hubble core 활성화
cilium hubble enable

# Hubble UI(웹 대시보드)까지 활성화
cilium hubble enable --ui


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

🔗 Hubble Relay 구성 확인

Relay는 각 노드의 Hubble 데이터를 집계해주는 서비스입니다.

# Relay Pod 확인
kubectl get pod -n kube-system -l k8s-app=hubble-relay
kc describe pod -n kube-system -l k8s-app=hubble-relay

# Relay 서비스 및 Endpoint 확인
kc get svc,ep -n kube-system hubble-relay
# NAME                     ENDPOINTS           AGE
# endpoints/hubble-relay   172.20.1.202:4245   7m54s

# Relay 설정(ConfigMap) 확인
kubectl describe cm -n kube-system hubble-relay-config
# → cluster-name, peer-service, listen-address 등 확인

# 예시: hubble-relay-config 일부
cluster-name: default
peer-service: "hubble-peer.kube-system.svc.cluster.local.:443"
listen-address: :4245


⸻

🔗 Hubble Peer 구성 확인

Peer는 각 노드에서 Hubble API를 제공합니다.

# Peer 서비스 및 Endpoint 확인
kubectl get svc,ep -n kube-system hubble-peer
# NAME                  TYPE        CLUSTER-IP    PORT(S)   AGE
# service/hubble-peer   ClusterIP   10.96.145.0   443/TCP   5h55m

# Endpoint 목록 (노드별 4244 포트)
# 192.168.10.100:4244,192.168.10.101:4244,192.168.10.102:4244


⸻

🖥️ Hubble UI(Web) 구성 확인

UI는 NGINX를 통해 Relay에 연결된 데이터를 시각화합니다.

# UI Pod 및 Container 확인
kc describe pod -n kube-system -l k8s-app=hubble-ui

# UI NGINX 설정 확인
kc describe cm -n kube-system hubble-ui-nginx

# UI 서비스 및 Endpoint 확인
kubectl get svc,ep -n kube-system hubble-ui
# NAME                TYPE       CLUSTER-IP    PORT(S)        AGE
# service/hubble-ui   NodePort   10.96.66.67   80:31234/TCP   17m
# endpoints/hubble-ui   172.20.2.70:8081   17m


⸻

🌐 Hubble UI 접속 방법
	1.	각 노드의 eth1 인터페이스 IP를 확인합니다.
	2.	브라우저에서 http://<NODE_IP>:31234 로 접속합니다.

NODEIP=$(ip -4 addr show eth1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
echo -e "Hubble UI 주소 → http://$NODEIP:31234"

## Hubble Client 설치 및 API 접근 검증

> 로컬 머신에서 Hubble Relay 서비스에 접근하기 위한 포트 포워딩 및 CLI 사용 방법

```bash
  #
  cilium hubble port-forward&
  Hubble Relay is available at 127.0.0.1:4245
  
  ss -tnlp | grep 4245
  
  # Now you can validate that you can access the Hubble API via the installed CLI
  hubble status
  Healthcheck (via localhost:4245): Ok
  Current/Max Flows: 12,285/12,285 (100.00%)
  Flows/s: 41.20
  
  # hubble (api) server 기본 접속 주소 확인
  hubble config view 
  ...
  port-forward-port: "4245"
  server: localhost:4245
  ...
  
  # (옵션) 현재 k8s-ctr 가상머신이 아닌, 자신의 PC에 kubeconfig 설정 후 아래 --server 옵션을 통해 hubble api server 사용해보자!
  hubble help status | grep 'server string'
        --server string                 Address of a Hubble server. Ignored when --input-file or --port-forward is provided. (default "localhost:4245")
  
  # You can also query the flow API and look for flows
  kubectl get ciliumendpoints.cilium.io -n kube-system # SECURITY IDENTITY
  hubble observe
  hubble observe -h
  hubble observe -f


⸻

Cilium Agent 단축키 지정

각 노드의 Cilium Agent Pod에 빠르게 접속하기 위한 alias 설정

[- Cilium Agent 단축키 지정ㄹㅇㄴㅁㄹㅇㄴㅁㄹㅇㄴㅁㄹ](<# cilium 파드 이름
export CILIUMPOD0=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-ctr -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD1=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w1  -o jsonpath='{.items[0].metadata.name}')
export CILIUMPOD2=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w2  -o jsonpath='{.items[0].metadata.name}')
echo $CILIUMPOD0 $CILIUMPOD1 $CILIUMPOD2

# 단축키(alias) 지정
alias c0="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- cilium"
alias c1="kubectl exec -it $CILIUMPOD1 -n kube-system -c cilium-agent -- cilium"
alias c2="kubectl exec -it $CILIUMPOD2 -n kube-system -c cilium-agent -- cilium"

alias c0bpf="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- bpftool"
alias c1bpf="kubectl exec -it $CILIUMPOD1 -n kube-system -c cilium-agent -- bpftool"
alias c2bpf="kubectl exec -it $CILIUMPOD2 -n kube-system -c cilium-agent -- bpftool"


# endpoint
c0 endpoint list
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


# Manage IP addresses and associated information - IP List
c0 ip list

# IDENTITY :  1(host), 2(world), 4(health), 6(remote), 파드마다 개별 ID
c0 ip list -n

# Retrieve information about an identity
c0 identity list

# 엔드포인트 기준 ID
c0 identity list --endpoints

# 엔드포인트 설정 확인 및 변경
c0 endpoint config <엔트포인트ID>

# 엔드포인트 상세 정보 확인
c0 endpoint get <엔트포인트ID>

# 엔드포인트 로그 확인
c0 endpoint log <엔트포인트ID>

# Show bpf filesystem mount details
c0 bpf fs show

# bfp 마운트 폴더 확인
tree /sys/fs/bpf


# Get list of loadbalancer services
c0 service list
c1 service list
c2 service list

## Or you can get the loadbalancer information using bpf list
c0 bpf lb list
c1 bpf lb list
c2 bpf lb list

## List reverse NAT entries
c1 bpf lb list --revnat
c2 bpf lb list --revnat


# List connection tracking entries
c0 bpf ct list global
c1 bpf ct list global
c2 bpf ct list global

# Flush connection tracking entries
c0 bpf ct flush
c1 bpf ct flush
c2 bpf ct flush


# List all NAT mapping entries
c0 bpf nat list
c1 bpf nat list
c2 bpf nat list

# Flush all NAT mapping entries
c0 bpf nat flush
c1 bpf nat flush
c2 bpf nat flush

# Manage the IPCache mappings for IP/CIDR <-> Identity
c0 bpf ipcache list# Display cgroup metadata maintained by Cilium
c0 cgroups list
c1 cgroups list
c2 cgroups list


# List all open BPF maps
c0 map list
c1 map list --verbose
c2 map list --verbose

c1 map events cilium_lb4_services_v2
c1 map events cilium_lb4_reverse_nat
c1 map events cilium_lxc
c1 map events cilium_ipcache


# List all metrics
c1 metrics list


# List contents of a policy BPF map : Dump all policy maps
c0 bpf policy get --all
c1 bpf policy get --all -n
c2 bpf policy get --all -n


# Dump StateDB contents as JSON
c0 statedb dump


#
c0 shell -- db/show devices
c1 shell -- db/show devices
c2 shell -- db/show devices>)

## Getting Started with the Star Wars Demo & Hubble/UI

> 스타워즈 데모 애플리케이션을 배포하고, Cilium/Hubble UI로 네트워크 플로우를 관찰하는 단계입니다.

---

### 1. 데모 애플리케이션 배포

아래 명령어를 그대로 실행하여 Deathstar, Tiefighter, Xwing 리소스를 생성합니다.  
(변경 없이 원본 그대로 유지)

```bash
#
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/http-sw-app.yaml
service/deathstar created
deployment.apps/deathstar created
pod/tiefighter created
pod/xwing created

# 파드 라벨 labels 확인
kubectl get pod --show-labels
NAME                        READY   STATUS    RESTARTS   AGE   LABELS
deathstar-8c4c77fb7-9klws   1/1     Running   0          29s   app.kubernetes.io/name=deathstar,class=deathstar,org=empire,pod-template-hash=8c4c77fb7
deathstar-8c4c77fb7-kkwds   1/1     Running   0          29s   app.kubernetes.io/name=deathstar,class=deathstar,org=empire,pod-template-hash=8c4c77fb7
tiefighter                  1/1     Running   0          29s   app.kubernetes.io/name=tiefighter,class=tiefighter,org=empire
xwing                       1/1     Running   0          29s   app.kubernetes.io/name=xwing,class=xwing,org=alliance

kubectl get deploy,svc,ep deathstar
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deathstar   2/2     2            2           114s

NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/deathstar   ClusterIP   10.96.154.223   <none>        80/TCP    114s

NAME                  ENDPOINTS                      AGE
endpoints/deathstar   172.20.1.34:80,172.20.2.1:80   114s

위 명령을 통해 Deathstar 서비스가 2개의 파드로 로드밸런싱되며, Tiefighter/Xwing 파드가 생성됩니다.

⸻

2. Cilium Endpoint & Identity 확인

아래 명령어로 Cilium 리소스를 조회하여, 현재 Policy가 비활성(Disabled) 상태임을 확인할 수 있습니다.

# 전체 Endpoint, Identity 리소스 조회
kubectl get ciliumendpoints.cilium.io -A
kubectl get ciliumidentities.cilium.io

# 노드별 Cilium Agent에서 Endpoint 목록 조회
kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium endpoint list

# alias를 사용한 단축 조회 (c0, c1, c2)
c0 endpoint list
c1 endpoint list
c2 endpoint list # 현재 ingress/egress 에 정책(Policy) 없음! , Labels 정보 확인

# Endpoint 상세 정보 및 라벨
ENDPOINT   POLICY (ingress)   POLICY (egress)   IDENTITY   LABELS (source:key[=value])
...
1579       Disabled           Disabled          318        k8s:app.kubernetes.io/name=deathstar                                                172.20.2.1    ready   
                                                           k8s:class=deathstar                                                                                       
                                                           k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default                                    
                                                           k8s:io.cilium.k8s.policy.cluster=default                                                                  
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                           
                                                           k8s:io.kubernetes.pod.namespace=default                                                                   
                                                           k8s:org=empire   

Policy(ingress/egress)가 Disabled 상태이므로, 이후 보안 정책 적용 전후를 비교 테스트할 수 있습니다.

## Check Current Access

> 현재 클러스터에서 org=empire 라벨이 없는 `xwing`, `tiefighter`도 Deathstar 서비스에 요청할 수 있는 상태입니다.  
> 아래 명령으로 각 Pod의 Identity ID를 확인하고, 모니터링을 준비합니다.

```bash
# Identity ID 확인 (c1, c2 노드)
c1 endpoint list | grep -iE 'xwing|tiefighter|deathstar'
c2 endpoint list | grep -iE 'xwing|tiefighter|deathstar'
XWINGID=17141
TIEFIGHTERID=56716
DEATHSTARID=8113

# 모니터링 준비: 터미널 3개에서 각각 Cilium Agent 모니터링
c0 monitor -v -v
c1 monitor -v -v
c2 monitor -v -v

# 모니터링 준비: 터미널 1개에서 Hubble 플로우 관찰
hubble observe -f

# Hubble 플로우 필터링 예시
hubble observe -f --from-identity $XWINGID
hubble observe -f --protocol udp --from-identity $XWINGID
hubble observe -f --protocol tcp --from-identity $XWINGID
hubble observe -f --protocol tcp --from-identity $DEATHSTARID

이제 xwing과 tiefighter에서 Deathstar Landing API를 호출하여, 실제 트래픽이 허용되는지 확인할 수 있습니다.

# 호출 시도 1: Xwing에서 요청
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
while true; do
  kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
  sleep 5
done

# 호출 시도 2: Tiefighter에서 요청
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
while true; do
  kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
  sleep 5
done

# 모니터링: TCP 플로우 확인
hubble observe -f --protocol tcp --from-identity $TIEFIGHTERID
hubble observe -f --protocol tcp --from-identity $DEATHSTARID


⸻

Apply an L3/L4 Policy

이제 org=empire 라벨이 있는 선박만 Deathstar 서비스에 접근하도록 L3/L4 정책을 적용합니다.
CiliumNetworkPolicy는 레이블을 기반으로 소스/목적지를 선택합니다.

# CiliumNetworkPolicy: empire 조직의 Deathstar 접근만 허용
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

# 호출 시도 2: Tiefighter에서 요청 (타임아웃 또는 연결 실패)
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

- 위 결과를 통해 org=empire 라벨을 가지지 않은 Pod의 Deathstar 접근이 차단됨을 확인할 수 있습니다.
## Inspecting the Policy

> CiliumNetworkPolicy가 `deathstar` 파드의 ingress에 적용되어 있는지 확인합니다.

```bash
# deathstar 에 ingress 에 policy 활성화 확인
c0 endpoint list
c1 endpoint list
c2 endpoint list
ENDPOINT   POLICY (ingress)   POLICY (egress)   IDENTITY   LABELS (source:key[=value]) 
...
1579       Enabled            Disabled          318        k8s:app.kubernetes.io/name=deathstar                                                172.20.2.1    ready   
                                                           k8s:class=deathstar                                                                                       
                                                           k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default                                    
                                                           k8s:io.cilium.k8s.policy.cluster=default                                                                  
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                           
                                                           k8s:io.kubernetes.pod.namespace=default                                                                   
                                                           k8s:org=empire  
                                                           
#
kc describe cnp rule1

위 출력에서 Enabled로 표시된 것을 통해 deathstar 파드의 ingress에 정책이 정상 적용되었음을 확인할 수 있습니다.
kc describe cnp rule1 명령으로 정책의 endpointSelector, ingress.fromEndpoints, toPorts 등의 상세 설정을 검토하세요.

⸻

Life of a Packet : L7 동작 처리는 cilium-envoy 데몬셋이 담당

Layer 7(HTTP/gRPC) 정책을 처리하기 위해 Envoy 프록시를 사용합니다. 아래 명령어로 데몬셋과 관련 리소스 상태를 확인합니다.

#
kubectl get ds -n kube-system
NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
cilium         3         3         3       3            3           kubernetes.io/os=linux   3h38m
cilium-envoy   3         3         3       3            3           kubernetes.io/os=linux   3h38m

kubectl get pod -n kube-system -l k8s-app=cilium-envoy -owide
NAME                 READY   STATUS    RESTARTS   AGE     IP               NODE      NOMINATED NODE   READINESS GATES
cilium-envoy-8427n   1/1     Running   0          3h38m   192.168.10.100   k8s-ctr   <none>           <none>
cilium-envoy-8rzsb   1/1     Running   0          3h36m   192.168.10.101   k8s-w1    <none>           <none>
cilium-envoy-vkqw6   1/1     Running   0          3h35m   192.168.10.102   k8s-w2    <none>           <none>

#
kc describe ds -n kube-system cilium-envoy
...
    Mounts:
      /sys/fs/bpf from bpf-maps (rw)
      /var/run/cilium/envoy/ from envoy-config (ro)
      /var/run/cilium/envoy/artifacts from envoy-artifacts (ro)
      /var/run/cilium/envoy/sockets from envoy-sockets (rw
   ...
   envoy-config:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      cilium-envoy-config
    Optional:  false
...

DaemonSet의 볼륨 마운트 정보를 통해 Envoy가 BPF 맵과 설정 파일, 소켓을 올바르게 마운트했는지 확인합니다.

kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- ss -xnp | grep -i -envoy
u_str ESTAB 0      0                                                  /var/run/cilium/envoy/sockets/xds.sock 32644            * 32610 users:(("cilium-agent",pid=1,fd=64))
u_str ESTAB 0      0                                                /var/run/cilium/envoy/sockets/admin.sock 29697            * 29345
u_str ESTAB 0      0                                                /var/run/cilium/envoy/sockets/admin.sock 29340            * 28672

cilium-agent가 Envoy의 XDS(xds.sock) 및 admin(admin.sock) 소켓과 연결되어 있는 것을 확인할 수 있습니다.

kc describe cm -n kube-system cilium-envoy-config
...
Data
====
bootstrap-config.json:
----
{"admin":{"address":{"pipe":{"path":"/var/run/cilium/envoy/sockets/admin.sock"}}}...
...

bootstrap-config.json에서 Envoy Bootstrap 설정(관리 소켓 경로 등)의 세부 내용을 검토합니다.

## Apply and Test HTTP-aware L7 Policy

> HTTP 계층(L7) 정책을 적용하여, 타이파이터가 호출할 수 있는 URL을 **POST /v1/request-landing** 으로 제한하고, 그 외(예: PUT /v1/exhaust-port) 호출을 차단합니다.

---

### 1. L3/L4 모니터링만으로는 HTTP 경로를 구분할 수 없음

```bash
# 모니터링 >> Layer3/4 에서는 애플리케이션 상태를 확인 할 수 없음!
hubble observe -f --protocol tcp --from-identity $DEATHSTARID

# 호출해서는 안 되는 일부 유지보수 API를 노출
kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port


⸻

2. L7-aware 정책 정의 (rule1 업데이트)

# 기존 rule1 정책을 업데이트 해서 사용
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

# Update the existing rule to apply L7-aware policy to protect deathstar using:
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/sw_l3_l4_l7_policy.yaml
kubectl get cnp
kc describe cnp
c0 policy get


⸻

3. HTTP 플로우 관찰 및 테스트

# 모니터링 : 파드 이름 지정
hubble observe -f --pod deathstar --protocol http
Jul 20 01:28:02.184: default/tiefighter:59020 (ID:19274) -> default/deathstar-8c4c77fb7-9klws:80 (ID:318) http-request FORWARDED (HTTP/1.1 POST http://deathstar.default.svc.cluster.local/v1/request-landing)
Jul 20 01:28:02.190: default/tiefighter:59020 (ID:19274) <- default/deathstar-8c4c77fb7-9klws:80 (ID:318) http-response FORWARDED (HTTP/1.1 200 6ms (POST http://deathstar.default.svc.cluster.local/v1/request-landing))

# We can now re-run the same test as above, but we will see a different outcome
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing


⸻

4. 차단된 L7 트래픽 관찰

# 모니터링 : 파드 이름 지정
hubble observe -f --pod deathstar --verdict DROPPED
Jul 20 01:31:21.248: default/tiefighter:44734 (ID:19274) -> default/deathstar-8c4c77fb7-9klws:80 (ID:318) http-request DROPPED (HTTP/1.1 PUT http://deathstar.default.svc.cluster.local/v1/exhaust-port)

혹은
c1 monitor -v --type l7
<- Request http from 3845 ([k8s:app.kubernetes.io/name=tiefighter k8s:class=tiefighter k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=default k8s:io.kubernetes.pod.namespace=default k8s:org=empire]) to 871 ([k8s:app.kubernetes.io/name=deathstar k8s:class=deathstar k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=default k8s:io.kubernetes.pod.namespace=default k8s:org=empire]), identity 19274->318, verdict Denied PUT http://deathstar.default.svc.cluster.local/v1/exhaust-port => 0
<- Response http to 3845 ([k8s:app.kubernetes.io/name=tiefighter k8s:class=tiefighter k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=default k8s:io.kubernetes.pod.namespace=default k8s:org=empire]) from 871 ([k8s:app.kubernetes.io/name=deathstar k8s:class=deathstar k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=default k8s:io.kubernetes.pod.namespace=default k8s:org=empire]), identity 318->19274, verdict Forwarded PUT http://deathstar.default.svc.cluster.local/v1/exhaust-port => 403
c2 monitor -v --type l7

위 테스트를 통해 POST 요청만 FORWARDED 되고, PUT 요청은 L7 정책에 의해 DROPPED(또는 403) 되는 것을 확인할 수 있습니다.

## Re-run Test & Observations

> L7 정책 적용 후 PUT 요청이 거부되고, POST 요청이 정상 처리되는지 확인합니다.

```bash
# We can now re-run the same test as above, but we will see a different outcome
kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Access denied



# 모니터링 : 파드 이름 지정
hubble observe -f --pod xwing

# 호출 시도 : 위와 아래 실행 종료의 차이점을 이해해보자!
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing --connect-timeout 2


⸻

Resource Cleanup

다음 실습을 위해 생성된 애플리케이션 리소스와 정책을 삭제합니다.

# 다음 실습을 위해 리소스 삭제
kubectl delete -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/minikube/http-sw-app.yaml
kubectl delete cnp rule1

# 삭제 확인
kubectl get cnp


⸻

Configuring Hubble Exporter

Hubble 흐름 로그를 파일로 저장하고, 로그 회전(rotation), 크기 제한, 필터, 필드 마스크를 지원합니다.
이 설정은 Hubble 설치 시 이미 기본 적용되어 있습니다.

# 설정 : 아래 설정할 필요 없음
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
   --set hubble.enabled=true \
   --set hubble.export.static.enabled=true \
   --set hubble.export.static.filePath=/var/run/cilium/hubble/events.log

kubectl -n kube-system rollout status ds/cilium

# 확인
kubectl get cm -n kube-system cilium-config -o json | grep hubble-export
cilium config view | grep hubble-export
hubble-export-allowlist                           
hubble-export-denylist                            
hubble-export-fieldmask                           
hubble-export-file-max-backups   # number of rotated Hubble export files to keep. (default 5)
hubble-export-file-max-size-mb   # size in MB at which to rotate the Hubble export file. (default 10)
hubble-export-file-path          # file path of target log file. (default /var/run/cilium/hubble/events.log)

# Verify that flow logs are stored in target files
kubectl -n kube-system exec ds/cilium -- tail -f /var/run/cilium/hubble/events.log
kubectl -n kube-system exec ds/cilium -- sh -c 'tail -f /var/run/cilium/hubble/events.log' | jq

## Performance Tuning for Hubble Exporter

> Hubble Exporter의 성능 최적화를 위해 설정할 수 있는 주요 옵션과 구성 예시입니다.

### 주요 설정 옵션
- `hubble.export.static.allowList`  
  JSON 인코딩된 FlowFilters로 **허용 리스트** 지정
- `hubble.export.static.denyList`  
  JSON 인코딩된 FlowFilters로 **차단 리스트** 지정
- `hubble.export.static.fieldMask`  
  Flow 프로토 정의의 필드 이름 **리스트**로 **필드 마스킹** 지정

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

Dynamic Exporter Configuration

Pod 재시작 없이 동적으로 여러 필터를 구성하고, 별도 파일로 로그를 저장할 수 있습니다.

# Dynamic exporter 활성화
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \
   --set hubble.enabled=true \
   --set hubble.export.static.enabled=false \
   --set hubble.export.dynamic.enabled=true

kubectl -n kube-system rollout status ds/cilium

static exporter를 비활성화하고 dynamic exporter를 활성화합니다.

# Dynamic flow logs 예시: system 흐름을 별도 파일에 저장하고 kube-system 네임스페이스 필터 적용
helm upgrade cilium cilium/cilium --version 1.17.6 \
   --set hubble.enabled=true \
   --set hubble.export.dynamic.enabled=true \
   --set hubble.export.dynamic.config.content[0].name=system \
   --set hubble.export.dynamic.config.content[0].filePath=/var/run/cilium/hubble/events-system.log \
   --set hubble.export.dynamic.config.content[0].includeFilters[0].source_pod[0]='kube_system/' \
   --set hubble.export.dynamic.config.content[0].includeFilters[1].destination_pod[0]='kube_system/'

# Dynamic exporter는 end property, field masking 등 static과 동일한 기능을 지원하며,
# maxOutputFileSizeMb 및 fileMaxBackups는 static 설정을 재활용합니다.

Dynamic exporter를 사용하면 여러 필터 세트를 동시에 관리하고, 구성 변경을 위해 Pod 재시작이 필요 없습니다.
## 2.1 샘플 애플리케이션 배포

> 웹 애플리케이션 역할을 하는 `webpod` Deployment(2 복제본)와 ClusterIP Service를 생성합니다.  
> `podAntiAffinity` 설정으로 동일 호스트에 중복 배치되지 않도록 보장합니다.

```bash
# 샘플 애플리케이션 배포
cat << EOF | kubectl apply -f -
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
EOF


⸻

2.2 curl-pod 배포

k8s-ctr 노드에서 HTTP 요청을 보낼 수 있는 curl-pod를 배포합니다.
nicolaka/netshoot 이미지를 사용하여 다양한 네트워크 도구를 활용할 수 있습니다.

# k8s-ctr 노드에 curl-pod 파드 배포
cat <<EOF | kubectl apply -f -
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
EOF


⸻

2.3 배포 확인 및 통신 테스트

생성된 리소스 상태를 확인하고, curl-pod에서 webpod 서비스로 요청을 반복하여 응답을 검증합니다.

# 배포 확인
kubectl get deploy,svc,ep webpod -owide
kubectl get endpointslices -l app=webpod
kubectl get ciliumendpoints
kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium-dbg endpoint list

# 통신 확인
kubectl exec -it curl-pod -- curl webpod | grep Hostname
kubectl exec -it curl-pod -- sh -c 'while true; do curl -s webpod | grep Hostname; sleep 1; done'

위 확인 과정을 통해:
	•	Deployment/Service/Endpoint가 정상 생성되었는지,
	•	Cilium CNI 관점에서 Endpoint가 등록되었는지,
	•	curl-pod에서 webpod로 요청 시 각 파드의 Hostname이 반복 출력되는지 확인할 수 있습니다.
## 3. Install Prometheus & Grafana

> Prometheus와 Grafana를 하나의 배포로 포함한 예제입니다.  
> 이 배포는 **Cilium** 및 **Hubble** 메트릭을 자동으로 스크랩하도록 구성되어 있습니다.  
> - **Grafana**: Cilium Dashboard가 미리 로드된 시각화 대시보드  
> - **Prometheus**: 시계열 데이터베이스 및 모니터링 시스템  

```bash
#
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.17.6/examples/kubernetes/addons/prometheus/monitoring-example.yaml
configmap/grafana-config created
configmap/grafana-cilium-dashboard created
configmap/grafana-cilium-operator-dashboard created
configmap/grafana-hubble-dashboard created
configmap/grafana-hubble-l7-http-metrics-by-workload created
configmap/prometheus created

#
kubectl get deploy,pod,svc,ep -n cilium-monitoring
kubectl get cm -n cilium-monitoring
NAME                                         DATA   AGE
grafana-cilium-dashboard                     1      113s
grafana-cilium-operator-dashboard            1      113s
grafana-config                               3      113s
grafana-hubble-dashboard                     1      113s
grafana-hubble-l7-http-metrics-by-workload   1      113s
prometheus                                   1      113s

# 프로메테우스 서버 설정 확인
kc describe cm -n cilium-monitoring prometheus

# 그라파나 서버 설정 확인
kc describe cm -n cilium-monitoring grafana-config

# Grafana 대시보드 주입 ConfigMap 확인
kc describe cm -n cilium-monitoring grafana-cilium-dashboard
kc describe cm -n cilium-monitoring grafana-hubble-dashboard
...

위 명령으로 cilium-monitoring 네임스페이스에 Prometheus, Grafana, 그리고 관련 ConfigMap들이 생성 및 주입되었음을 확인합니다.

⸻

4. Cilium & Hubble 메트릭 활성화

Cilium, Cilium-Operator, Hubble는 기본적으로 메트릭을 노출하지 않으므로, Helm 값을 통해 포트(9962, 9963, 9965)를 열어 스크래핑하도록 합니다.

# 이미 설정되어 있음
helm install cilium cilium/cilium --version 1.17.6 \
   --namespace kube-system \
   --set prometheus.enabled=true \
   --set operator.prometheus.enabled=true \
   --set hubble.enabled=true \
   --set hubble.metrics.enableOpenMetrics=true \
   --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}"

# 호스트에서 메트릭 포트 열림 확인
ss -tnlp | grep -E '9962|9963|9965'
LISTEN 0      4096                *:9963             *:*    users:(("cilium-operator",pid=1488,fd=7)) # cilium-operator 메트릭        
LISTEN 0      4096                *:9962             *:*    users:(("cilium-agent",pid=1894,fd=7))    # cilium 메트릭  
LISTEN 0      4096                *:9965             *:*    users:(("cilium-agent",pid=1894,fd=40))   # hubble 메트릭

# 워커 노드별 포트 확인
for i in w1 w2 ; do
  echo ">> node : k8s-$i <<"
  sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo ss -tnlp | grep -E '9962|9963|9965'
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