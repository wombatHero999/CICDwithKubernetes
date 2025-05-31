### CICD with Kubernetes 실습 구조(Kubernetes Cluster 구성: master node(1EA), worker nodes(2EA), ArgoCD 구성)

### Labs Server List
| Server Name        | Server Hostname    | Specs                           | IP Address     | Port Forwarding(ssh) | Port Forwarding(http) |
| ------------------ | ------------------ | ------------------------------- | -------------- | -------------------- | --------------------- |
| k8s-control        | k8s-control        | 2 vCPU, 4 GB RAM, 100GB Disk    | 192.168.15.101 |  25 -> 22            |  -                    | 
| worker-node-01     | worker-node-01     | 2 vCPU, 4 GB RAM, 100GB Disk    | 192.168.15.102 |  26 -> 22            |  -                    |
| worker-node-02     | worker-node-02     | 2 vCPU, 4 GB RAM, 100GB Disk    | 192.168.15.103 |  27 -> 22            |  -                    |
* 각각의 서버의 CPU는 2EA로 구성해야함. 1EA로 하면 실행 불가함. master node인 k8s-control은 실행시 느린 경우는 메모리를 8GB로 변경함. 다른 worker node도 Resource가 충분하면 8GB로 변환함.

### Visual Studio Code & VirtualBox

          VirtualBox 설치
          VirtualBox 명령으로 설치하는 방법(https://www.virtualbox.org/manual/, https://www.oracle.com/technical-resources/articles/it-infrastructure/admin-manage-vbox-cli.html)          
          
          VBoxManage setextradata global GUI/Input/HostKeyCombination 162,164
          VBoxManage natnetwork add --netname NatNetwork --network "192.168.15.0/24" --enable --dhcp off --port-forward-4 "ssh:tcp:[]:25:[192.168.15.101]:22"
          VBoxManage natnetwork modify --netname NatNetwork --port-forward-4 "ssh1:tcp:[]:26:[192.168.15.102]:22"
          VBoxManage natnetwork modify --netname NatNetwork --port-forward-4 "ssh2:tcp:[]:27:[192.168.15.103]:22"          
          
          Vboxmanage natnetwork list

          Visual Studio Code 설치
          Visual Studio Code 확장 설치    
          code --install-extension MS-CEINTL.vscode-language-pack-ko
          code --install-extension ms-vscode-remote.remote-ssh
          code --install-extension ms-azuretools.vscode-docker
          code --install-extension vscjava.vscode-java-pack
          code --install-extension vscjava.vscode-gradle
         함

### Kubernetes Install Guides 
          
          # 각각의 버전별 설치 가이드
          https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

### Configure Kubernetes IPv4 networking (all nodes)
          sudo nano   /etc/sysctl.d/k8s.conf
          # 밑에 3줄의 내용이 k8s.conf에 입력
          net.bridge.bridge-nf-call-iptables  = 1
          net.bridge.bridge-nf-call-ip6tables = 1
          net.ipv4.ip_forward   = 1
          # 위의 내용을 입력하고 다음 명령어 실행
          sudo sysctl --system

### Install Docker (on all nodes)
          sudo apt update
          sudo apt install docker.io -y # 설치해주거나 직접 설치할 수 있음
          # 도커 설치 경로 및 기타 설정
          https://docs.docker.com/engine/install/ubuntu/
          https://docs.docker.com/engine/install/linux-postinstall/

          sudo systemctl status docker
          sudo systemctl enable docker
          sudo mkdir /etc/containerd
          sudo sh -c "containerd config default > /etc/containerd/config.toml"
          sudo sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
          sudo systemctl restart containerd.service
          sudo systemctl status containerd.service

### Install Kubernetes components (on all nodes)
          sudo apt-get install curl ca-certificates apt-transport-https  -y
          curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
          echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
          sudo apt update
          sudo apt install kubelet kubeadm kubectl -y

### Initialize Kubernetes cluster (on master node)
          sudo kubeadm init --pod-network-cidr=10.10.0.0/16
          # 위의 문장 실행시 밑의 내용이 출력됨. 복사해서 사용
          
          mkdir -p $HOME/.kube
          sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
          sudo chown $(id -u):$(id -g) $HOME/.kube/config

### Install Calico network add-on plugin
          # https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart 설치 및 가이드
          kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
          curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml -O
          sed -i 's/cidr: 192\.168\.0\.0\/16/cidr: 10.10.0.0\/16/g' custom-resources.yaml
          kubectl create -f custom-resources.yaml
          kubectl get nodes
          kubectl get pods -A
          
### Add worker nodes to the cluster (on worker nodes)

          # worker nodes를 master node에 join시 접속 경로를 모르는 경우, 확인 방법
          kubeadm token create --print-join-command
          # 실행 명령어 형식
          kubeadm join [master node ip 주소를 확인해서 입력]:6443 --token xxxxxxxx \
        --discovery-token-ca-cert-hash xxxxxxxxxxxxxxxxxxxxxxxxxxxxx

### master node 초기화시
          
          sudo kubeadm reset
          sudo systemctl restart kubelet
          sudo reboot          
          
### Testing Kubernetes cluster
          kubectl create namespace demo-namespace
          kubectl create deployment my-app --image nginx --replicas 2 --namespace demo-namespace
          kubectl get deployment -n demo-namespace
          kubectl expose deployment my-app -n demo-namespace --type NodePort --port 80
          kubectl get svc -n demo-namespace
          curl http://<any-worker-IP>:node-port
          # 예) curl http://10.168.253.29:30115 

### ArgoCD 설치 및 사용방법
  
          https://argo-cd.readthedocs.io/en/stable/getting_started/ # 설치 가이드
  
          # 설치시
          kubectl create namespace argocd
          kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  
          # 설치시 설치 상태 확인
          kubectl get pods -n argocd -w
  
          # 설치후 접속 방법
          # 1. Service Type Load Balancer
          # kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
          # 2. Ingress
          # 3. Port Forwarding
          kubectl port-forward svc/argocd-server -n argocd 8080:443
  
          # 비밀번호를 확인 할때
          kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo
  
          # argocd-server에 접속해서 확인하기
          # kubectl exec -it -n argocd deployment/argocd-server -- /bin/bash
          # argocd login localhost:8080 -> admin/확인된 비밀번호(비밀번호를 확인 할때 나온 비밀번호)
          # argocd account update-password 비빌번호를 변경하고 싶음

          # default namespace를 argocd로 설정하고 싶을때
          # kubectl get pods -n argocd
          # kubectl config set-context --current --namespace=argocd
          # kubectl get pods -n argocd
          # kubectl get pods
          # kubectl config view --minify | grep namespace:
          # kubectl config set-context --current --namespace=default

  
          # 삭제하고 싶을때
          kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
          kubectl delete namespace argocd

### ArgoCD Rollouts 설치

          # https://argoproj.github.io/argo-rollouts/
          # https://github.com/argoproj/argo-rollouts

          # namespace 생성
          kubectl create namespace argo-rollouts
          
          # apply argo-rollouts
          kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
          
          # install kubectl plugin for argo rollout & 설치 확인
          curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
          sudo install -o root -g root -m 0755 kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
          kubectl argo rollouts version

### ArgoCD Rollouts 

          # 예제(github기반)
          https://github.com/dennislee-it

### Etc
- Docker install Ubuntu
 
          sudo apt update && sudo apt full-upgrade
          sudo apt install apt-transport-https ca-certificates curl 
          
          # Add Docker's official GPG key:
          sudo apt-get update
          sudo apt-get install ca-certificates curl
          sudo install -m 0755 -d /etc/apt/keyrings
          sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
          sudo chmod a+r /etc/apt/keyrings/docker.asc
          
          # Add the repository to Apt sources:
          echo \
            "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
            $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
            sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
          sudo apt-get update
          
          sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
          
          sudo docker run hello-world
          sudo groupadd docker
          sudo usermod -aG docker $USER
          => logout
          newgrp docker
          docker run hello-world

- Java(17)

          sudo apt update
          sudo apt install openjdk-17-jdk
          JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"
          java -version

- Git
 
          sudo apt install git
     
- Maven 

          sudo apt install maven

- Git config(Local)

          git config user.name "Dennis"
          git config user.email "itgenius1004@gmail.com"


- Git config(Global)

          git config --global user.name "Dennis"
          git config --global user.email "itgenius1004@gmail.com"

- Git Management

          git config --list
          git config --unset user.name
          git config --unset user.email

          git config --unset --global user.name
          git config --unset --global user.email

          git remote -v
          git push --force myapp-test
          
          git config credential.helper store
          git config credential.helper store --global
          
          git config --unset credential.helper
          git config --global --unset credential.helper


- Generate SSH Key
 
          ssh-keygen -t rsa -b 4096
          cd ~/.ssh
          cat id_rsa.pub >> ~/.ssh/authorized_keys
          cat authorized_keys
          cat id_rsa
          # Usage Visual Studio Code
          copy id_rsa on Host Windows(C:\Users\사용자\.ssh)

- Docker Security Issues

          // Security Issues 
          sudo chmod 666 /var/run/docker.sock or sudo chown root:docker /var/run/docker.sock
          sudo usermod -a -G docker jenkins

