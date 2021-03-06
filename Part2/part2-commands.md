### Agenda
In this session we will discuss briefly on the following topics:
- Quick Intro to Kubernetes
- Setting up Kubernetes cluster
- Running containers on Kubernetes
- Differences between Docker Swarm and Kubernetes
- Kubernetes Components and Architecture
- ConfigMaps, Secrets, Environment Variables, Services
- Namespaces, Pods, Pod Logs
- Tools and Shortcuts for Kubernetes


#### Installing Kubernetes
```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
sudo sysctl net/netfilter/nf_conntrack_max=393216
sudo usermod -aG docker $USER && newgrp docker
sudo swapoff -a
minikube start --nodes 2 --driver=docker
minikube addons enable metrics-server
minikube addons enable dashboard
kubectl get po -A
source <(kubectl completion bash) # setup autocomplete in bash into the current shell, bash-completion package should be installed first.
echo "source <(kubectl completion bash)" >> ~/.bashrc # add autocomplete permanently to your bash shell.
alias k=kubectl
complete -F __start_kubectl k
k get po -A
minikube stop
minikube start
```

#### Running Kubernetes Pods
```
k run --image=keshavprasad/shell-app:0.0.1 shellapp
```

#### Viewing Pod Information
```
k get pods
```

#### Exposing Pod and Accessing the Applications
```
k describe pod shellapp
```

#### Inspecting Pod Logs
```
k logs shellapp
k logs shellapp --tail 1 -f
```

#### Login to the Pods
```
k exec -it shellapp -- sh
ls
hostname
exit
```

#### Stop / Restart / Kill Pods
```
k delete pod shellapp
# Stop and Restart we will see in subsequent commands
```

#### Running Pods with Replicas
```
k create deployment --image=keshavprasad/shell-app:0.0.1 shellapp
k get deployments
k get pods
k delete pod POD_ID --grace-period 0 --force
k get pods
k scale deployment shellapp --replicas=2
k get pods
k scale deployment shellapp --replicas=0
k get pods
```

#### Inter Pod Communication
```
k run --image=keshavprasad/python-server:0.0.1 pyserver
k run --image=keshavprasad/go-server:0.0.1 goserver
k get pods -o wide
k exec -it goserver -- apk add curl --no-cache
k exec -it goserver -- curl PYSERVER_POD_IP:8000
k expose pod pyserver --port 8000 --target-port 8000
k get svc
k exec -it goserver -- curl pyserver:8000
```

#### Namespaces
```
k create namespace myns
k get ns
k run -n myns --image=keshavprasad/go-server:0.0.1 goserver-myns
k get pods -n myns -o wide
k exec -n myns -it goserver-myns -- apk add curl --no-cache
k exec -n myns -it goserver-myns -- curl --connect-timeout 1 pyserver:8000
k exec -n myns -it goserver-myns -- curl pyserver.default:8000
```

#### Environment Variables and Configmaps
```
k create -f shellapp-envs.yaml
k logs shellappenv --tail 10
k create configmap --from-file envfile envfile-cm
k get cm envfile-cm
k get cm envfile-cm -o yaml
k create -f shellapp-configmap-as-mount.yaml
k logs shellappcm --tail 10
k create configmap envfromliteral --from-literal=FIRSTNAME=Popeye --from-literal=LASTNAME=Sailor
k get cm envfromliteral -o yaml
k create -f shellapp-configmap-as-env.yaml
k logs shellappcmenv --tail 10
k delete -f shellapp-configmap-as-env.yaml
k delete cm envfromliteral
k create configmap --from-env-file envfile envfromliteral
k get cm envfromliteral -o yaml
k create -f shellapp-configmap-as-env.yaml
k logs shellappcmenv --tail 10
```

#### Secrets
```
docker build -t keshavprasad/shell-app:0.0.2 -f ./DockerfileShell .
minikube image load keshavprasad/shell-app:0.0.2
k create secret generic literalsecret --from-literal=SECRETNAME=SHHH!
k create secret generic secretenvfile --from-env-file=secret-env-file
k create secret generic secretfile --from-file secrets=secret-env-file
k run secretpod --image=keshavprasad/shell-app:0.0.2
k get pods -o wide --watch
k delete pod secretpod --force --grace-period 0
k run secretpod --image=keshavprasad/shell-app:0.0.2 --restart OnFailure
k create -f secretpod.yaml
k get secrets secretenvfile -o yaml
k get secrets literalsecret -o jsonpath='{.data.SECRETNAME}' | base64 -d && echo
```

#### Manifest File
```
ubuntu-pod.yaml shellapp-envs.yaml shellapp-configmap-as-mount.yaml are all called manifests files
```

#### Few good tools
```
krew - https://krew.sigs.k8s.io/
stern - https://github.com/wercker/stern
k9s - https://github.com/derailed/k9s
kubens and kubectx - https://github.com/ahmetb/kubectx#installation

(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
kubectl krew install ctx
kubectl krew install ns

git clone https://github.com/ahmetb/kubectx.git ~/.kubectx
COMPDIR=$(pkg-config --variable=completionsdir bash-completion)
ln -sf ~/.kubectx/completion/kubens.bash $COMPDIR/kubens
ln -sf ~/.kubectx/completion/kubectx.bash $COMPDIR/kubectx
# Fuzzy Search
sudo apt-get install -y fzf
cat << EOF >> ~/.bashrc

# Krew
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

#kubectx and kubens
export PATH=~/.kubectx:\$PATH
alias kns=kubens
alias kctx=kubectx
EOF

git clone https://github.com/jonmosco/kube-ps1.git
# Add the below to your bashrc

#kube-ps1
source /path/to/kube-ps1/kube-ps1.sh
PS1='[\u@\h \w $(kube_ps1)]\$ '

# Default Ubuntu Style with Kube PS1
PS1='\[\033[01;32m\]\u@\h:\[\033[01;34m\]\w\[\033[00m\] $(kube_ps1)]\$ '

kubeon
kubeoff

wget https://github.com/wercker/stern/releases/download/1.11.0/stern_linux_amd64
wget https://github.com/derailed/k9s/releases/download/v0.25.18/k9s_Linux_x86_64.tar.gz

mv stern_linux_amd64 stern
chmod +x stern
sudo mv stern /usr/local/bin/
stern shell --tail 1

tar -xf k9s_Linux_x86_64.tar.gz
sudo mv k9s /usr/local/bin/
k9s
```