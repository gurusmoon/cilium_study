
#### 환경설정
Virtual Box 설치
<img width="1077" height="269" alt="image" src="https://github.com/user-attachments/assets/c6e4e599-95fd-49e5-b5e5-dac7f6f8d60c" />

Vagrant 설치
<img width="1072" height="241" alt="image" src="https://github.com/user-attachments/assets/7f8a4bf4-4d20-448e-b1fb-5a578893f3a5" />


### 실습 환경 개요
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

### 실습 구성 파일 정리

#### 1. `Vagrantfile`

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
#### VirtualBox
<img width="1009" height="506" alt="image" src="https://github.com/user-attachments/assets/60ce0daa-6a1b-4a47-b6be-0d68d856aa2a" />

#### 노드들 eth0 ip 확인
<img width="1079" height="268" alt="image" src="https://github.com/user-attachments/assets/49e48f2a-d95f-469b-9206-b2a193a7231f" />

#### k8s-ctl에서 기본정보 확인
<img width="451" height="214" alt="image" src="https://github.com/user-attachments/assets/99e9d932-85f1-45fe-a51c-157c2df02d3d" />
<img width="614" height="405" alt="image" src="https://github.com/user-attachments/assets/81a84007-dfde-4f52-b5d5-ed22b907a24a" />
<img width="777" height="85" alt="image" src="https://github.com/user-attachments/assets/54fe9e14-38af-42ea-9df3-d61d738e19ff" />
<img width="1012" height="437" alt="image" src="https://github.com/user-attachments/assets/9c21c93e-1e0c-4066-80c6-c935f8f7b8b3" />
<img width="836" height="184" alt="image" src="https://github.com/user-attachments/assets/f8736c1b-b6cb-45c4-b724-d1e2eb537db5" />

#### k8s-ctl에서 Kubernetes 정보 확인
<img width="1324" height="421" alt="image" src="https://github.com/user-attachments/assets/15b8ca14-cda3-4549-8213-2473617ecfee" />
<img width="1459" height="300" alt="image" src="https://github.com/user-attachments/assets/60e54a2b-bc5e-4973-82ac-6752930cc7cf" />
#### k8s-w1, k8s-w1에서 internal ip 변경설정
<img width="1360" height="325" alt="image" src="https://github.com/user-attachments/assets/4d9f76db-3fa4-441f-9ce1-702071cff720" />















