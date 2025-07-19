
### 환경설정
Virtual Box 설치
<img width="1077" height="269" alt="image" src="https://github.com/user-attachments/assets/c6e4e599-95fd-49e5-b5e5-dac7f6f8d60c" />

Vagrant 설치
<img width="1072" height="241" alt="image" src="https://github.com/user-attachments/assets/7f8a4bf4-4d20-448e-b1fb-5a578893f3a5" />


## 실습 환경 개요
<img width="1750" height="614" alt="image" src="https://github.com/user-attachments/assets/1424a85d-0aad-405f-b02d-2de2ee0c59c2" />

* **기본 구성**:

  * **Control Plane**: `k8s-ctr` (IP: `192.168.10.100`)
  * **Worker 1**: `k8s-w1` (IP: `192.168.10.101`)
  * **Worker 2**: `k8s-w2` (IP: `192.168.10.102`)

* **공통사항**:

  * `eth0` (NAT): `10.0.2.15` (모든 노드 동일)
  * `eth1` (Private network): 고정 IP 할당
  * Ubuntu 24.04 베이스 이미지 사용
  * `kubeadm`으로 초기화 및 워커 조인 완료
  * **CNI 미설치 상태**

---

## 📂 실습 구성 파일 정리

### 1. `Vagrantfile`

* VM 정의 및 초기 설치 자동화
* 변수로 쿠버네티스/컨테이너 버전 지정
* Control Plane 및 Worker 노드 자동 생성

``` 
K8SV = '1.33.2-1.1'       # Kubernetes Version
CONTAINERDV = '1.7.27-1'  # Containerd Version
N = 2                     # Worker 노드 수

BOX_IMAGE = "bento/ubuntu-24.04"
BOX_VERSION = "202502.21.0"
```

<img width="1009" height="506" alt="image" src="https://github.com/user-attachments/assets/60ce0daa-6a1b-4a47-b6be-0d68d856aa2a" />





