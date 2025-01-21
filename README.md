# Configuração de Ambiente Kubernetes em EC2

Este arquivo fornece instruções e um script com comentários técnicos para configurar em uma instância EC2 um work node para Kubernetes: **kubelet**, **kubeadm** e **kubectl**.

Para mais informações detalhadas, consulte o [guia oficial do Kubernetes](https://v1-31.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

---

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


