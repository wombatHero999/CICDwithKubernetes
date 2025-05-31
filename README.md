### CICD-with-Kubernetes

### Labs Server List
| Server Name        | Server Hostname    | Specs                           | IP Address     | Port Forwarding(ssh) | Port Forwarding(http) |
| ------------------ | ------------------ | ------------------------------- | -------------- | -------------------- | --------------------- |
| k8s-control        | k8s-control        | 2 vCPU, 4 GB RAM, 100GB Disk    | 192.168.15.101 |  25 -> 22            |  -                    |
| worker-node-01     | worker-node-01     | 2 vCPU, 4 GB RAM, 100GB Disk    | 192.168.15.102 |  26 -> 22            |  -                    |
| worker-node-02     | worker-node-02     | 2 vCPU, 4 GB RAM, 100GB Disk    | 192.168.15.103 |  27 -> 22            |  -                    |

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
          code --install-extension vmware.vscode-boot-dev-pack
          code --install-extension vscode-kubernetes-tools
          code --install-extension vscode-docker
          Visual Studio Code Remote SSH로 접속해서 접속이 되는지 확인

### Ubuntu 64bit Server 22.04.x(Minimized) 설치 및 설정
- After installing ubuntu 64 server minimum specifications
- Create User => user1/1234
          
          sudo su
          apt-get install net-tools iputils-ping nano vim
          printf "Server Name(Each Server)" > /etc/hostname
          printf "\n192.168.15.101 k8s-control\n192.168.15.102 worker-node-01\n192.168.15.103 worker-node-02\n\n" >> /etc/hosts
          # 해당 터미널 세션 종류후 다시 접속
          cat /etc/hosts
          cat /etc/hostname
          cat /etc/netplan/50-cloud-init.yaml
          # ip를 수정하려면 50-cloud-init.yaml을 수정하기 위해서는 해당 파일을 생성하고 다음의 내용을 추가해야함
          nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
          network: {config: disabled}
          nano /etc/netplan/50-cloud-init.yaml
          => check ip or change ip
          netplan apply

### Disable swap space (all nodes) 스왑 항목을 off
          sudo swapoff -a
          # 재부팅 시 변경 사항을 유지하려면 /etc/fstab파일에 접근하여 스왑 항목 앞에 기호를 추가하여 주석 처리하세요 
          swapon --show

### Load Containerd modules (all nodes) 
          sudo modprobe overlay
          sudo modprobe br_netfilter
          sudo tee /etc/modules-load.d/k8s.conf <<EOF
          overlay
          br_netfilter
          EOF

### Configure Kubernetes IPv4 networking (all nodes)
          sudo nano   /etc/sysctl.d/k8s.conf
          net.bridge.bridge-nf-call-iptables  = 1
          net.bridge.bridge-nf-call-ip6tables = 1
          net.ipv4.ip_forward   = 1
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
          kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
          curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml -O
          sed -i 's/cidr: 192\.168\.0\.0\/16/cidr: 10.10.0.0\/16/g' custom-resources.yaml
          kubectl create -f custom-resources.yaml
          kubectl get nodes
          kubectl get pods -A
          
### Add worker nodes to the cluster (on worker nodes)
          kubeadm token create --print-join-command
          kubeadm join xx.xx.xx.xx:6443 --token xxxxxxxx \
        --discovery-token-ca-cert-hash xxxxxxxxxxxxxxxxxxxxxxxxxxxxx

### 초기화
          #################################
          # kubeadm 초기화
          $ sudo kubeadm reset
          $ sudo systemctl restart kubelet
          $ sudo reboot
          ##################################
          
### Testing Kubernetes cluster
          kubectl create namespace demo-namespace
          kubectl create deployment my-app --image nginx --replicas 2 --namespace demo-namespace
          kubectl get deployment -n demo-namespace
          kubectl expose deployment my-app -n demo-namespace --type NodePort --port 80
          kubectl get svc -n demo-namespace
          curl http://<any-worker-IP>:node-port
          # 예) curl http://10.168.253.29:30115 
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

- Etc

          kubectl create -f nginx-pod.xml
          kubectl get pods
          kubectl get describe pod nginx
          kubectl port-forward nginx 9000:80
          kubectl delete pod nginx
          
          kubectl apply -f declarative-deployment.yaml
          kubectl get deployments
          kubectl apply -f declarative-deployment.yaml
          kubectl diff -f declarative-deployment.yaml
          
          kubectl get deployment nginx-declarative -o=yaml

- ArgoCD
          https://argo-cd.readthedocs.io/en/stable/getting_started/ # 설치 가이드

          kubectl create namespace argocd
          kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
          
          kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo

          kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
          kubectl delete namespace argocd         


       
       
