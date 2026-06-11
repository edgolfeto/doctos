# Guia Definitivo: Laboratório JuseByte (WorldSkills) - Do Zero ao Deploy

Este é o seu manual completo e detalhado para resolver o laboratório de infraestrutura da JuseByte. Vamos construir a solução do absoluto zero, passo a passo, diretamente no Console da AWS, garantindo que você entenda o **porquê** de cada configuração.

O objetivo é criar uma arquitetura onde um balanceador de carga recebe o tráfego da internet e o distribui para instâncias EC2. Essas instâncias compartilharão um disco de rede (EFS) para uploads de arquivos e serão configuradas de forma 100% automatizada.

---

## Passo 1: Permissões Seguras (IAM Role)

**O Problema:** O laboratório exige que a instância baixe o código do S3 sem usar credenciais ou chaves de acesso embutidas no script de inicialização.
**A Solução:** Criar uma "Função" (Role) que a instância "vestirá" para ter permissão de ler o S3 nativamente.

1. No console da AWS, busque por **IAM** e acesse o painel.
2. No menu lateral esquerdo, clique em **Roles** (Funções) e depois no botão **Create role** (Criar função).
3. Em "Trusted entity type", escolha **AWS service**.
4. Em "Use case", escolha **EC2** e clique em **Next**.
5. Na tela de permissões, pesquise por `AmazonS3ReadOnlyAccess`. Marque a caixa ao lado dessa política e clique em **Next**.
6. Em "Role name", digite `JuseByte-EC2-Role`.
7. Clique em **Create role** no final da página.

---

## Passo 2: Isolamento de Rede (Security Groups)

**O Problema:** Precisamos garantir que ninguém acesse os servidores ou o banco de dados/arquivos diretamente pela internet, aplicando a regra do "menor privilégio".
**A Solução:** Criar três Security Groups (firewalls virtuais) em cascata.

1. Busque por **EC2** e acesse o painel. No menu lateral, em *Network & Security*, clique em **Security Groups** > **Create security group**.

**2.1. Criando o SG do Balanceador (A porta de entrada):**
*   **Nome:** `SG-ALB`
*   **Descrição:** Permite acesso HTTP da internet.
*   **Inbound rules (Regras de entrada):** Adicione uma regra do tipo **HTTP** (Porta 80), e em *Source* (Origem) escolha **Anywhere-IPv4** (`0.0.0.0/0`).
*   Clique em **Create security group**.

**2.2. Criando o SG das Instâncias EC2 (A aplicação):**
*   **Nome:** `SG-EC2`
*   **Descrição:** Recebe tráfego apenas do Balanceador.
*   **Inbound rules:** Adicione uma regra do tipo **HTTP** (Porta 80). Em *Source*, clique na caixa de busca e **selecione o `SG-ALB`** que você acabou de criar.
*   Clique em **Create security group**.

**2.3. Criando o SG do Armazenamento (O disco compartilhado):**
*   **Nome:** `SG-EFS`
*   **Descrição:** Permite a montagem do disco apenas pelas instâncias[cite: 1].
*   **Inbound rules:** Adicione uma regra do tipo **NFS** (Porta 2049)[cite: 1]. Em *Source*, **selecione o `SG-EC2`** criado no passo anterior.
*   Clique em **Create security group**.

---

## Passo 3: Armazenamento Compartilhado (Amazon EFS)

**O Problema:** Um arquivo enviado para o Servidor A precisa ser lido pelo Servidor B[cite: 1]. Discos normais (EBS) são atrelados a uma única máquina[cite: 1].
**A Solução:** Usar o Amazon Elastic File System (EFS), que funciona como uma pasta de rede compartilhada para Linux[cite: 1].

1. Busque por **EFS** na barra de pesquisa superior e acesse o serviço.
2. Clique no botão **Create file system** e, na janela que abrir, clique em **Customize** (para termos mais controle).
3. **Passo 1 (Settings):** Dê um nome (ex: `JuseByte-Storage`). Em *Performance*, deixe as opções padrão. **Importante:** Certifique-se de que a opção **Enable encryption of data at rest** (Criptografia em repouso) esteja marcada[cite: 1]. Clique em **Next**.
4. **Passo 2 (Network):** Aqui está o segredo. Você verá uma lista com suas Zonas de Disponibilidade (Subnets). Na coluna **Security groups**, remova o grupo "default" de todas as linhas e adicione o **`SG-EFS`** que criamos no Passo 2.
5. Clique em **Next** até o final e depois em **Create**.
6. **Atenção:** Na tela principal do seu novo EFS, anote o **File system ID** (ex: `fs-0abcd1234efgh5678`). Você precisará dele no script final.

---

## Passo 4: O Balanceador de Carga (Application Load Balancer)

