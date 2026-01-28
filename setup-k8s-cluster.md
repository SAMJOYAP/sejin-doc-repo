# Setup k8s cluster

# Kubespray로 Kubernetes 클러스터 구성: 실행 커맨드별 설명

---

## 1) Kubespray 내려받기 & 인벤토리 준비

### 패키지 업데이트 / 필수 도구 설치

```bash
sudo apt update
sudo apt install -y git
```

- APT 패키지 목록을 최신으로 갱신. 설치/업데이트 전에 거의 항상 실행.
- Kubespray 소스를 GitHub에서 받기 위해 git 설치.
- `y`: 설치 중간 확인 질문을 자동으로 “yes” 처리.

---

### Kubespray 클론

```bash
git clone --branch=release-2.27 https://github.com/kubernetes-sigs/kubespray.git
```

- Kubespray 레포를 로컬로 다운로드.
- `-branch=release-2.27`: 특정 릴리즈 브랜치를 고정해(재현성/안정성 확보).

---

### 샘플 인벤토리 복사 → 내 클러스터 인벤토리 생성

```bash
cd kubespray/
cp -r inventory/sample/ inventory/mykube
```

- `sample`을 `mykube`로 복사해서 **내 클러스터 설정 전용 폴더**를 만든다.
- `r`: 디렉토리(하위 파일 포함) 재귀 복사.

---

### 인벤토리/설정 파일 편집

```bash
vim inventory/mykube/inventory.ini

# inventory.ini
[kube_control_plane]
kube-control1   ansible_host=192.168.76.11 ip=192.168.76.11 ansible_connection=local

[etcd:children]
kube_control_plane

[kube_node]
kube-node1      ansible_host=192.168.76.21 ip=192.168.76.21
kube-node2      ansible_host=192.168.76.22 ip=192.168.76.22
kube-node3      ansible_host=192.168.76.23 ip=192.168.76.23
```

- 어떤 노드가 control plane인지, worker인지, etcd인지 등을 정의하는 핵심 파일.
- 보통 IP/호스트명, 그룹(`kube_control_plane`, `kube_node`, `etcd` 등)이 여기 들어감.

```bash
vim inventory/mykube/group_vars/k8s_cluster/addons.yml

# addons.yml
helm_enabled: true
metrics_server_enabled: true
ingress_nginx_enabled: true
metallb_enabled: true

metallb_config:
  address_pools:
    primary:
      ip_range:
        - 192.168.76.200-192.168.76.210
      auto_assign: true
  layer2:
    - primary
			-> 주석 제거 및 주소 값 수정
```

- 클러스터에 추가로 설치할 애드온 설정(예: ingress controller, metrics-server 등).
- “어떤 부가 기능을 켤지/말지”를 결정.
- 변경부분 한줄 설명
  - **helm_enabled: true**
    → Kubernetes 리소스를 패키지(Chart) 단위로 관리하기 위해 Helm을 설치한다.
  - **metrics_server_enabled: true**
    → 노드·파드의 CPU/메모리 사용량을 수집하여 kubectl top 및 HPA가 가능하도록 한다.
  - **ingress_nginx_enabled: true**
    → 외부 HTTP/HTTPS 요청을 받아 내부 Service로 라우팅하기 위한 Nginx Ingress Controller를 설치한다.
  - **metallb_enabled: true**
    → 온프레미스 환경에서 LoadBalancer 타입 Service에 외부 IP를 할당하기 위해 MetalLB를 설치한다.
  - **metallb_config.ip_range**
    → MetalLB가 LoadBalancer Service에 자동 할당할 외부 IP 범위를 정의한다.
  - **metallb_config.auto_assign: true**
    → LoadBalancer Service 생성 시 IP를 자동으로 할당하도록 설정한다.
  - **metallb_config.layer2**
    → BGP 없이 ARP 기반 Layer2 모드로 MetalLB를 동작시킨다.

```bash
vim inventory/mykube/group_vars/k8s_cluster/k8s-cluster.yml

#k8s-cluster.yml
kube_proxy_strict_arp: true
```

- Kubernetes 버전, 네트워크(CNI), kube-proxy 모드 등 **클러스터 전역 설정**을 조정하는 파일.
- 변경부분 한줄 설명
  - **kube_proxy_strict_arp: true**
    → MetalLB Layer2 모드에서 LoadBalancer IP에 대해 **해당 노드만 ARP 응답**하도록 kube-proxy 동작을 제한한다.

---

## 2) SSH 키 기반 접속 준비 (Ansible가 노드에 접속해야 함)

### SSH 키 생성

```bash
ssh-keygen
```

- 현재 머신(Ansible 실행 머신)에서 SSH 키 페어 생성.
- Ansible이 각 노드에 비밀번호 없이 접속하기 위해 필요.

### Worker 노드들에 공개키 배포

```bash
ssh-copy-id vagrant@192.168.76.21
ssh-copy-id vagrant@192.168.76.22
ssh-copy-id vagrant@192.168.76.23
```

- 대상 노드의 `~/.ssh/authorized_keys`에 내 공개키를 넣어준다.
- 이후 `ssh vagrant@IP`가 비밀번호 없이 동작 → Ansible 자동화가 가능.

---

## 3) Python 가상환경 & 의존성 설치 (Kubespray/Ansible 실행 환경 만들기)

```bash
sudo apt install -y python3 python3-pip python3-venv
```

