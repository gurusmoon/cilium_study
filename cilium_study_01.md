
# Cilium Study 가이드 - 01 주차

> 이 문서는 CloudNet@ 팀에서 진행한 Cilium Study 내용을 정리한 자료입니다.
> 본 스터디를 제공해주신 가시다님과 CloudNet@ 팀 모든 분들께 감사드립니다.

## 목차
1. [환경 구성](#1-환경-구성)
   - 필수 도구 설치
   - 실습 환경 구성
   - 클러스터 설정
2. [Flannel CNI](#2-flannel-cni)
   -#### 컨테이너 디버깅 및 런타임 관리

containerd 기반의 컨테이너 런타임을 직접 제어하고 문제를 진단합니다:

| 명령어 | 설명 |
|--------|------|
| `crictl ps` | 실행 중인 컨테이너 목록을 확인합니다. |
| `crictl logs <container-id>` | 특정 컨테이너의 로그를 확인합니다. |
| `crictl inspect <container-id>` | 컨테이너의 상세 설정과 상태를 검사합니다. |
| `crictl exec -it <container-id> <command>` | 실행 중인 컨테이너에 접속하여 명령을 실행합니다. |

다중 노드 컨테이너 상태 확인:

| 명령어 | 설명 |
|--------|------|
| `for i in w1 w2 ; do`<br>`  echo ">> node : k8s-$i <<"`<br>`  sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo crictl ps`<br>`  echo`<br>`done` | - 각 워커 노드(`k8s-w1`, `k8s-w2`)의 컨테이너 목록 확인<br>- containerd 기반 컨테이너의 저수준 상태 확인<br>- `kubectl`보다 더 상세한 런타임 정보 제공<br>- 장애 진단 및 문제 해결에 유용 |

> **디버깅 베스트 프랙티스**
> - 컨테이너 ID는 `crictl ps`로 먼저 확인
> - 로그 확인 시 `--tail` 옵션으로 출력량 제한
> - 문제 발생 시 `crictl inspect`로 상세 설정 검토
> - 네트워크 이슈는 `crictl exec`로 내부 진단 및 구성
   - 네트워크 확인
3. [Cilium CNI](#3-cilium-cni)
   - 설치 준비
   - 구성 및 설치
   - 네트워크 확인

## 1. 환경 구성

## 1. 환경설정

### 1.1 필수 도구 설치

#### VirtualBox 설치

VirtualBox를 통해 가상 머신 환경을 구성합니다:

> **설치 확인 화면**
> - VirtualBox 버전: 7.x.x
> - CLI 도구: VBoxManage 사용 가능
> - 가상화 지원: 하드웨어 가상화 활성화 상태

<img width="1077" height="269" alt="VirtualBox 설치 및 버전 확인" src="https://github.com/user-attachments/assets/c6e4e599-95fd-49e5-b5e5-dac7f6f8d60c" />

#### Vagrant 설치
- Vagrant를 사용하여 가상 머신 프로비저닝을 자동화합니다.
<img width="1072" height="241" alt="Vagrant 설치 확인" src="https://github.com/user-attachments/assets/7f8a4bf4-4d20-448e-b1fb-5a578893f3a5" />

### 1.2 실습 환경 구성

#### 클러스터 구성도

실습에 사용될 Kubernetes 클러스터의 구성입니다:

> **구성 요소**
> - Control Plane 노드 1대
> - Worker 노드 2대
> - 내부 네트워크: 192.168.10.0/24
> - 외부 연결: NAT를 통한 인터넷 접속
> - CNI: 초기 상태는 미설치 (Flannel -> Cilium 전환 예정)

<img width="1750" height="614" alt="Kubernetes 클러스터 구성도" src="https://github.com/user-attachments/assets/1424a85d-0aad-405f-b02d-2de2ee0c59c2" />

#### 노드 구성
| 노드 유형 | 호스트명 | IP 주소 |
|-----------|----------|----------|
| Control Plane | `k8s-ctr` | `192.168.10.100` |
| Worker 1 | `k8s-w1` | `192.168.10.101` |
| Worker 2 | `k8s-w2` | `192.168.10.102` |

#### 공통 환경 설정
- **네트워크 인터페이스**
  - `eth0`: NAT (IP: `10.0.2.15`, 모든 노드 동일)
  - `eth1`: Private network (고정 IP 할당)
- **시스템 환경**
  - OS: Ubuntu 24.04 베이스 이미지
  - 클러스터: `kubeadm`으로 초기화 및 워커 노드 조인 완료
  - 상태: **CNI 미설치 상태**

---

### 1.3 실습 환경 자동화

#### Vagrantfile 구성

Vagrant를 통해 실습 환경을 자동으로 구성합니다:

- **주요 설정**
  - VM 정의 및 초기 설치 자동화
  - 쿠버네티스/컨테이너 버전 관리
  - Control Plane 및 Worker 노드 자동 생성

```ruby
# Vagrantfile 주요 설정값
K8SV = '1.33.2-1.1'       # Kubernetes 버전
CONTAINERDV = '1.7.27-1'  # Containerd 버전
N = 2                     # Worker 노드 수

# 기본 이미지 설정
BOX_IMAGE = "bento/ubuntu-24.04"
BOX_VERSION = "202502.21.0"
```
#### VirtualBox
<img width="1009" height="506" alt="image" src="https://github.com/user-attachments/assets/60ce0daa-6a1b-4a47-b6be-0d68d856aa2a" />

#### 노드들 eth0 ip 확인
<img width="1079" height="268" alt="image" src="https://github.com/user-attachments/assets/49e48f2a-d95f-469b-9206-b2a193a7231f" />

### 1.4 클러스터 기본 정보 확인

기본적인 시스템 상태와 네트워크 연결성을 확인합니다.

#### 시스템 정보 확인

기본적인 시스템 설정과 상태를 확인합니다:

| 명령어 | 설명 |
|--------|------|
| `whoami` | 현재 로그인된 사용자 이름을 확인합니다. (`root`) |
| `pwd` | 현재 작업 중인 디렉토리 경로를 확인합니다. (`/root`) |
| `hostnamectl` | 호스트 이름, OS, 커널, 가상화, 아키텍처 등 시스템 정보를 확인합니다. |
| `htop` | CPU, 메모리 사용량, 프로세스 목록 등 시스템 리소스를 실시간으로 모니터링합니다. |

> **확인된 시스템 정보**
> - 호스트명: k8s-ctr (Control Plane 노드)
> - OS: Ubuntu 24.04 LTS
> - 커널: Linux 6.x
> - 가상화: VirtualBox 환경에서 실행 중

<img width="451" height="214" alt="시스템 기본 정보 확인 결과" src="https://github.com/user-attachments/assets/99e9d932-85f1-45fe-a51c-157c2df02d3d" />

#### 네트워크 연결 확인
| 명령어 | 설명 |
|--------|------|
| `cat /etc/hosts` | 호스트 파일의 IP 주소와 호스트 이름 매핑을 확인합니다. |
| `ping -c 1 k8s-w1` | Worker 1 노드(`192.168.10.101`)와의 네트워크 연결을 확인합니다. |
| `ping -c 1 k8s-w2` | Worker 2 노드(`192.168.10.102`)와의 네트워크 연결을 확인합니다. |
<img width="614" height="405" alt="image" src="https://github.com/user-attachments/assets/81a84007-dfde-4f52-b5d5-ed22b907a24a" />

#### SSH 접속 테스트
| 명령어 | 설명 |
|--------|------|
| `sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-w1 hostname` | Worker 1 노드에 SSH 접속하여 호스트 이름을 확인합니다. |
| `sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-w2 hostname` | Worker 2 노드에 SSH 접속하여 호스트 이름을 확인합니다. |

#### 네트워크 인터페이스 상태 확인
| 명령어 | 설명 |
|--------|------|
| `ss -tnp \| grep sshd` | 현재 SSH 연결 상태를 확인합니다. |
| `ip -c addr` | 컬러로 구분된 IP 인터페이스 정보를 확인합니다. |
| `ip -c route` | 컬러로 구분된 라우팅 테이블을 확인합니다. |
|----------------------------------------|------|
| `ss -tnp | grep sshd`                  | 현재 SSH 연결 중인 세션 확인 (ESTAB: 연결 상태) |
| `ip -c addr`                           | 컬러로 구분된 IP 인터페이스 및 주소 정보 출력 |
| `ip -c route`                          | 컬러로 구분된 라우팅 테이블 출력 |
<img width="1012" height="437" alt="image" src="https://github.com/user-attachments/assets/9c21c93e-1e0c-4066-80c6-c935f8f7b8b3" />

| 명령어                     | 설명                                              |
|----------------------------|---------------------------------------------------|
| resolvectl                 | 시스템의 DNS 설정 정보를 조회하는 명령어          |
<img width="836" height="184" alt="image" src="https://github.com/user-attachments/assets/f8736c1b-b6cb-45c4-b724-d1e2eb537db5" />

---

## 2. Kubernetes 클러스터 구성

### 2.1 클러스터 상태 확인

클러스터의 전반적인 상태와 구성을 확인합니다:
| 명령어 | 설명 |
|--------|------|
| `kubectl cluster-info` | 클러스터의 컨트롤 플레인 및 CoreDNS 상태를 확인합니다. |
| `kubectl get node -o wide` | 노드의 상태, IP, OS, 컨테이너 런타임 정보를 확인합니다. |
| `kubectl get pod -A -o wide` | 모든 네임스페이스의 파드 상태와 위치를 확인합니다. |

#### 클러스터 구성 확인
| 명령어 | 설명 |
|--------|------|
| `cat /var/lib/kubelet/kubeadm-flags.env` | kubelet 설정 환경 변수를 확인합니다. |
| `NODEIP=$(ip -4 addr show eth1 \| grep -oP '(?<=inet\s)\d+(\.\d+){3}')` | eth1 인터페이스의 IPv4 주소를 추출합니다. |
| `echo $NODEIP` | 추출된 노드 IP를 확인합니다. |

| 명령어                                                                                             | 설명                                                                                     |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| cat /var/lib/kubelet/kubeadm-flags.env                                                             | kubelet 실행 시 설정되는 환경 변수 확인                                                  |
| echo $NODEIP                                                                                       | NODEIP 변수에 저장된 IP 주소 출력 (ex. 192.168.10.100)                                    |
| sed -i "s/^(KUBELET_KUBEADM_ARGS=\"[^\"]*)\"/\1 --node-ip=${NODEIP}\"/" ...                        | kubelet 설정 파일에 `--node-ip=...` 옵션을 추가                                          |
| systemctl daemon-reexec && systemctl restart kubelet                                               | systemd 재실행 및 kubelet 서비스 재시작                                                  |
| cat /var/lib/kubelet/kubeadm-flags.env                                                             | 설정 파일 수정 결과 확인                                                                 |
| kubectl get node -o wide                                                                           | 클러스터 노드 상태, 내부 IP, OS, 커널, 컨테이너 런타임 정보 등 상세 정보 출력           |
<img width="1459" height="300" alt="image" src="https://github.com/user-attachments/assets/60e54a2b-bc5e-4973-82ac-6752930cc7cf" />
#### k8s-w1, k8s-w1에서 internal ip 변경설정
<img width="1360" height="325" alt="image" src="https://github.com/user-attachments/assets/4d9f76db-3fa4-441f-9ce1-702071cff720" />


| 명령어                                                                                                               | 해석                                                                                 |
|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| kubectl cluster-info dump \| grep -m 2 -E "cluster-cidr\|service-cluster-ip-range"                                   | 클러스터 CIDR 및 서비스 CIDR 정보를 `cluster-info dump`로 출력                       |
| kubectl get pod -n kube-system -l k8s-app=kube-dns -o wide                                                           | kube-system 네임스페이스에서 CoreDNS 파드들의 상태 및 노드 정보 확인                |
<img width="1133" height="140" alt="image" src="https://github.com/user-attachments/assets/218291e3-7122-4d9a-a60d-56e65b7dd02e" />
<img width="1104" height="778" alt="image" src="https://github.com/user-attachments/assets/de074aa4-19c7-4c2d-8247-604b7268e834" />

| 인터페이스 | 역할             | IP 주소         | 비고               |
|------------|------------------|------------------|--------------------|
| eth0       | NAT (외부 통신용) | 10.0.2.15        | Vagrant 기본       |
| eth1       | 내부 네트워크     | 192.168.10.100   | 클러스터 통신용    |
| lo         | 루프백           | 127.0.0.1        | 내부 전용          |

<img width="1465" height="562" alt="image" src="https://github.com/user-attachments/assets/e9b19911-4a53-4d8e-8166-3ea5fe694bb8" />
<img width="1460" height="680" alt="image" src="https://github.com/user-attachments/assets/658e5de4-4824-48b9-886e-859b4eee40de" />
<img width="2940" height="372" alt="image" src="https://github.com/user-attachments/assets/2c14c098-d25a-4d1d-8ea2-3523fa32f838" />
<img width="1470" height="84" alt="image" src="https://github.com/user-attachments/assets/78dbd24d-86b8-44f5-8c27-1a265898f9d8" />

#### install flannel
<img width="1470" height="264" alt="image" src="https://github.com/user-attachments/assets/1db3793a-3431-4a0b-b1ae-58ed93b6d045" />

| 명령어 | 설명 |
|--------|------|
| `kc describe pod -n kube-flannel -l app=flannel` | Flannel Pod 상태 확인 |
| `tree /opt/cni/bin/` | CNI 바이너리 존재 여부 확인 |
| `tree /etc/cni/net.d/` | CNI 설정 파일 위치 확인 |
| `cat /etc/cni/net.d/10-flannel.conflist \| jq` | CNI 설정 내용 상세 확인 |
| `kc describe cm -n kube-flannel kube-flannel-cfg` | Flannel 네트워크 설정 ConfigMap 확인 |

<img width="1470" height="401" alt="image" src="https://github.com/user-attachments/assets/fe0741cc-0266-40d2-8ee4-eb918c869084" />
<img width="1465" height="598" alt="image" src="https://github.com/user-attachments/assets/76169cbd-376c-4539-9195-92d4584917d6" />
<img width="2914" height="808" alt="image" src="https://github.com/user-attachments/assets/567a3ebf-0f36-4ff4-88d4-7bd4f7154ae0" />
<img width="1467" height="265" alt="image" src="https://github.com/user-attachments/assets/1f05076d-71c3-4103-9978-6d93f3033e26" />
<img width="1463" height="564" alt="image" src="https://github.com/user-attachments/assets/bfbf9076-e492-4e64-8363-3f9baa92942c" />
<img width="1467" height="644" alt="image" src="https://github.com/user-attachments/assets/b639d003-ffb4-4c85-bb57-b9b523a3731f" />

| 명령어 | 설명 |
|--------|------|
| ip -c link | 노드 내 네트워크 인터페이스 상태 확인 (flannel, veth 등) |
| ip -c route | Pod 네트워크 라우팅 경로 확인 |
| brctl show | CNI 브리지(`cni0`)와 연결된 veth 확인 |
| sudo iptables -t nat -S | 쿠버네티스/Flannel이 생성한 NAT 룰 확인 |

| 명령어 | 설명 |
|--------|------|
| `for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i ip -c link ; echo; done` | 각 워커 노드의 네트워크 인터페이스 상태 확인 (`eth0`, `flannel.1`, `veth` 등) |
| `for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i ip -c route ; echo; done` | 각 워커 노드의 라우팅 테이블 확인 (Pod 네트워크 `10.244.x.x` 경로 등) |
| `for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i brctl show ; echo; done` | 브리지(`cni0`)와 연결된 인터페이스(veth) 확인 |
| `for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i sudo iptables -t nat -S ; echo; done` | iptables NAT 테이블의 쿠버네티스/Flannel 관련 규칙 확인 (`KUBE-SERVICES` 등) |

| 명령어 | 설명 |
|--------|------|
| `for i in w1 w2 ; do`<br>`  echo ">> node : k8s-$i <<"`<br>`  sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo crictl ps`<br>`  echo`<br>`done` | `k8s-w1`, `k8s-w2` 두 워커 노드 각각에 SSH로 접속하여,<br>해당 노드에서 실행 중인 **containerd 기반 컨테이너 목록**을 `crictl ps`로 출력하는 반복 명령어입니다. <br><br>컨테이너 런타임의 저수준 상태를 확인할 수 있어, `kubectl`보다 더 세밀한 장애 진단에 유용합니다. |
<img width="1469" height="825" alt="image" src="https://github.com/user-attachments/assets/f00486b2-40ab-4f4a-9019-6820b57d475b" />
<img width="1469" height="327" alt="image" src="https://github.com/user-attachments/assets/00709d58-e951-4e4d-8b05-1d03ebd0ade5" />
<img width="1444" height="363" alt="image" src="https://github.com/user-attachments/assets/fd99ff9b-e77d-45da-91f5-1ce5d898187c" />
<img width="1465" height="804" alt="image" src="https://github.com/user-attachments/assets/7f5daf60-00a3-4ffa-b54c-84d52838ee0f" />
<img width="1466" height="331" alt="image" src="https://github.com/user-attachments/assets/b9e3ec05-187b-472c-9d78-0448424979a5" />

#### 기존 Flannel CNI 제거


## 3. Cilium CNI 설치 및 구성

### 3.1 설치 준비

#### 시스템 요구사항 확인
<img width="1467" height="822" alt="image" src="https://github.com/user-attachments/assets/9e5201bd-b013-4108-95ae-82e3b8076d19" />
<img width="1469" height="225" alt="image" src="https://github.com/user-attachments/assets/59d2d048-f5bc-4247-92d8-064641571874" />

### 3.2 Cilium 설치

Helm을 사용하여 Cilium을 설치하고 구성합니다:

| 명령어 | 설명 |
|--------|------|
| `helm repo add cilium https://helm.cilium.io` | Helm 저장소에 Cilium Chart를 추가 |
| `helm install cilium cilium/cilium ...` | Cilium CNI 플러그인을 Helm을 통해 설치 |
| `--version 1.17.5` | Cilium Helm Chart의 버전 지정 |
| `--namespace kube-system` | 설치할 네임스페이스를 `kube-system`으로 설정 |
| `--set k8sServiceHost=192.168.10.100` | Kubernetes API 서버의 IP 주소 지정 |
| `--set k8sServicePort=6443` | Kubernetes API 서버의 포트 번호 설정 |
| `--set kubeProxyReplacement=true` | kube-proxy 기능을 Cilium이 완전히 대체하도록 설정 |
| `--set routingMode=native` | Cilium이 Linux 커널의 기본 라우팅 기능 사용 |
| `--set autoDirectNodeRoutes=true` | 노드 간 직접 경로를 자동으로 설정 |
| `--set ipam.mode="cluster-pool"` | IP 주소 할당 방식을 클러스터 풀 방식으로 설정 |
| `--set ipam.operator.clusterPoolIPv4PodCIDRList={"172.20.0.0/16"}` | Pod IP 범위를 172.20.0.0/16으로 지정 |
| `--set ipv4NativeRoutingCIDR=172.20.0.0/16` | 노드 간 native routing 시 사용할 CIDR 설정 |
| `--set endpointRoutes.enabled=true` | 각 Pod에 대해 고유한 라우팅 경로 설정 허용 |
| `--set installNoConntrackIptablesRules=true` | iptables에서 conntrack을 사용하지 않도록 설정 |
| `--set bpf.masquerade=true` | BPF를 사용하여 SNAT 처리 허용 |
| `--set ipv6.enabled=false` | IPv6 기능 비활성화 |
<img width="1467" height="525" alt="image" src="https://github.com/user-attachments/assets/7e8cb879-3645-445e-85a6-d0a031fe2fc6" />

| 명령어 | 설명 |
|--------|------|
| `helm get values cilium -n kube-system` | Helm으로 설치된 Cilium의 사용자 정의 설정 값을 조회 |
| `helm list -A` | 클러스터 내 설치된 Helm 릴리스 목록 확인 |
| `kubectl get crd` | 클러스터에 설치된 Cilium 관련 CRD 목록 조회 |
| `watch -d kubectl get pod -A` | 모든 네임스페이스의 Pod 상태를 실시간 모니터링 |
<img width="1467" height="743" alt="image" src="https://github.com/user-attachments/assets/dedfaeb9-0a55-4ca0-8f18-5537c671cddd" />

| 명령어                                                                                                          | 설명                                                                                      |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| `kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium-dbg status --verbose`                    | Cilium 에이전트의 상세 상태 확인 (kube-proxy 대체 여부, 라우팅, masquerading 등 포함)       |
| `iptables -t nat -S`                                                                                           | 현재 노드(로컬)의 NAT 테이블에 설정된 체인/규칙 목록 출력                                 |
| `for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo iptables -t nat -S ; echo; done` | 각 워커 노드(k8s-w1, k8s-w2)의 NAT 테이블 상태 출력                                      |
| `iptables-save`                                                                                                | 현재 노드의 전체 iptables 설정 출력                                                       |
| `for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh vagrant@k8s-$i sudo iptables-save ; echo; done`     | 각 워커 노드의 전체 iptables 설정 저장된 내용 출력                                       |

#### Pod CIDR IPAM 확인
| 명령어 | 설명 |
|--------|------|
| `kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'` | Kubernetes 노드별 Pod CIDR 범위 확인 |
| `kubectl get pod -o wide` | 파드의 IP, 노드, 상태 등 상세 정보 확인 |
| `kubectl get ciliumnodes` | Cilium이 할당한 내부 IP 확인 |
| `kubectl get ciliumnodes -o json | grep podCIDRs -A2` | Cilium의 PodCIDR 할당 확인 |
| `kubectl rollout restart deployment webpod` | 기존 파드 재시작 (새 네트워크 설정 적용) |
| `kubectl get pod -owide` | 파드 재생성 후 IP 적용 여부 확인 |
<img width="1466" height="661" alt="image" src="https://github.com/user-attachments/assets/a8074f79-ac82-4274-b20f-821d44da1e49" />

| 명령어 | 설명 |
|--------|------|
| kubectl delete pod curl-pod --grace-period=0 | `curl-pod`를 즉시 강제 삭제 |
| cat <<EOF ... | kubectl apply -f - | 새로운 `curl-pod` 생성 |
| kubectl get pod -owide | 파드 상태 및 IP 확인 |
| kubectl get ciliumnendpoints | Cilium에서 파드가 추적되고 있는지 확인 |
<img width="1468" height="581" alt="image" src="https://github.com/user-attachments/assets/b66b8f9e-ab9d-4032-8d4d-f8b6e5d2f304" />

| 명령어 | 설명 |
|--------|------|
| kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium-dbg endpoint list | Cilium에 의해 관리되는 Endpoint 상태와 정책 라벨 등을 확인 |
<img width="1470" height="751" alt="image" src="https://github.com/user-attachments/assets/f8a3980b-255b-4403-ac6b-3383724687f8" />

| 명령어 | 설명 |
|--------|------|
| kubectl exec -it curl-pod -- curl webpod | curl-pod 파드에서 webpod 서비스로 HTTP 요청 전송 |
| grep Hostname | 응답에서 Hostname 라인이 포함된 부분만 출력 |
<img width="1469" height="86" alt="image" src="https://github.com/user-attachments/assets/c768b348-af7f-4eb9-abda-ff7c357f3086" />

#### Cilium 설치 확인
### 3.3 Cilium CLI 설치 및 구성

Cilium CLI를 설치하고 기본 설정을 진행합니다:

| 명령어 | 설명 |
|--------|------|
| `CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)` | 최신 Cilium CLI 버전 정보를 GitHub에서 가져와 `CILIUM_CLI_VERSION` 변수에 저장합니다. |
| `CLI_ARCH=amd64` | 기본 아키텍처를 amd64로 설정합니다. |
| `if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi` | 시스템 아키텍처가 `aarch64`이면 ARM64용으로 설정합니다. |
| `curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz >/dev/null 2>&1` | 위에서 지정한 버전 및 아키텍처에 맞는 Cilium CLI 바이너리를 GitHub에서 다운로드합니다. 출력은 화면에 표시되지 않도록 처리합니다. |
| `tar xzvf cilium-linux-${CLI_ARCH}.tar.gz -C /usr/local/bin` | 다운로드한 `.tar.gz` 파일을 `/usr/local/bin` 경로로 압축 해제하여 실행 파일로 설치합니다. |
| `rm cilium-linux-${CLI_ARCH}.tar.gz` | 설치가 끝난 후 불필요한 압축 파일을 삭제합니다. |
<img width="1470" height="162" alt="image" src="https://github.com/user-attachments/assets/8525aca4-c8ed-48db-ac66-4c50b87d7bdf" />

##### Cilium 상태확인
| 명령어 | 설명 |
|--------|------|
| `which cilium` | 시스템에 설치된 `cilium` CLI 바이너리 경로를 확인합니다. 설치 여부 확인용. |
| `cilium status` | 현재 노드에서 실행 중인 Cilium 에이전트 및 Hubble 상태를 확인합니다. |
| `cilium config view` | Cilium 에이전트의 현재 설정값을 CLI 상에서 조회합니다. (`cilium-config` ConfigMap 기반) |
| `kubectl get cm -n kube-system cilium-config -o json \| jq` | `cilium-config`를 JSON 형태로 가져와 `jq`로 가독성 좋게 출력합니다. |
| `cilium config set debug true` | Cilium 에이전트 설정 중 `debug` 옵션을 `true`로 설정합니다. 재시작 후 적용됩니다. |
| `watch kubectl get pod -A` | Cilium 파드 재시작 등을 실시간으로 모니터링하기 위해 전체 네임스페이스의 파드 상태를 지속적으로 출력합니다. |
| `cilium config view \| grep -i debug` | 현재 Cilium 설정 중 `debug` 관련 설정만 필터링해서 확인합니다. |
<img width="1467" height="482" alt="image" src="https://github.com/user-attachments/assets/3c619b1d-04f5-4546-818b-71f783d87ac7" />

| 명령어 | 설명 |
|--------|------|
| `kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg config` | Cilium의 현재 에이전트 구성값(config) 확인 |
| `kubectl exec -n kube-system -c cilium-agent -it ds/cilium -- cilium-dbg status --verbose` | Cilium의 상태, 환경, API 지원 범위 등을 상세히 확인 |
<img width="1467" height="601" alt="image" src="https://github.com/user-attachments/assets/8c3843a8-47ad-4fdc-84fc-7f9683e2e752" />

### 3.4 네트워크 구성 확인

Cilium이 구성한 네트워크 설정을 확인합니다:
| 명령어 | 설명 |
|--------|------|
| `ip -c addr show lxc_health` | 현재 노드에서 Cilium의 health 체크용 인터페이스 `lxc_health`의 IP 주소 출력 |
| `sshpass -p 'vagrant' ssh vagrant@k8s-w1 ip -c addr show lxc_health` | k8s-w1 노드에서 `lxc_health` 인터페이스 IP 확인 |
| `sshpass -p 'vagrant' ssh vagrant@k8s-w2 ip -c addr show lxc_health` | k8s-w2 노드에서 `lxc_health` 인터페이스 IP 확인 |

| 명령어 | 설명 |
|--------|------|
| `kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium-dbg status --verbose` | Cilium 전체 상태 출력. BPF, Routing, IP 설정, kube-proxy 대체 여부 등 포함 |
| `kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium-dbg endpoint list \| grep health` | 등록된 엔드포인트 중 Health 관련 엔드포인트만 필터링 |
| `kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium-dbg status --all-addresses` | Cilium에서 관리하는 모든 IP 주소 상태 (노드/엔드포인트/인터페이스별) 출력 |
| `kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium bpf ct list global \| grep ICMP \| head -n4` | BPF 커넥션 추적 테이블 중 ICMP 프로토콜 관련 엔트리 상위 4개 확인 |
| `kubectl exec -it -n kube-system ds/cilium -c cilium-agent -- cilium bpf nat list \| grep ICMP \| head -n4` | BPF NAT 테이블 중 ICMP 프로토콜 관련 엔트리 상위 4개 확인 |

##### Routing
| 명령어 | 설명 |
|--------|------|
| `ip -c route | grep 172.20 | grep eth1` | 현재 노드의 라우팅 테이블에서 `172.20.x.x` 대역이 `eth1`을 통해 라우팅되는지 확인 (Cilium의 Native Routing 설정 결과 확인) |
| `for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i ip -c route | grep 172.20 | grep eth1 ; echo; done` | `k8s-w1`, `k8s-w2` 두 워커 노드에서 위와 동일한 Native Routing 설정 확인 반복 실행 |

| 명령어 | 설명 |
|--------|------|
| `kubectl get ciliumendpoints -A` | 모든 네임스페이스에 존재하는 Cilium 관리 Endpoint 목록 출력. Cilium이 관리하는 각 Pod의 IP, 상태, Identity 확인 가능 |
| `ip -c route | grep lxc` | 현재 노드에서 `lxc`로 시작하는 인터페이스(lxcX 등) 기반 라우팅 존재 여부 확인. 이는 `endpointRoutes.enabled=true` 설정 시 생성됨 |
| `for i in w1 w2 ; do echo ">> node : k8s-$i <<"; sshpass -p 'vagrant' ssh -o StrictHostKeyChecking=no vagrant@k8s-$i ip -c route | grep lxc ; echo; done` | `k8s-w1`, `k8s-w2` 워커 노드에서 lxc 인터페이스 기반 라우팅 설정 확인 반복 실행 |
<img width="1470" height="689" alt="image" src="https://github.com/user-attachments/assets/f7dc34d2-c8a4-4b1f-9baa-1c1cf620ff3d" />

##### Cilium 정보확인
| 명령어 | 설명 |
|--------|------|
| `kubectl get pod -owide` | 모든 파드의 IP 및 노드 위치 등 상세 정보 확인 |

| 명령어 | 설명 |
|--------|------|
| `kubectl get svc,ep webpod` | webpod의 서비스 및 엔드포인트 정보를 확인합니다. |
| `c0 map get cilium_ipcache` | Cilium의 IP 캐시를 확인합니다. (목적지 Pod 정보 포함) |
| `c0 map get cilium_ipcache \| grep $WEBPOD1IP` | 특정 웹 파드 IP에 대한 Cilium 캐시를 확인합니다. |
| `LXC=<k8s-ctr의 가장 나중에 lxc 이름>` | curl-pod이 사용하는 LXC 인터페이스를 식별합니다. |
| `c0bpf net show` | 노드의 eBPF 프로그램 전체 목록을 출력합니다. |
| `c0bpf net show \| grep $LXC` | 특정 LXC 인터페이스와 연결된 eBPF 프로그램을 확인합니다. |
| `c0bpf prog show id 1584` | 특정 eBPF 프로그램의 상세 정보를 확인합니다. (프로그램 ID 기준) |
| `c0bpf map list` | 전체 BPF 맵 목록을 확인합니다. |
<img width="1469" height="664" alt="image" src="https://github.com/user-attachments/assets/40dce98b-c3f6-45da-a120-9bdeb49b5216" />
<img width="1469" height="641" alt="image" src="https://github.com/user-attachments/assets/3b0b4001-960d-4e17-8b0a-1b2c64c15a51" />
<img width="736" height="174" alt="image" src="https://github.com/user-attachments/assets/e33828a3-f41b-4693-8663-83103fe7918b" />
 
##### 다른 노드간 pod->pod 통신
| 명령어 | 설명 |
|--------|------|
| `ngrep -tW byline -d eth1 '' 'tcp port 80'` | eth1 인터페이스에서 TCP 80 포트 트래픽을 실시간으로 캡처하여 줄 단위로 출력합니다. |
| `kubectl exec -it curl-pod -- curl $WEBPOD1IP` | curl-pod 안에서 webpod의 IP로 HTTP 요청을 수행합니다. |
<img width="1470" height="850" alt="image" src="https://github.com/user-attachments/assets/ab993ddb-e870-4e9c-a331-754f3f347754" />





