# Instalação e Configuração do Kubernets

## Definições

* **kubelet**: Agente de comunicação entre os nós do cluster. Também é responsável pela inicialização dos `pods` e `containers`.
* **kubectl**: Ferramenta de linha de comando para implantar aplicativos, inspecionar e gerenciar recursos de cluster e visualizar logs.
* **kubeadmin**: Comando para inicializar o cluster.
* **kubernetes-cni**: Habilita a rede dentro dos containers, permitindo comunicação e troca de dados.
* **kube-flanel**: Fábrica de networks para os containeres.
* **minikube ou kind**: Ferramenta que permite executar o Kubernetes localmente. Executa um cluster Kubernetes de nó único em seu computador pessoal para fins de testes e desenvolvimento.

## Portas

* Porta Principal: `6443`

Veja outras [portas](https://kubernetes.io/docs/reference/ports-and-protocols/) que devem estar liberadas.

## Requerimentos Mínimos

* 2 CPUs e 2 GB ou mais de RAM por máquina.
* Nome da máquina na rede, endereço MAC e producy_uuid únicos para cada nó.

## Preparação e Instalação

**Execução**: Nó mestre e nó trabalhador.

Antes de iniciar vamos verificar se os recursos das máquinas são únicos na rede:

```shell
# verifica se o endereço mac é único
ip link
# ou
ipconfig -a

# verificase o product_uuid do servidor
sudo cat /sys/class/dmi/id/product_uuid
```

Os comandos a seguir prepararm o ambiente para a instalação do `Kubernetes`.

Primeira vamos atualizar a lista de pacotes com os repositórios do `Kubernets`.

```shell
# instala o pacote de transporte https e o curl
sudo apt install apt-transport-https curl

# adiciona a chave de assinatura do Kubernetes
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add

# adiciona o repositório Kubernetes como uma fonte de pacote
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" >> ~/kubernetes.list
sudo mv ~/kubernetes.list /etc/apt/sources.list.d

# atualiza a lista de pacotes
sudo apt update
```

O Kubernets não funciona em sistema que possuem memória de troca (swap). Assim é necessário desabilitá-la:

```shell
# desabilita a swap memory durante a sessão atual
sudo swapoff -a

# desabilita a swap memory na reinicialização
# comentar a linha da swap dentro do arquivo fstab
sudo nano /etc/fstab
```

É necessário ajustar o nome dos servidores do Kubernets para estes possuam identificação única dentro do cluster:

```shell
# ajusta nome do host
sudo hostnamectl set-hostname kubernetes-master
```

Em seguida, precisamos permitir que o `iptables` veja o tráfego dos interfaces de rede em modo `bridge`:

```shell
# verifica se o módulo br_netfilter está carregado
lsmod | grep br_netfilter

# caso não esteja, vamos carregá-lo
sudo modprobe br_netfilter

# configura o iptables para permitir o tráfego em bridge
sudo sysctl net.bridge.bridge-nf-call-iptables=1
```

Agora, vamos instalar o `docker`:

```shell
sudo apt install docker.io -y
```

Em seguida, vamos mudar o driver do `docker` de `cgroups` para `systemd`

> _Cgroupfs é uma função de opções para configurar uma LinuxFactory para retornar contêineres que usam a implementação nativa do sistema de arquivos cgroups para criar e gerenciar cgroups._

```shell
sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{ "exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts":
{ "max-size": "100m" },
"storage-driver": "overlay2"
}
EOF
```

Em seguida, execute os seguintes comandos para reiniciar e habilitar o Docker na inicialização do sistema:

```shell
# habilita o serviço docker na inicialização do sistema
sudo systemctl enable docker

# recarrega o daemon
sudo systemctl daemon-reload

# restart o serviço do docker
sudo systemctl restart docker
```

É necessário adicionar regras no firewall do servidor para permitir a comnunicação (caso o firewall esteja ativado):

```shell
sudo ufw allow 6443
sudo ufw allow 6443/tcp
```

Vamos configurar o IP do servidor:

```shell
sudo nano /etc/netplan/01-netcfg.yaml

# conteudo do arquivo
network:
  ethernets:
    ens33:
      addresses: [192.168.1.100/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [192.168.1.1, 8.8.8.8]
  version: 2

# verificar se a configuração está correta
sudo netplan try

# aceitar a configuração
[enter]
```

Por fim, vamos instalar as ferramentas que compõe o `Kubernetes`:

```shell
# instala as ferramentas do kubernetes
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni
```

## Inicialização do cluster

**Execução**: Nó mestre.

A primeira etapa de configuração do `cluster` é acionar o `nó mestre`:

```shell
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

_* O IP `10.244.0.0` é padrão para uso do `kube-flannel`._

Se tudo ocorrer bem, você deverá visualizar na saída o comando `kubeadmin join` com o token de acesso para adicionar os `nós filhos`. Guarde o `token` em um lugar para uso mais adiante.

Agora, é necessário copiar o arquivo de configuração para que o `kubectl` possa se comunicar com o cluster:

```shell
# cria a pasta home do kubernets para o usuário atual
mkdir -p $HOME/.kube

#  copioa o arquivo de configuração para a pasta do usuário atual
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# seta permissões para o arquivo de conmfiguração do usuário atual
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Implantação da rede de pods

**Execução**: Nó mestre.

A rede `pod` facilita a comunicação entre os servidores do `Kubernets`. `Flannel` é uma rede de sobreposição que atende aos requisitos do `Kubernetes`.

Após, vamos implantar a rede de `pod`:

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
```

Após, é necessário verificar se tudo ficou ativado:

```shell
# mostra o status dos serviços do kubernets
kubectl get pods --all-namespaces

# verifica integridade dos componentes
kubectl get componentstatus

# ou
kubectl get cs
```

Caso algum componente não esteja saudável, execute:

```shell
# edita a linha spec->containers->command contendo a linha - --port=0
sudo nano /etc/kubernetes/manifests/kube-scheduler.yaml
sudo nano /etc/kubernetes/manifests/kube-controller-manager.yaml

# reinicia o serviço do kubernets
sudo systemctl restart kubelet.service
```

## Adicionando os nós de trabalho ao cluster

**Execução**: Nó trabalhador.

Para adicionar um `nó de trabalho` ao clustes `Kubernets`, basta executar o comando capturado anteriormente:

```shell
# comando de exemplo
sudo kubeadm join 192.168.1.2:6443 --token u48s0y.0ckj645z0nfw15b9 \
        --discovery-token-ca-cert-hash sha256:f68b286f07c3b21bd5f850de18b0ae6222d5bfb1654fbf3667de7f6e69dc8710
```

Após, deve executar o comando abaixo no `nó mestre` para verificar se o `nó de trabalho` foi adicionado corretamente:

```shell
kubectl get nodes
```

## Adicionando um aplicativo/container de teste

Para implantar um serviço teste e verificar se o cluster está funcionando adequadamente execute:

```shell
# cria/implanta um serviço para o nginx
kubectl create deployment nginx --image=nginx

# visualiza os detalhes da implantação
kubectl describe deployment nginx

# torna o serviço acessível na rede externa pela porta 80
kubectl create service nodeport nginx --tcp=80:80
```

Após, verifique se o serviço está executando:

```shell
kubectl get svc
```

## Instalando o Dashboard

Para implantar os serviços do painel do Kubernetes, basta executar:

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml
```

Agora precisamos gerar o `token` de acesso ao `Dashboard`:

```shell
# cria a conta de serviço para o dashboard
kubectl create serviceaccount dashboard-admin-sa

# associa a conta de serviço ao grupo de cluster-admin
kubectl create clusterrolebinding dashboard-admin-sa  --clusterrole=cluster-admin --serviceaccount=default:dashboard-admin-sa

# verifica os segredos criados
kubectl get secrets

# captura o token de acesso
kubectl describe secret dashboard-admin-sa-token-<id>

# Token: 
 eyJhbGciOiJSUzI1NiIsImtpZCI6InRCc1o5U1hFQUxqNzZRTDVsLUZVQkx4cGhfS2IwUUhSMmdVTU9HU3dMNVkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRhc2hib2FyZC1hZG1pbi1zYS10b2tlbi05NW5qNSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tc2EiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJlNzU2ZTJkNS0yNzRjLTQ1MTEtYmExYS02MzdmZDE5MmUwNzQiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkYXNoYm9hcmQtYWRtaW4tc2EifQ.r_e0W529aOKbN58tq7iPAoUD1HpSDPaoDwkmievUg-3e9KH43oi-ElPr-jRRna76IWY7X7j6Sw3K2VrJgyVVSITapH6h1CEJTISJfM_kIhVSpZIrcId2e5m6j4Bl4AIju6troxqjwEMH8MOPuoZockFAGcWGmxUHrG_iPZQHKqi3W0979n-jbjLe6Qsegw-ZPmoJnGpPU8B6jAmi9Ulg26Tif_p5W5l8xTML2TlQd-Vbb8_J0nFeijxN3vh8Dl7ZZMi5hHXbBr1TUi0U5_gM3Wm_uvEjASvhHMnPTsHP9Q9KFXtATF3s5Y2ZKgb76XQ_L9hEgWESBqKXksAiyBU-eA
```

Guarde o `token` acima, este será utilizado mais tarde.

O acesso web ao Dashboard do Kubernetes deve ser realizado somente pelas máquinas dos administradores por questões de segurança. Assim, é necesário configurar o  `kubectl` na máquina `local` do administrador, para ter acesso ao Dashboard.

```shell
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update

sudo apt-get install -y kubectl
```

Agora, é necessário copiar o arquivo `/etc/kubernetes/admin.conf` do servidor `master` para a máquina local.

Em nosso caso, estamos usando uma distribução Linux em WSL2:

```shell
# cria a pasta home do kubernets para o usuário atual
mkdir -p $HOME/.kube

#  copioa o arquivo de configuração para a pasta do usuário atual
sudo cp -i /mnt/c/Temp/admin.conf $HOME/.kube/config
```

Teste a comunicação com o cluster:

```shell
kubectl get svc
```

Caso seja retornado os dados dos serviços do cluster, a conexão está funcionando.

Agora, é necessário habilitar o `proxy` local de acesso ao Dashboard usando o `kubectl`, executando o seguinte comando:

```shell
kubectl proxy
```

Agora, basta acessar o `Dashboard` pelo navegador:

```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

Será solicitado o `token` de acesso gerado anteriormente.

## Referências

* https://kubernetes.io/pt-br/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
* https://www.cloudsigma.com/how-to-install-and-use-kubernetes-on-ubuntu-20-04/
* https://www.cloudsigma.com/getting-to-know-kubernetes/
* https://kubernetes.io/docs/concepts/services-networking/service/
* https://kubernetes.io/docs/concepts/cluster-administration/networking/
* https://blog.laputa.io/kubernetes-flannel-networking-6a1cb1f8ec7c
* https://kubernetes.io/docs/reference/ports-and-protocols/
* https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
