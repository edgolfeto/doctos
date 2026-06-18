# Resolução do Laboratório: Armazenamento Compartilhado e Provisionamento Automatizado (Ambiente Sandbox)

[cite_start]Este guia detalha a solução para o laboratório de implantação de uma aplicação web em Go[cite: 11], adaptada para ambientes restritos de Sandbox (sem permissões para IAM Roles e sem acesso ao Amazon EFS).

## Arquitetura da Solução

Para contornar as restrições do ambiente, a arquitetura foi desenhada da seguinte forma:
* **Application Load Balancer (ALB):** Ponto único de entrada, distribuindo o tráfego na porta 80.
* **Instância EC2 (NFS Server):** Atua como o servidor de arquivos compartilhado, substituindo o EFS.
* **Instâncias EC2 (Web Servers):** Executam a aplicação e são provisionadas automaticamente via *User Data*. [cite_start]Baixam o pacote via URL pública do S3 [cite: 55] (substituindo o uso de IAM Roles) e montam o diretório do servidor NFS localmente.

---

## Passo a Passo da Implementação

### Passo 1: Configuração dos Security Groups (Menor Privilégio)

Você precisará criar três Grupos de Segurança (Security Groups) na mesma VPC:

1. **SG-LoadBalancer:**
   * **Inbound:** HTTP (80) de `0.0.0.0/0` (Internet).
   * **Outbound:** Todo o tráfego permitido.

2. **SG-WebServers:**
   * [cite_start]**Inbound:** HTTP (80) permitindo tráfego **apenas** com origem no `SG-LoadBalancer`[cite: 40].
   * **Inbound:** SSH (22) do seu IP (apenas para *troubleshooting*, se necessário).
   * **Outbound:** Todo o tráfego permitido.

3. **SG-NFSServer:**
   * [cite_start]**Inbound:** NFS (2049) permitindo tráfego **apenas** com origem no `SG-WebServers`[cite: 39].
   * **Inbound:** SSH (22) do seu IP.
   * **Outbound:** Todo o tráfego permitido.

---

### Passo 2: Provisionamento do Armazenamento Compartilhado (Servidor NFS)

Como não podemos usar o EFS, criaremos nosso próprio servidor NFS.

1. Inicie uma instância EC2 simples (ex: `t2.micro` com Amazon Linux 2023).
2. Atribua a ela o **SG-NFSServer**.
3. Acesse a instância via SSH e execute os comandos abaixo para configurar o NFS:

\`\`\`bash
sudo su
dnf install nfs-utils -y

# Criar a pasta que será compartilhada
mkdir -p /mnt/shared_uploads
chmod -R 777 /mnt/shared_uploads

# Configurar as regras de exportação do NFS
echo "/mnt/shared_uploads *(rw,sync,no_root_squash,no_subtree_check)" >> /etc/exports

# Iniciar e habilitar o serviço
systemctl enable rpcbind nfs-server
systemctl start rpcbind nfs-server
exportfs -a
\`\`\`

> ⚠️ **IMPORTANTE:** Anote o **IP Privado** desta instância NFS (ex: `10.0.1.50`). Você precisará dele no próximo passo.

---

### Passo 3: Provisionamento Automatizado dos Web Servers (User Data)

[cite_start]Agora vamos criar as instâncias que rodam a aplicação de forma 100% automatizada[cite: 43]. 

1. Inicie duas instâncias EC2 (ex: `t2.micro` ou `t3.micro`).
2. Atribua a elas o **SG-WebServers**.
3. Em **Configurações Avançadas > User Data**, cole o script abaixo. **Lembre-se de substituir `<IP_PRIVADO_DO_NFS>` pelo IP que você anotou no Passo 2.**

\`\`\`bash
#!/bin/bash
# 1. Atualizar e instalar dependências
dnf update -y
dnf install wget unzip nfs-utils -y

# 2. Criar diretório da aplicação e ponto de montagem
mkdir -p /var/www/jusebyte/uploads
cd /var/www/jusebyte

# 3. Montar o armazenamento compartilhado (NFS)
# Substitua o IP abaixo pelo IP privado real do seu Servidor NFS
NFS_SERVER_IP="<IP_PRIVADO_DO_NFS>"
mount -t nfs $NFS_SERVER_IP:/mnt/shared_uploads /var/www/jusebyte/uploads
echo "$NFS_SERVER_IP:/mnt/shared_uploads /var/www/jusebyte/uploads nfs defaults 0 0" >> /etc/fstab

# 4. Baixar a aplicação via URL pública (Sem uso de IAM Roles)
wget https://worldskills.s3.us-east-1.amazonaws.com/br-pr-2026/lab2/site.zip
unzip site.zip

# 5. Detectar a arquitetura dinamicamente e dar permissão de execução
ARCH=$(uname -m)
if [ "$ARCH" == "x86_64" ]; then
    BIN="site-linux-amd64"
elif [ "$ARCH" == "aarch64" ]; then
    BIN="site-linux-arm64"
fi
chmod +x $BIN

# 6. Criar serviço Systemd para manter a aplicação rodando em background na porta 80
cat <<EOF > /etc/systemd/system/jusebyte.service
[Unit]
Description=JuseByte Web App
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/var/www/jusebyte
# Executando o binário na porta 80 (parâmetro ilustrativo, ajuste conforme o --help da app)
ExecStart=/var/www/jusebyte/$BIN -port 80
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 7. Iniciar a aplicação
systemctl daemon-reload
systemctl enable jusebyte
systemctl start jusebyte
\`\`\`
[cite_start]*(Nota de infraestrutura: O script acima avalia a arquitetura usando `uname -m` para executar o binário correto automaticamente[cite: 62].)*

---

### Passo 4: Configuração do Balanceador de Carga (ALB)

1. Vá em **Target Groups** e crie um grupo do tipo *Instances*.
2. [cite_start]Configure o *Health Check* (geralmente caminho `/` na porta 80, a depender da documentação da aplicação [cite: 25]).
3. Registre as instâncias Web criadas no Passo 3.
4. Vá em **Load Balancers** e crie um *Application Load Balancer*.
5. Adicione um *Listener* na porta 80 e encaminhe o tráfego para o Target Group criado.
6. Associe o ALB ao **SG-LoadBalancer**.

---

### Passo 5: Validação (Critérios de Sucesso)

1. **Acesso:** Copie o DNS gerado pelo ALB e cole no navegador. [cite_start]O site deve carregar perfeitamente[cite: 47].
2. [cite_start]**Consistência:** Faça o upload de um arquivo ou imagem[cite: 48]. Como as requisições estão sendo balanceadas, tente recarregar a página algumas vezes ou forçar o acesso a uma instância diferente. O arquivo continuará visível, provando que o NFS está operando com sucesso como sistema de arquivos centralizado.