**O Problema:** Distribuir os acessos dos clientes para instâncias saudáveis usando um único endereço[cite: 1].
**A Solução:** Criar um Target Group (Grupo de Destino) e, em seguida, o Load Balancer (Balanceador).

**4.1. Criando o Target Group (Quem recebe o tráfego?):**
1. Volte ao painel do **EC2**. No menu lateral esquerdo, desça até *Load Balancing* e clique em **Target Groups** > **Create target group**.
2. Em *Choose a target type*, selecione **Instances**.
3. Em *Target group name*, digite `JuseByte-TG`. Protocolo: HTTP, Porta: 80.
4. Em *Health checks*, deixe o caminho como `/`. (Isso diz ao balanceador para acessar a raiz do site e verificar se ele responde bem[cite: 1]).
5. Clique em **Next** e, na tela seguinte (sem selecionar nenhuma instância por enquanto), clique em **Create target group**.

**4.2. Criando o Load Balancer:**
1. No menu lateral, clique em **Load Balancers** > **Create Load Balancer**.
2. Em *Application Load Balancer*, clique em **Create**.
3. **Nome:** `JuseByte-ALB`.
4. **Scheme:** `Internet-facing` (para que os clientes possam acessá-lo da internet).
5. **Network mapping:** Selecione sua VPC (geralmente a Default) e marque **pelo menos duas** zonas de disponibilidade (Availability Zones).
6. **Security groups:** Remova o "default" e adicione o **`SG-ALB`**.
7. **Listeners and routing:** Na porta HTTP 80, no campo *Default action*, selecione o grupo que criamos: **`JuseByte-TG`**.
8. Vá até o final e clique em **Create load balancer**.

---

## Passo 5: Automação Total (Launch Template e User Data)

**O Problema:** Iniciar instâncias automaticamente, sem ninguém fazer login manual, já configuradas e prontas para rodar[cite: 1].
**A Solução:** Criar um "Molde" (Launch Template) contendo o script mágico de inicialização (User Data).

1. No painel do **EC2**, clique em **Launch Templates** no menu lateral > **Create launch template**.
2. **Nome:** `JuseByte-Template`. Marque a caixinha "Provide guidance...".
3. **AMI (Sistema Operacional):** Clique em *Quick Start* e escolha **Amazon Linux 2023** ou Amazon Linux 2.
4. **Instance type:** Selecione `t2.micro` (família x86) ou `t4g.micro` (família ARM).
5. **Key pair:** Selecione *Proceed without a key pair* (Não precisamos de chave SSH, pois não faremos login manual! O laboratório proíbe intervenção pós-boot[cite: 1]).
6. **Network settings:**
   * Em *Firewall (security groups)*, selecione **Select existing security group**.
   * Escolha o **`SG-EC2`**.
7. **Advanced details (Aqui a mágica acontece):**
   * Em **IAM instance profile**, selecione a role criada no Passo 1: **`JuseByte-EC2-Role`**.
   * Role a tela até o final, até encontrar o campo **User data**. Copie e cole o código abaixo.
   * **ATENÇÃO:** Substitua a palavra `ID_DO_SEU_EFS_AQUI` pelo ID que você anotou no Passo 3.

```bash
#!/bin/bash
# 1. Atualizar o sistema e instalar ferramentas basicas
yum update -y
yum install -y amazon-efs-utils unzip wget

# 2. Criar a pasta dos uploads e montar o disco compartilhado (EFS)
# O parametro '-o tls' garante a criptografia em transito exigida pelo lab
mkdir -p /mnt/uploads
mount -t efs -o tls ID_DO_SEU_EFS_AQUI:/ /mnt/uploads

# 3. Preparar o ambiente para o site
mkdir -p /opt/jusebyte
cd /opt/jusebyte

# 4. Baixar a aplicacao do bucket S3 do lab
wget [https://worldskills.s3.us-east-1.amazonaws.com/br-pr-2026/lab2/site.zip](https://worldskills.s3.us-east-1.amazonaws.com/br-pr-2026/lab2/site.zip)
unzip -o site.zip

# 5. Descobrir a arquitetura da maquina para rodar o arquivo certo
# O comando uname -m retorna x86_64 ou aarch64
ARQUITETURA=$(uname -m)

if [ "$ARQUITETURA" == "x86_64" ]; then
    EXECUTAVEL="site-linux-amd64"
elif [ "$ARQUITETURA" == "aarch64" ]; then
    EXECUTAVEL="site-linux-arm64"
fi

# 6. Dar permissao para o sistema rodar o arquivo e inicia-lo
# O 'nohup' e o '&' no final fazem o site rodar em segundo plano, liberando o script
chmod +x ./$EXECUTAVEL
nohup ./$EXECUTAVEL > log_aplicacao.txt 2>&1 &
