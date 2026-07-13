# 04 — EC2: Servidores Virtuais

---

## 1. Conceitos-chave

| Termo | Definição |
|---|---|
| **Instância** | O servidor virtual em si |
| **AMI** | Imagem (SO + software) usada para criar a instância |
| **Tipo de instância** | Combinação de CPU/RAM/rede (ex.: `t3.micro`) |
| **Key pair** | Par de chaves para SSH (AWS guarda a pública, você baixa a `.pem`) |
| **User data** | Script que roda **como root** no **primeiro boot** |
| **EBS** | Disco virtual persistente da instância |
| **ENI** | Placa de rede virtual |
| **Elastic IP** | IP público **fixo** (o normal muda ao parar/iniciar) |

### Famílias de instância (decore a letra)

| Família | Otimizada para | Exemplo de uso |
|---|---|---|
| **t** | Uso geral **burstável** (créditos de CPU) | `t3.micro` — labs, sites pequenos |
| **m** | Uso geral equilibrado | Aplicações web |
| **c** | **C**omputação (CPU) | Processamento, jogos |
| **r** | **R**AM (memória) | Bancos em memória, cache |
| **i / d** | Disco local rápido | Big data |
| **g / p** | **G**PU | ML, renderização |

Leitura do nome: `t3.micro` = família **t**, geração **3**, tamanho **micro**.

---

## 2. Lançar uma instância web (passo a passo de prova)

**EC2 → Instances → Launch instance:**

1. **Name:** `ws-web-01`
2. **AMI:** Amazon Linux 2023 (ou Ubuntu 24.04 se a prova pedir)
3. **Instance type:** `t3.micro` (ou o que a prova pedir)
4. **Key pair:** criar/selecionar (`ws-key`) — baixe o `.pem` e guarde!
5. **Network settings → Edit:**
   - VPC: `ws` | Subnet: `publica-a`
   - Auto-assign public IP: **Enable**
   - SG: criar `ws-sg-web` liberando 80 (0.0.0.0/0) e 22 (My IP)
6. **Advanced details → IAM instance profile:** role (se precisar de S3/SSM)
7. **Advanced details → User data:**

```bash
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl enable --now httpd
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
echo "<h1>Servidor ws-web-01 na AZ: $AZ</h1>" > /var/www/html/index.html
```

*(Ubuntu: troque `dnf` por `apt-get`, `httpd` por `apache2`.)*

8. **Launch** → aguarde **2/2 checks passed** → abra `http://IP_PUBLICO`.

> **User data só roda no PRIMEIRO boot.** Log de execução: `sudo cat /var/log/cloud-init-output.log` (primeiro lugar a olhar quando "o script não funcionou").

---

## 3. Formas de conectar

| Método | Requisitos | Quando usar |
|---|---|---|
| **EC2 Instance Connect** (botão *Connect* no console) | SG liberando 22; AMI compatível (Amazon Linux/Ubuntu) | O mais rápido em prova ⭐ |
| **SSH pelo terminal** | `.pem` + IP público + porta 22 | Padrão de mercado |
| **Session Manager** (aba *Session Manager* do Connect) | Role com `AmazonSSMManagedInstanceCore`; **não precisa de porta 22 nem key pair** | Instâncias privadas! ⭐ |

### SSH manual

```bash
chmod 400 ws-key.pem
ssh -i ws-key.pem ec2-user@IP_PUBLICO
```

**Usuário padrão por AMI:** Amazon Linux → `ec2-user` | Ubuntu → `ubuntu` | Debian → `admin` | RHEL/CentOS → `ec2-user`/`centos`.

---

## 4. Metadados da instância (IMDSv2)

De **dentro** da instância:

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/
```

Caminhos úteis: `public-ipv4`, `local-ipv4`, `instance-id`, `placement/availability-zone`, `iam/security-credentials/`.

---

## 5. EBS — discos

| Tipo | Perfil | Uso |
|---|---|---|
| **gp3** | SSD geral (3000 IOPS base) | Padrão para tudo ⭐ |
| gp2 | SSD geral (geração anterior) | Legado |
| io1/io2 | SSD provisionado (alto IOPS) | Bancos exigentes |
| st1 | HDD throughput | Big data sequencial |
| sc1 | HDD frio | Arquivamento barato |

**Fatos de prova:**
- EBS fica preso a **uma AZ**. Para mover: **snapshot** → criar volume em outra AZ (ou AMI p/ outra região).
- **Snapshots são incrementais** e ficam no S3 (gerenciado).
- **Delete on termination:** o volume raiz por padrão morre com a instância; volumes extras não.
- Dá para **aumentar** o volume a quente (Modify volume). Depois, dentro do SO:

```bash
sudo growpart /dev/xvda 1
sudo xfs_growfs -d /   # (XFS, Amazon Linux)  |  ext4: sudo resize2fs /dev/xvda1
```

- **Anexar 2º disco:** Volumes → Create (mesma AZ!) → Attach → na instância: `lsblk`, `sudo mkfs -t xfs /dev/xvdf`, `sudo mount /dev/xvdf /dados` (+ `/etc/fstab` p/ persistir).

### AMI × Snapshot

- **Snapshot** = foto de **um volume**.
- **AMI** = molde de **instância completa** (SO + volumes). Criar: instância → **Actions → Image and templates → Create image**. Use AMI para "clonar" servidor configurado — base do Auto Scaling.

---

## 6. Ciclo de vida e IPs

| Ação | Efeito no IP público | Cobrança |
|---|---|---|
| **Stop → Start** | **Muda** (a menos que use Elastic IP) | Para de cobrar computação; EBS continua |
| **Reboot** | Mantém | Continua |
| **Terminate** | Some; instância é destruída | Para tudo (EBS com delete-on-termination some) |

- **Elastic IP:** Network & Security → Elastic IPs → Allocate → **Associate** à instância. 💰 EIP **não associado** (ou associado a instância parada) é cobrado.
- **Termination protection** (Actions → Instance settings): impede terminate acidental — provas adoram pedir.
- **Status checks 2/2:** o 1º é da AWS (hardware), o 2º do SO. `1/2` = problema dentro da instância (kernel, rede interna) → reboot costuma resolver.

---

## 7. Dúvidas comuns (FAQ)

**"Permission denied (publickey) no SSH."**
① `chmod 400` na .pem; ② usuário certo da AMI (`ec2-user` vs `ubuntu`); ③ key pair correto da instância; ④ está usando o IP público, não o privado.

**"Timeout no SSH."**
Não é chave — é **rede**: SG porta 22 liberada para o seu IP? Sub-rede pública com rota p/ IGW? Instância tem IP público? (Checklist do arquivo 03.)

**"Rodei de novo o user data e nada aconteceu."**
Ele só roda no primeiro boot. Para reexecutar lógica, rode manualmente via SSH ou recrie a instância. Veja erros em `/var/log/cloud-init-output.log`.

**"Meu IP público mudou sozinho!"**
Você parou e iniciou a instância. Solução definitiva: **Elastic IP**.

**"t3.micro está lenta em teste de carga."**
Família *t* usa **créditos de CPU**: acabou o crédito, limita a CPU. Para carga sustentada, use *m*/*c* (ou ative modo `unlimited`).

**"Apago a instância e ela volta sozinha!"**
Ela pertence a um **Auto Scaling Group** com desired > 0 (arquivo 07). Ajuste o ASG, não a instância.

**"Esqueci de anexar a role."**
Actions → Security → **Modify IAM role** — sem recriar a instância.
