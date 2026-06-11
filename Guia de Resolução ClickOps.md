# Guia de Resolução ClickOps

Este manual detalha o passo a passo para resolver o cenário da JuseByte operando exclusivamente pelo Console de Gerenciamento da AWS. [cite_start]O objetivo é redesenhar a camada de computação com provisionamento automatizado de instâncias e armazenamento compartilhado[cite: 2, 3, 4, 9].

---

## Passo 1: Configurar a Função do IAM (Permissões sem credenciais)
[cite_start]Para que a instância baixe o pacote da aplicação sem chaves de acesso embutidas no código[cite: 41, 42], crie um perfil de instância:

1. Acesse o painel **IAM** e vá em **Roles** (Funções) > **Create role**.
2. Escolha **AWS service** e selecione **EC2**.
3. Na aba de políticas, pesquise e anexe a política `AmazonS3ReadOnlyAccess`.
4. Nomeie a função como `JuseByte-EC2-Role` e clique em **Create role**.

---

## Passo 2: Criar os Security Groups (Controle de Rede)
[cite_start]Você precisa isolar o tráfego em camadas, seguindo o princípio do menor privilégio[cite: 40]. Acesse **EC2 > Security Groups** e crie três grupos na sua VPC:

* **SG-ALB (Balanceador):**
  * **Inbound:** HTTP (Porta 80) / Origem: `0.0.0.0/0` (Internet).
* **SG-EC2 (Instâncias da Aplicação):**
  * **Inbound:** HTTP (Porta 80) / Origem: Selecione o `SG-ALB`. [cite_start]*(Isso garante que as instâncias não recebam tráfego direto da internet, apenas do balanceador)*[cite: 35, 36].
* **SG-EFS (Armazenamento Compartilhado):**
  * **Inbound:** NFS (Porta 2049) / Origem: Selecione o `SG-EC2`. [cite_start]*(A porta NFS deve ser liberada para a montagem funcionar)*[cite: 39].

---

## Passo 3: Provisionar o Amazon EFS (Armazenamento Compartilhado)
[cite_start]O armazenamento precisa permitir leitura e escrita simultânea por todas as instâncias no ambiente Linux[cite: 27, 37]. 

1. Acesse **EFS** > **Create file system** > Clique em **Customize**.
2. [cite_start]Habilite a **Encryption of data at rest** (Criptografia em repouso)[cite: 38].
3. [cite_start]Na etapa de rede (Network), para cada zona de disponibilidade nos destinos de montagem (Mount targets), remova o Security Group padrão e adicione o `SG-EFS`[cite: 39].
4. Conclua a criação e **anote o ID do File System** (ex: `fs-0123456789abcdef`).

> **Nota:** Opções como EBS (anexado a uma única instância) não atendem a esse cenário de acessos simultâneos. [cite_start]Caso o ambiente fosse Windows, a alternativa correta seria o Amazon FSx[cite: 28].

---

## Passo 4: Criar o Target Group e o Application Load Balancer
[cite_start]O balanceador distribuirá o tráfego de entrada apenas para as instâncias saudáveis[cite: 15, 24, 25].

1. **Target Group:**
   * Acesse **EC2 > Target Groups** > **Create target group**.
   * Escolha **Instances**, protocolo HTTP, nomeie como `JuseByte-TG`.
   * No *Health checks*, mantenha o caminho padrão (`/`). Crie o grupo.
2. **Load Balancer:**
   * Acesse **EC2 > Load Balancers** > **Create Load Balancer** > **Application Load Balancer**.
   * Nome: `JuseByte-ALB`. Esquema: **Internet-facing**.
   * Mapeamento de Rede: Selecione sua VPC e pelo menos duas Zonas de Disponibilidade.
   * Security Groups: Selecione apenas o `SG-ALB`.
   * Listeners: Na porta 80, selecione o `JuseByte-TG` como destino. Crie o ALB.

---

## Passo 5: Configurar o Launch Template com User Data
[cite_start]Toda instância nova deve subir com a aplicação implantada e o disco montado sem nenhuma configuração manual[cite: 31, 43].

1. Acesse **EC2 > Launch Templates** > **Create launch template**.
2. Nomeie como `JuseByte-Template`.
3. **AMI:** Escolha o Amazon Linux 2023.
4. **Instance Type:** Escolha o tipo de instância (ex: `t2.micro` para x86 ou `t4g.micro` para ARM).
5. **Network settings:** Em Security Groups, selecione o `SG-EC2`.
6. **Advanced details:**
   * Em **IAM instance profile**, selecione a `JuseByte-EC2-Role`.
   * Na caixa **User data**, insira o script abaixo (substitua `<SEU_EFS_ID>` pelo ID anotado no Passo 3):

```bash
#!/bin/bash
# 1. Atualizar e instalar utilitarios
yum update -y
yum install -y amazon-efs-utils unzip wget

# 2. Montar o armazenamento EFS com criptografia em transito (tls)
mkdir -p /mnt/uploads
mount -t efs -o tls <SEU_EFS_ID>:/ /mnt/uploads

# 3. Preparar o diretorio da aplicacao
mkdir -p /opt/jusebyte
cd /opt/jusebyte

# 4. Baixar a aplicacao do S3 e descompactar
wget [https://worldskills.s3.us-east-1.amazonaws.com/br-pr-2026/lab2/site.zip](https://worldskills.s3.us-east-1.amazonaws.com/br-pr-2026/lab2/site.zip)
unzip -o site.zip

# 5. Descobrir a arquitetura dinamicamente
ARCH=$(uname -m)
if [ "$ARCH" == "x86_64" ]; then
    BIN="site-linux-amd64"
elif [ "$ARCH" == "aarch64" ]; then
    BIN="site-linux-arm64"
fi

# 6. Conceder permissao de execucao e iniciar o servidor web
chmod +x ./$BIN
nohup ./$BIN > jusebyte.log 2>&1 &
