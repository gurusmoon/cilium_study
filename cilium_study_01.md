


Window
실습환경 : Ultra 5 135U (12core) /  32GB

Vagrant설치
[Install | Vagrant | HashiCorp Developer](https://developer.hashicorp.com/vagrant/install#windows)
Virtual Box설치
[Downloads – Oracle VirtualBox](https://www.virtualbox.org/wiki/Downloads)


##### 프록시 해제
netsh winhttp reset proxy


Vagrant 설정 파일 다운로드
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/gasida/vagrant-lab/main/cilium-study/1w/Vagrantfile" -OutFile "Vagrantfile"

Vagrant Plungin 다운로드 
vagrant plugin install vagrant-proxyconf

Vagrant box 설치
vagrant box add bento/ubuntu-24.04 --provider virtualbox

Vagrant Proxy Settings
HTTP_PROXY = "http://121.134.28.20:3128"
HTTPS_PROXY = "http://121.134.28.20:3128"
NO_PROXY = "localhost,127.0.0.1,192.168.10.0/24"



Cilium 설치
