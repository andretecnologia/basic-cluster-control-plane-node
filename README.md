# Configuração de Ambiente Kubernetes em EC2

Este arquivo fornece instruções e um script com comentários técnicos para configurar uma instância EC2 com as ferramentas Kubernetes: **kubelet**, **kubeadm** e **kubectl**.

Para mais informações detalhadas, consulte o [guia oficial do Kubernetes](https://v1-31.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

---

## Instruções Técnicas

O script abaixo executa os seguintes passos:

1. **Atualizar a lista de pacotes**.
2. **Instalar pacotes necessários para gerenciar repositórios seguros e chaves GPG**.
3. **Adicionar a chave GPG do repositório Kubernetes**.
4. **Configurar o repositório oficial do Kubernetes**.
5. **Instalar as ferramentas kubelet, kubeadm e kubectl**.
6. **Proteger os pacotes contra atualizações automáticas**.
7. **Habilitar o serviço kubelet para iniciar automaticamente**.

### Código do Script

```bash
#!/bin/bash

# Configuração do ambiente para Kubernetes em EC2
# Este script deve ser executado como root ou com sudo.

# === Atualizar a lista de pacotes ===
# Garante que você tenha a versão mais recente dos pacotes disponíveis no repositório.
sudo apt-get update

# === Instalar pacotes necessários ===
# Os pacotes instalados permitem que o sistema utilize repositórios HTTPS, valide certificados
# e baixe chaves de repositório.
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# === Adicionar chave GPG do repositório Kubernetes ===
# A chave GPG garante a segurança do repositório e autentica os pacotes baixados.
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Caso o diretório para chaves não exista, crie-o com permissões adequadas:
# sudo mkdir -p -m 755 /etc/apt/keyrings

# === Configurar o repositório Kubernetes ===
# O repositório contém as versões estáveis das ferramentas kubelet, kubeadm e kubectl.
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# === Atualizar a lista de pacotes novamente ===
# Atualiza a lista de pacotes disponíveis com os novos repositórios adicionados.
sudo apt-get update

# === Instalar ferramentas do Kubernetes ===
# Instala o kubelet (executor de pods), kubeadm (inicializador de clusters)
# e kubectl (cliente CLI para interagir com o cluster).
sudo apt-get install -y kubelet kubeadm kubectl

# === Proteger pacotes contra atualizações automáticas ===
# Impede que esses pacotes sejam atualizados automaticamente para evitar incompatibilidades.
sudo apt-mark hold kubelet kubeadm kubectl

# === Habilitar e iniciar o serviço kubelet ===
# O kubelet é o serviço que gerencia a execução de pods e deve estar ativo e configurado para iniciar automaticamente.
sudo systemctl enable --now kubelet
```

# Parte 1

## Script de Configuração

O script abaixo realiza as seguintes ações:
1. **Desativa o swap**: Necessário para garantir a estabilidade do Kubernetes.
2. **Configura módulos do kernel**: Carrega módulos essenciais como `overlay` e `br_netfilter`.
3. **Ajusta parâmetros do sistema**: Configura roteamento de pacotes e parâmetros de rede.

### Código do Script

```bash
#!/bin/bash

# Configuração do ambiente para Kubernetes em EC2
# Este script deve ser executado como root ou usando sudo.

# === 1. Desativar o swap ===
# O Kubernetes exige que o swap seja desativado para evitar conflitos de alocação de recursos.
echo "=== Desativando o swap ==="
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
echo "Swap desativado e removido do /etc/fstab."

# === 2. Configurar módulos do kernel ===
# Os módulos overlay e br_netfilter são essenciais para redes de contêineres no Kubernetes.
echo "=== Configurando módulos do kernel ==="
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
echo "Módulos overlay e br_netfilter carregados."

# === 3. Configurar parâmetros de rede ===
# Configurações de roteamento e iptables para garantir a comunicação entre os pods e nós.
echo "=== Configurando parâmetros de rede ==="
cat <<EOF | tee /etc/sysctl.d/kubernetes.conf
net.ipv4.ip_forward = 1
net.ipv4.conf.all.forwarding = 1
net.ipv6.conf.all.forwarding = 1
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.conf.all.rp_filter = 0
net.ipv6.conf.all.rp_filter = 0
EOF

# Aplica as configurações definidas no arquivo acima
sysctl --system
echo "Parâmetros de rede aplicados."

# Conclusão
echo "=== Configuração concluída! ==="
```

# Parte 2

# Configuração do Containerd para Kubernetes

Este arquivo contém as instruções e os comandos necessários para configurar o **containerd**, um runtime de contêiner recomendado pelo Kubernetes. O **containerd** será configurado para funcionar com o gerenciador de cgroups do sistema, garantindo compatibilidade com o Kubernetes.

---

## Explicação Técnica

### O que é o Containerd?
- O **containerd** é um runtime de contêiner leve responsável por gerenciar o ciclo de vida dos contêineres:
  - Baixa imagens.
  - Gerencia a execução e o armazenamento de contêineres.
- É compatível com os padrões do OCI (Open Container Initiative) e recomendado pelo Kubernetes pela sua simplicidade e estabilidade.

### Por que realizar estas configurações?
- O Kubernetes exige que o **containerd** esteja configurado para usar o **SystemdCgroup**, garantindo compatibilidade com o gerenciamento de recursos do sistema.

### Descrição dos Comandos
1. **`apt install -y containerd`**:
   - Instala o **containerd** usando o gerenciador de pacotes `apt`.
   - A flag `-y` aceita automaticamente os prompts de confirmação.

2. **`mkdir -p /etc/containerd`**:
   - Cria o diretório de configuração do **containerd** caso ele não exista.

3. **`containerd config default > /etc/containerd/config.toml`**:
   - Gera o arquivo de configuração padrão do **containerd** e salva em `/etc/containerd/config.toml`.

4. **`sed -i 's/SystemdCgroup.*/SystemdCgroup = true/g' /etc/containerd/config.toml`**:
   - Altera o arquivo de configuração para habilitar o uso de **SystemdCgroup**, necessário para o Kubernetes.

5. **`sudo systemctl enable --now containerd`**:
   - Ativa o serviço **containerd** para iniciar automaticamente com o sistema e o inicia imediatamente.

6. **`sudo systemctl status containerd`**:
   - Exibe o status atual do serviço **containerd** para verificar se está funcionando corretamente.

---

# Parte 3

# Inicializando um Cluster Kubernetes com Kubeadm

Este arquivo fornece as instruções necessárias para inicializar um cluster Kubernetes utilizando o **kubeadm**. O comando `kubeadm init` é usado para configurar o nó principal do cluster (control plane).

---

## Explicação Técnica

### O que é o Kubeadm?
- O **kubeadm** é uma ferramenta oficial do Kubernetes que simplifica o processo de configuração de clusters.
- Ele configura os componentes essenciais do plano de controle (**control plane**), como:
  - **kube-apiserver**: Interface de comunicação com o cluster.
  - **etcd**: Banco de dados que armazena a configuração e o estado do cluster.
  - **kube-scheduler**: Gerencia a alocação de pods em nós disponíveis.
  - **kube-controller-manager**: Supervisiona os controladores que mantêm o estado desejado do cluster.

### O que o comando `kubeadm init` faz?
- Inicializa o plano de controle do Kubernetes no nó principal.
- Gera um token para que outros nós (workers) possam ingressar no cluster.
- Configura a comunicação segura entre os componentes do cluster.

---

## Pré-requisitos

1. **Runtime de contêiner instalado**: Certifique-se de que o **containerd** está configurado corretamente.
2. **Swap desativado**: O Kubernetes exige que o swap esteja desabilitado para evitar problemas de desempenho.
3. **Rede configurada**: Você precisará instalar um plugin de rede (como Calico, Flannel ou Cilium) após inicializar o cluster.

---

## Comando para Inicializar o Cluster

Para inicializar o cluster Kubernetes, use o seguinte comando:

```bash
sudo kubeadm init
```

# Explicando a Saída do Comando `kubeadm init`

Este documento detalha cada etapa do processo de inicialização de um cluster Kubernetes com o comando `kubeadm init`, explicando as mensagens exibidas na saída.

---

## Explicação das Mensagens de Saída

### `[init] Using Kubernetes version: v1.31.5`
- O **kubeadm** está configurado para usar a versão especificada do Kubernetes (`v1.31.5`).
- Isso define a versão dos componentes do cluster (API Server, Controller Manager, etcd, etc.).

---

### `[preflight] Running pre-flight checks`
- **Pre-flight checks** são verificações iniciais para garantir que o sistema atende aos requisitos mínimos para executar o Kubernetes.

#### `[preflight] Pulling images required for setting up a Kubernetes cluster`
- Baixa as imagens de contêiner necessárias para os componentes do cluster, como `kube-apiserver`, `kube-controller-manager` e `kube-scheduler`.

#### `[preflight] You can also perform this action beforehand using 'kubeadm config images pull'`
- Sugestão para economizar tempo: você pode baixar as imagens antecipadamente com o comando indicado.

#### `W0121 01:17:56.180546 detected that the sandbox image...`
- **Aviso:** A imagem do sandbox usada pelo runtime de contêiner é diferente da recomendada.
- Solução sugerida: Atualize para a imagem recomendada, `registry.k8s.io/pause:3.10`.

---

### `[certs] Using certificateDir folder "/etc/kubernetes/pki"`
- O kubeadm armazena os certificados gerados no diretório `/etc/kubernetes/pki`.

#### `[certs] Generating certificates and keys`
- Certificados são gerados para:
  - Autenticação entre componentes (por exemplo, `apiserver` e `etcd`).
  - Comunicação segura entre nós do cluster.

---

### `[kubeconfig] Writing kubeconfig files`
- Arquivos **kubeconfig** são gerados para autenticar componentes como:
  - `admin.conf`: Para o administrador do cluster.
  - `kubelet.conf`: Para o kubelet.
  - `controller-manager.conf` e `scheduler.conf`: Para os respectivos controladores.

---

### `[etcd] Creating static Pod manifest for local etcd`
- Cria o arquivo de manifesto para o **etcd**, o banco de dados distribuído que armazena o estado do cluster.

---

### `[control-plane] Creating static Pod manifests`
- Cria os manifestos para os componentes principais do Kubernetes:
  - `kube-apiserver`: Interface principal do Kubernetes.
  - `kube-controller-manager`: Gerencia controladores responsáveis por tarefas automáticas.
  - `kube-scheduler`: Agenda workloads nos nós do cluster.

---

### `[kubelet-start] Starting the kubelet`
- Configura o **kubelet** (agente que gerencia os nós) para executar como serviço e iniciar automaticamente.

#### `[kubelet-check] Waiting for a healthy kubelet`
- Verifica se o kubelet está funcionando corretamente.

#### `[api-check] Waiting for a healthy API server`
- Verifica se o **kube-apiserver** está funcionando corretamente.

---

### `[upload-config] Storing configuration in ConfigMaps`
- Armazena as configurações usadas durante a inicialização em ConfigMaps na namespace `kube-system`.

---

### `[mark-control-plane] Marking the node as control-plane`
- Marca o nó como **control-plane**, adicionando:
  - **Labels**: Identificam o nó como um nó de controle.
  - **Taints**: Impedem que workloads sejam agendados no nó de controle.

---

### `[bootstrap-token] Using token`
- Cria um **token de bootstrap**, usado para adicionar outros nós ao cluster com o comando `kubeadm join`.

---

### `[kubelet-finalize] Updating kubelet configuration`
- Atualiza a configuração do kubelet para usar certificados rotativos, garantindo segurança contínua.

---

### **Erro: `error execution phase addon/coredns`**
- Ocorreu um erro ao criar o serviço DNS central do cluster (**CoreDNS**).

#### **Causa**
- O kube-apiserver não estava acessível no endereço configurado (`172.31.25.35:6443`) devido a uma falha de conectividade.

#### **Possíveis Soluções**
1. Verifique se o kube-apiserver está em execução:
   ```bash
   kubectl get pods -n kube-system
    ```

# Verificando o Status do Cluster Kubernetes

Este documento descreve como iniciar o serviço do **kubelet**, configurar as credenciais para o `kubectl` e verificar os nós do cluster Kubernetes.

---

## 1. Iniciar o Serviço do Kubelet

### Comando:
```bash
systemctl start kubelet

export KUBECONFIG=/etc/kubernetes/admin.conf
```

### Explicação Técnica:
###O que é o admin.conf?

O admin.conf é o arquivo de configuração gerado após a execução do comando kubeadm init. Ele contém as credenciais necessárias e informações sobre o cluster para permitir que você se conecte a ele com o kubectl.
Por que configurar o KUBECONFIG?

O kubectl precisa saber onde buscar as credenciais e informações do cluster. Sem esse arquivo de configuração, o kubectl não conseguirá se comunicar com o cluster, e os comandos não funcionarão.
Comando export KUBECONFIG=/etc/kubernetes/admin.conf:

Esse comando define a variável de ambiente KUBECONFIG para apontar para o arquivo de configuração admin.conf. A partir desse momento, qualquer comando kubectl usará esse arquivo para se conectar ao cluster.


# Explicação do Comando `kubectl get po -A`

O comando `kubectl get po -A` é usado para listar todos os pods em todos os namespaces do cluster Kubernetes. Ele fornece uma visão geral de todos os pods, independentemente de onde estejam localizados no cluster. A flag `-A` significa "all namespaces" (todos os namespaces), o que permite que você veja os pods em todos os namespaces ao mesmo tempo.

---

## Saída Esperada

Aqui está um exemplo de saída do comando `kubectl get po -A`:

```plaintext
kube-system   coredns-7c65d6cfc9-cl2hk                  0/1     Pending   0               10m
kube-system   coredns-7c65d6cfc9-tdcmm                  0/1     Pending   0               10m
kube-system   etcd-ip-172-31-25-35                      1/1     Running   4 (3m19s ago)   9m1s
kube-system   kube-apiserver-ip-172-31-25-35            1/1     Running   3 (2m35s ago)   10m
kube-system   kube-controller-manager-ip-172-31-25-35   1/1     Running   5 (3m15s ago)   9m1s
kube-system   kube-scheduler-ip-172-31-25-35            1/1     Running   5 (3m15s ago)   9m1s
```

## Explicação da Saída
A saída mostra os pods que estão sendo executados em cada namespace, incluindo o namespace, o nome do pod, o número de réplicas disponíveis, o status do pod, o número de reinicializações e o tempo de execução. Vamos analisar cada coluna:

1. Namespace
Exemplo: kube-system
O namespace é uma forma de isolar e organizar os recursos dentro do Kubernetes. O namespace kube-system é onde o Kubernetes coloca os pods essenciais para a operação do cluster, como o controlador de rede e o servidor API.
2. Nome do Pod
Exemplo: coredns-7c65d6cfc9-cl2hk
O nome do pod identifica de forma única o pod dentro do namespace. O nome do pod é gerado automaticamente, mas inclui o nome do recurso que o gerou (por exemplo, o nome do deploy).
3. Status
Exemplo: Pending, Running
O status indica o estado atual do pod. Pode ter vários estados, incluindo:
Pending: O pod foi criado, mas não foi agendado para um nó ainda.
Running: O pod foi agendado para um nó e está em execução.
Outros status possíveis incluem Succeeded, Failed, e CrashLoopBackOff, entre outros.
4. Número de Réplicas Disponíveis
Exemplo: 0/1, 1/1
A primeira parte (0 ou 1) é o número de réplicas do pod que estão atualmente em execução.
A segunda parte (/1) é o número total de réplicas esperadas. Se a primeira parte não for igual à segunda, isso indica que o pod não está funcionando corretamente ou que não foi possível alocar os recursos necessários.
5. Reinicializações
Exemplo: 0, 4 (3m19s ago)
O número de reinicializações indica quantas vezes o pod foi reiniciado. Se esse número for alto, pode ser um sinal de que há problemas com o pod. Por exemplo, se o pod estiver falhando repetidamente, o número de reinicializações aumentará.
6. Tempo de Execução
Exemplo: 10m, 9m1s
O tempo de execução mostra quanto tempo se passou desde que o pod foi criado. Isso ajuda a entender a "idade" do pod no cluster.
Conclusão
A saída do comando kubectl get po -A fornece uma visão geral importante sobre o estado dos pods em todos os namespaces do cluster. Ao verificar o status e as reinicializações, você pode rapidamente identificar se há pods com problemas ou que precisam de atenção.

# Parte 4

## As instruções abaixo foram encontradas no link (opção Linux):
https://docs.cilium.io/en/v1.14/installation/k8s-install-kubeadm/#installation-using-kubeadm

**Linux/macOS/Other**

```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

To validate that Cilium has been properly installed, you can run:

```
cilium status --wait
```

Example output:

```
   /¯¯/¯¯\__/¯¯\    Cilium:         OK
\__/¯¯\__/    Operator:       OK
/¯¯\__/¯¯\    Hubble:         disabled
\__/¯¯\__/    ClusterMesh:    disabled
   \__/

DaemonSet         cilium             Desired: 2, Ready: 2/2, Available: 2/2
Deployment        cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
Containers:       cilium-operator    Running: 2
                  cilium             Running: 2
Image versions    cilium             quay.io/cilium/cilium:v1.9.5: 2
                  cilium-operator    quay.io/cilium/operator-generic:v1.9.5: 2
```

Run the following command to validate that your cluster has proper network connectivity:

```
cilium connectivity test
```

### Para Finalizar:

```
cilium install

kubectl get pods -A

```

# Componentes do Kubernetes no Namespace kube-system

O `kube-system` é o namespace do Kubernetes onde ficam os pods que executam os componentes essenciais do sistema. Abaixo, explicamos os principais componentes encontrados no comando `kubectl get pods -n kube-system`, que mostra o status dos pods no cluster.

## Componentes do Kubernetes no Namespace `kube-system`

### 1. **Cilium Envoy (`cilium-envoy-p8dgs`)**
   - **Função**: O pod `cilium-envoy` está relacionado à integração do Cilium com o proxy Envoy. O Cilium pode usar o Envoy para oferecer soluções de observabilidade e controle de tráfego de rede entre os pods, especialmente em um ambiente de serviço-mesh.
   - **Status**: Normalmente, deve estar "Running", indicando que o Cilium está integrado e funcionando corretamente.

### 2. **Cilium Operator (`cilium-operator-5c44ff484c-2lhhf`)**
   - **Função**: O `cilium-operator` é um componente central que gerencia a instalação, configuração e manutenção do Cilium no cluster Kubernetes. Ele também é responsável pela manipulação de objetos de rede, como políticas de rede e redes virtuais.
   - **Status**: O operador deve sempre estar ativo e funcionando para garantir que o Cilium esteja operando conforme o esperado.

### 3. **Cilium Pod (`cilium-pfdzs`)**
   - **Função**: Este pod contém os containers que executam o Cilium, uma solução de rede de alto desempenho baseada em eBPF para Kubernetes. Ele oferece funcionalidades como políticas de rede, serviços, roteamento de pacotes, e visibilidade de tráfego.
   - **Status**: O pod deve estar "Running", o que significa que o Cilium está ativo e monitorando o tráfego de rede entre os pods do cluster.

### 4. **CoreDNS (`coredns-7c65d6cfc9-cl2hk`, `coredns-7c65d6cfc9-tdcmm`)**
   - **Função**: O CoreDNS é o servidor DNS usado pelo Kubernetes para resolver nomes de serviços e pods dentro do cluster. Ele também pode ser configurado para resolver DNS fora do cluster.
   - **Status**: O CoreDNS deve estar em execução para que os serviços do Kubernetes se comuniquem corretamente entre si. Sem o CoreDNS, os pods não conseguirão localizar uns aos outros através de nomes de serviço.

### 5. **ETCD (`etcd-ip-172-31-25-35`)**
   - **Função**: O `etcd` é o banco de dados chave-valor altamente consistente que o Kubernetes usa para armazenar todas as informações do cluster, como configurações, estados de pods, e objetos do Kubernetes.
   - **Status**: O `etcd` deve estar ativo e em funcionamento para garantir que o Kubernetes possa acessar e modificar seu estado. Se o `etcd` falhar, o cluster Kubernetes pode ficar instável.

### 6. **Kube API Server (`kube-apiserver-ip-172-31-25-35`)**
   - **Função**: O `kube-apiserver` é o servidor central que expõe a API do Kubernetes. Ele é responsável por autenticar, autorizar e gerenciar as requisições feitas aos componentes do Kubernetes.
   - **Status**: Deve estar "Running" para que as interações com o cluster possam ser realizadas, seja através de `kubectl`, controladores ou outros serviços.

### 7. **Kube Controller Manager (`kube-controller-manager-ip-172-31-25-35`)**
   - **Função**: O `kube-controller-manager` é o componente responsável por gerenciar os controladores do Kubernetes, que são responsáveis por manter o estado desejado do cluster, como criar pods, implementar políticas e garantir que a quantidade de réplicas desejadas esteja em execução.
   - **Status**: O controlador deve estar em funcionamento contínuo para garantir que o cluster esteja sendo mantido corretamente.

### 8. **Kube Scheduler (`kube-scheduler-ip-172-31-25-35`)**
   - **Função**: O `kube-scheduler` é responsável por agendar os pods para serem executados em nodes do cluster. Ele decide qual node deve hospedar cada pod com base em recursos disponíveis e restrições de afinidade.
   - **Status**: O scheduler deve estar "Running", garantindo que os pods sejam corretamente distribuídos pelo cluster conforme a demanda e os recursos disponíveis.

## Resumo dos Componentes

- **Cilium**: Responsável pela rede e segurança de rede no Kubernetes, utilizando eBPF para fornecer visibilidade e políticas de rede.
- **CoreDNS**: Gerencia a resolução de DNS dentro do cluster, permitindo que os pods se comuniquem usando nomes de serviços.
- **ETCD**: Armazena os dados de configuração e estado do cluster.
- **Kube API Server**: A interface central do Kubernetes, que recebe e processa as requisições de controle do cluster.
- **Kube Controller Manager**: Gerencia os controladores que mantêm o estado desejado do cluster.
- **Kube Scheduler**: Responsável por agendar e alocar recursos para os pods no cluster.

Esses componentes são essenciais para garantir que o Kubernetes funcione corretamente e os serviços e pods dentro do cluster possam se comunicar e operar conforme o esperado.