- Python 실행/패키지 설치(pip)/가상환경(venv) 생성에 필요한 패키지 설치.

```bash
python3 -m venv ~/kubespray/
```

- 홈 디렉토리에 Kubespray용 Python 가상환경 생성.
- 시스템 파이썬을 더럽히지 않고(버전 충돌 방지) Kubespray 요구 패키지를 격리 설치.

```bash
source ~/kubespray/bin/activate
```

- 현재 터미널 세션에서 가상환경 활성화.
- 이후 `pip install ...`이 이 가상환경에 설치됨.

---

### 요구 패키지 설치

```bash
pip install -r requirements.txt
```

- Kubespray가 요구하는 Ansible/라이브러리 버전을 `requirements.txt`로 제공.
- `pip install -r ...`: 파일에 적힌 패키지들을 한 번에 설치.

---

## 4) Ansible 연결 확인(ping) → 실제 클러스터 배포 실행

### 인벤토리 명시 ping (기본 동작 확인용)

```bash
ansible -m ping all -i inventory/mykube/inventory.ini
```

- 내가 만든 인벤토리 기준으로 모든 노드에 SSH 접속/권한/파이썬 실행이 되는지 확인.

---

### Kubespray 실행(클러스터 설치)

```bash
ansible-playbook -i inventory/mykube/inventory.ini cluster.yml -b
```

- Kubespray의 핵심 플레이북 실행: Kubernetes 클러스터를 “설치/구성”한다.
- `i ...`: 내 인벤토리 사용
- `cluster.yml`: 설치 메인 플레이북
- `b`: become(sudo) 권한으로 실행 (노드에 root 권한 작업 필요)

> 이 단계에서 Kubespray가 내부적으로 수행:

- container runtime(containerd 등) 설치/설정
- kubelet 설치/설정
- kubeadm 기반 control plane/bootstrap
- 워커 노드 join
- CNI 설치
- kube-system 컴포넌트 구성

---

## 5) kubectl 편의 설정 + 관리자 kubeconfig 설정

### kubectl 자동완성

```bash
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl
```

- bash에서 `kubectl get ...` 자동완성 기능을 활성화하기 위한 스크립트 설치.
- `tee`: root 권한으로 파일에 출력 저장.

---

### 현재 사용자(vagrant)에서 kubectl 사용하도록 kubeconfig 복사

```bash
mkdir ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown vagrant:vagrant ~/.kube/config
```

- `~/.kube/config`가 있으면 기본적으로 `kubectl`이 이 파일을 사용.
- `cp`: 관리자 kubeconfig를 사용자 홈으로 복사
- `chown`: 소유권을 사용자(vagrant)로 바꿔서 `kubectl` 실행 시 권한 문제 방지

---

### kubectl 동작 확인

```bash
kubectl version
```

- kubeconfig가 잘 잡혔는지(클러스터 연결이 되는지) 확인.

---

## 6) kube-system 핵심 컴포넌트 확인 (Control Plane/Proxy 등)

```bash
kubectl get pod -n kube-system | grep api
```

- `kube-apiserver` 파드가 떠 있는지 확인.
- apiserver는 클러스터 “API 관문”이므로 가장 핵심.

```bash
kubectl get pod -n kube-system | grep sch
```

- `kube-scheduler` 파드 확인.
- 파드를 어느 노드에 배치할지 스케줄링하는 구성요소.

```bash
kubectl get pod -n kube-system | grep controller
```

- `kube-controller-manager` 파드 확인.
- ReplicaSet/Node/Endpoints 등 여러 컨트롤 루프를 돌며 “원하는 상태”로 맞춘다.

```bash
kubectl get pod -n kube-system | grep kube-proxy
```

- 각 노드의 `kube-proxy` 파드(또는 데몬셋) 확인.
- Service(ClusterIP/NodePort 등) 트래픽 라우팅 규칙을 노드에 설정.

---

## 7) 노드 서비스 상태 확인 (kubelet / containerd)

### Control plane에서 로컬 서비스 상태

```bash
sudo systemctl status kubelet
sudo systemctl status containerd
```

- `kubelet`: 노드에서 파드를 실제로 실행/관리하는 에이전트
- `containerd`: 컨테이너 런타임(이미지 pull, 컨테이너 실행 담당)

---

### SSH로 각 워커 노드 kubelet 상태 확인 (원격 점검)

```bash
ssh vagrant@192.168.76.21 sudo systemctl is-active kubelet
ssh vagrant@192.168.76.22 sudo systemctl is-active kubelet
ssh vagrant@192.168.76.23 sudo systemctl is-active kubelet
```

- 각 워커 노드에 접속해서 kubelet이 **active**인지 확인.
- `is-active`: `active`/`inactive` 같은 상태만 간단히 출력

---

# 빠른 요약

- Kubespray를 `release-2.27`로 고정해 클러스터 구성의 재현성을 확보했다.
- inventory.ini와 group_vars를 통해 노드 역할과 클러스터 전역 설정(CNI/애드온)을 정의했다.
- Ansible이 노드에 접근할 수 있도록 SSH 키 기반 접속을 구성했다.
- Python venv에 requirements를 설치하여 Kubespray 실행 환경을 격리했다.
- `ansible-playbook cluster.yml -b`로 클러스터를 배포하고,
  `kubectl get pods -n kube-system` 및 `systemctl status`로 핵심 컴포넌트 동작을 검증했다.

---
