# ☸️ Kubernetes Infrastructure Automation Module (Ubuntu 22.04)

본 저장소는 전체 프로젝트의 **인프라 자동화 모듈**로, **Ansible**을 사용하여 **Ubuntu 22.04** 환경에 쿠버네티스(Kubernetes) 클러스터를 신속하게 프로비저닝하고 초기 설정을 완료하는 기능을 수행합니다.
<br>

## 📌 개요

이 모듈은 수동 설치의 번거로움을 제거하고, 코드 기반(IaC)으로 일관된 쿠버네티스 실습 환경을 구축하기 위해 제작되었습니다. 해당 모듈은 쿠버네티스에서 제공하는 **Calico CNI**를 설치하며, 다른 플러그인으로 대체 하고자 할 경우 06-install-calico.yaml 파일을 수정해야 합니다. 
<br>

## 🛠 멱등성 및 안정성 (Idempotency)

본 자동화 모듈의 모든 플레이북은 멱등성을 고려하여 설계되었습니다. 

- 동일한 플레이북을 여러 번 실행하더라도 시스템 상태를 확인한 후 필요한 변경사항만 적용합니다.
- 이미 설치가 완료된 노드에서 재실행 시 기존 설정을 파괴하지 않고 “OK” 상태를 유지하므로 네트워크 단절 등으로 중단되었을 경우에도 안심하고 다시 실행할 수 있습니다.
<br>

## 📂 디렉토리 구조 및 역할

```
.
├── task/
│   ├── 01-disable-swap.yaml            # k8s cluster 스왑 메모리 삭제 
│   ├── 02-sysctl-k8s-config.yaml       # Kernel 모듈 로드 및 네트워크 설정
│   ├── 03-setup-container-runtime.yaml # 마스터 초기화 및 조인 토큰 관리
|   ├── 04-k8s-binary-setup.yaml        # kubeadm, kubelet, kubectl 설치 
|   ├── 05-k8s-master-init.yaml         # 마스터 노드 초기화 
|   ├── 06-install-calico.yaml          # calico (CNI) 설치 
|   └── 07-k8s-worker-join.yaml         # 워커 노드 조인  
├── scripts/
│   └── install-docker.sh   # docker, containerd 설치 스크립트 
├── main.yaml               # task/ 전체 실행 파일 
└── README.md
```
<br>

## ❓실행 방법

이 프로젝트는 Ansible을 통해 전체 인프라 구축 과정을 자동화합니다. /scripts 내의 런타임 설치부터 쿠버네티스 클러스터 구성까지 아래 순서대로 플레이북을 실행하면 클러스터가 완성됩니다. 
<br>

### 1. 사전 준비

실행 전 모든 노드(Master, Workers)에 대해 다음 사항이 준비되어야 합니다. 

- **OS** : Ubuntu 22.04 LTS 권장
- **SSH Key 공유**: Ansible Control Node에서 모든 Managed Node(Master, Worker)로 `ssh-copy-id`가 완료되어야 합니다.
- **인벤토리 설정**: `/etc/ansible/hosts` 파일에 노드 IP를 그룹별로 설정합니다.
<br>

### 2. 클러스터 자동 구축

파일 번호 순서(01~07)에 따라 설정을 진행합니다. 

03-setup-container-runtime.yaml 실행 시 내부적으로 /scripts/install-docker.sh를 호출하여 Docker 및 Continer를 설치합니다. 따라서 파일 구조를 그대로 유지해야 합니다. 
<br>

### **2-1. 전체 일괄 실행**

```bash
ansible-playbook -i /etc/ansible/hosts main.yaml 
```
<br>

### **2-2. 단계별 개별 실행**

특정 단계의 로그를 확인하거나 개별적으로 실행하고 싶을 때 사용합니다. (개별 실행 권장)

**(1) OS 커널 및 환경 최적화** 
    
```bash
    ansible-playbook playbooks/01-disable-swap.yaml
    ansible-playbook playbooks/02-sysctl-k8s-config.yaml
```
    
- **실행 확인**
    
```bash
    # 1. 스왑 메모리 비활성화 확인 
    free -h  # Swap 항목이 0B인지 확인
    
    # 2. 커널 모듈 및 네트워크 설정 확인 
    lsmod | grep br_netfilter  # 모듈 로드 확인
    sysctl net.bridge.bridge-nf-call-iptables  # 값이 1인지 확인
```
<br>

**(2) 컨테이너 런타임 설치 (Script 연동)**
    
```bash
    # /scripts/install-docker.sh가 각 노드에서 자동 실행됨
    ansible-playbook playbooks/03-setup-container-runtime.yaml
```
    
- **실행 확인**
    
```bash
    # Docker/Containerd 서비스 상태 및 버전 확인 
    sudo systemctl status containerd  # 서비스 실행 중(Active) 확인
    docker version  # 또는 containerd --version 확인
```
<br>

**(3) k8s 바이너리 설치 및 마스터 초기화** 
    
```bash
    ansible-playbook playbooks/04-k8s-binary-setup.yaml
    ansible-playbook playbooks/05-k8s-master-init.yaml
```
    
- **실행 확인**
    
  ```bash
    # 1. 설치된 바이너리 버전 일치 확인 
    kubeadm version
    kubelet --version
    
    # 2. 쿠버네티스 Control Plane 확인 
    kubectl get pods -n kube-system  # apiserver, etcd, scheduler 등이 Running인지 확인
    ls /etc/kubernetes/admin.conf    # 관리자 설정 파일 생성 확인
  ```
<br>

**(4) 네트워크(CNI) 배포 및 워커 노드 조인** 
    
```bash
    ansible-playbook playbooks/06-install-calico.yaml
    ansible-playbook playbooks/07-k8s-worker-join.yaml
```
    
- **실행 확인**
    
```bash
    # 1. 네트워크 플러그인 파드 상태 확인
    kubectl get pods -n kube-system -l k8s-app=calico-node
    # 모든 노드 수만큼 calico-node 파드가 생성되고 Running 상태여야 함
    
    # 2. 클러스터 노드 리스트 및 역할 확인
    kubectl get nodes
    # Master 노드 외에 Worker 노드들이 추가되었는지, STATUS가 Ready인지 확인
```
<br>

### 3. 최종 상태 확인

모든 플레이북 실행 후 마스터 노드에서 다음 명령어로 클러스터 상태를 점검합니다. 

```bash
# 전체 노드의 상세 정보(IP, OS, 런타임 버전 등) 확인
kubectl get nodes -o wide 

# 모든 네임스페이스의 파드 정상 작동 여부 확인
kubectl get pods -A
```
