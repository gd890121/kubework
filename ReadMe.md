# kubeadm, kubectl, kubelet 설치

control 

    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl
    
    sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

    echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

    sudo apt-get update
    sudo apt-get -y install kubeadm=[version]  kubectl=[version[ kubelet=[version]
    sudo apt-mark hold kubelet kubeadm kubectl

각각의 노드에도 같은방식으로 kubeadm, kubectl, kubelet설치  

# kubernetes cluster 구성

## - control plane  

    sudo kubeadm init --control-plane-endpoint 192.168.200.50 --pod-network-cidr 192.168.0.0/16 --apiserver-advertise-address [control plane의 IP]

-- init을 하고 나면 token값과 hash값이 나온다.
  
    mkdir -p $HOME/.kube  
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config  
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

-- sudo를 붙이지 않고 kubernetes명령어 사용가능(자격증명)

    kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

-- calico설치 네트워크 폴리시 적용

## - node
각각의 노드에 접근해서 실행한다.  

    sudo kubeadm join [control-plane IP]:6443 --token [token값] --discovery-token-ca-cert-hash sha256:[hash값]
-- cluter에 join시키는 명령

+) token값과 hash값을 모를경우 control plane 들어가서 찾을 수 있다.
- token  

        kubeadm token list
- hash  
        
      openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

# kubernetes version up

k8s 버전을 업그레이드 할때는 꼭 kubeadm부터 업그레이드 해줘야한다.  
컨트롤플레인에서 노드순서로 하면된다.   
업그레이드는 한번에 2단계 이상 건너뛰어서 할 수 없다.

## - control plane
- kubeadm

      apt-get update && apt-get install -y --allow-change-held-packages kubeadm=[upgrade version]  

      kubeadm version
      sudo kubeadm upgrade plan 

      sudo kubeadm upgrade apply [upgrade version( v1.19.x 처럼)]

 

- 그리고 kubelet과 kubectl upgrade  
                    
      apt-get update && apt-get install -y --allow-change-held-packages kubelet=[upgrade version] kubectl=[upgrade version]  

      sudo systemctl daemon-reload  
      sudo systemctl restart kubelet  

## - worker node

- kubeadm  

      apt-get update && apt-get install -y --allow-change-held-packages kubeadm=[upgrade version]
      
      sudo kubeadm upgrade plan  
      sudo kubeadm upgrade node  

- 그리고 kubelet과 kubectl upgrade  

      apt-get update && apt-get install -y --allow-change-held-packages kubelet=[upgrade version] kubectl=[upgrade version]    

      sudo systemctl daemon-reload
      sudo systemctl restart kubelet  

+) control plane에서 kube get nodes로 확인

# node 추가
node 생성 후에 컨트롤에서  
token값 생성 -> token값 확인, hash값 확인 후  
생성한 node에서 join

# node 삭제
삭제하려는 node에서  

    sudo kubeadm reset 

그 후에 관련 설정파일 모두 삭제  
    
    sudo rm -rf /etc/kubernetes/
    sudo rm -rf /etc/cni/net.d
    sudo rm -rf ~/.kube/  

    sudo apt remove kubeadm kubelet kubectl

control plane 가서  

    kubectl delete node [삭제할 node]

# 애드온(Metallb, Ingress, rook, Metrix-Server)
## - Metallb
control-plane에서

    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml

    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml

    vi config.yaml  # 밑에 파일 샘플참조

    kubectl apply -f config.yaml

    kubectl get all -n metallb-system  # system 상태확인

config.yaml 파일  

    apiVersion: v1
    kind: ConfigMap
    metadata:
      namespace: metallb-system
      name: config
    data:
      config: |
        address-pools:
        - name: default
          protocol: layer2
          addresses:
          - 192.168.200.200-192.168.200.210  # 예시일뿐 범위설정은 알아서하면된다.

## - Ingress
nginx 컨트롤러 
  
  control-plane에서  
    
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.47.0/deploy/static/provider/baremetal/deploy.yaml

    kubectl edit svc -n ingress-nginx ingress-nginx-controller

그러면 파일편집기가 열리는데
밑에 예시처럼 spec 아래에 externalIPs 추가해준다.  

    spec:
      externalIPs:
      - 192.168.200.51
      - 192.168.200.52
      - 192.168.200.53
    
  
  ## - rook-ceph

  rook-ceph는 전제로 아무것도 없는 빈 디스크가 필요하다.
(파일시스템도 없는 완전 빈 디스크)  
  
control-plane에서  

    git clone --single-branch --branch v1.6.7\
    https://github.com/rook/rook.git

    cd rook/cluster/examples/kubernetes/ceph
    
    kubectl create -f crds.yaml -f common.yaml -f operator.yaml
    
    kubectl create -f cluster.yaml   # worker node 3개 이상
    kubectl create -f cluster-test.yaml  # worker node 1개

    kubectl create -f csi/rbd/storageclass.yaml  # 블록스토리지

    kubectl create -f filesystem.yaml  # 파일스토리지 worker node 3개 이상 
    kubectl create -f filesystem-test.yaml  # worker node 1개

    kubectl create -f csi/cephfs/storageclass.yaml

    kubectl get sc # 상태확인
    kubectl -n rook-ceph get pod # 상태확인

  
    
## - Metrix-server
  
    wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    # yaml 파일 다운로드

    vi components.yaml  # 아래 예시처럼 수정

    kubectl create -f components.yaml

components.yaml 

    ```
    - args
      - --kubelet-insecure-tls   #args 아래에 추가
    ```


  
  



       


        
    





