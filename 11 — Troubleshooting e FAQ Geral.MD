# 11 — Troubleshooting e FAQ Geral ⭐ (leia na véspera da prova)

---

## 1. Método universal de diagnóstico

Sempre na mesma ordem — **de fora para dentro**:

```
1. O recurso existe e está "running/available"?         (console)
2. A REDE alcança?     Rota → IP público → SG → NACL    (VPC)
3. O SERVIÇO responde? systemctl / logs / curl local     (dentro da máquina)
4. O DNS resolve?      dig / nslookup                    (nome vs IP)
5. A PERMISSÃO existe? IAM role / policy                 (Access Denied)
```

> Ferramenta secreta: **VPC → Reachability Analyzer** — mostra exatamente onde o pacote morre.

---

## 2. Playbooks (sintoma → checklist)

### 🔴 "Não consigo SSH na instância" (timeout)

1. Instância **running** + **2/2 checks**?
2. Tem **IP público**? (privada não recebe SSH direto da internet)
3. **SG** entrada TCP 22 liberada p/ **seu IP atual** (ele muda!)?
4. Sub-rede tem rota `0.0.0.0/0 → IGW`?
5. NACL default (permite tudo)?
6. Plano B: **EC2 Instance Connect** ou **Session Manager**.

### 🔴 "SSH: Permission denied (publickey)"

Não é rede — é autenticação: `.pem` com `chmod 400` | usuário certo (`ec2-user`/`ubuntu`) | key pair correspondente à instância.

### 🔴 "O site não abre (http://IP)"

1. `curl -I http://localhost` **dentro** da instância → responde?
   - Não → serviço parado: `sudo systemctl status httpd` / `nginx` → `start` + veja `journalctl -u httpd -n 50`.
   - User data falhou? → `sudo cat /var/log/cloud-init-output.log`.
2. Responde local mas não de fora → **SG porta 80** / IP público / rota IGW.
3. Está usando **https://** num servidor que só tem 80? Teste com `http://`.

### 🔴 "Instância privada não baixa pacotes (yum/apt trava)"

1. Tabela de rotas da sub-rede privada tem `0.0.0.0/0 → nat-...`?
2. O **NAT Gateway está numa sub-rede PÚBLICA** e *Available*?
3. A rota da pública tem IGW? (NAT depende disso)
4. SG de saída da instância liberado (default = sim)?

### 🔴 "Aplicação não conecta no RDS"

1. SG do RDS: entrada 3306/5432 **a partir do SG do app** (ou CIDR da VPC)?
2. Mesma VPC? (VPCs diferentes exigem peering + rotas)
3. Usando o **endpoint DNS** (não IP)? Porta certa?
4. Status *Available*? Credenciais corretas? (`mysql -h endpoint -u admin -p` testa rápido)

### 🔴 "S3 dá 403 / AccessDenied"

1. **Block Public Access** (bucket E conta) — desligado, se o objetivo é público?
2. **Bucket policy** com `s3:GetObject` e `Resource: ...:::bucket/*` (o `/*` importa!)
3. Acesso via app/EC2 → a **role** tem permissão?
4. Site estático → usando o **endpoint website**, e o hosting está *Enabled*?

### 🔴 "ALB retorna 5xx"

| Código | Significado | Onde olhar |
|---|---|---|
| **503** | Nenhum target healthy/registrado | Target group → Health status |
| **502** | App respondeu inválido/caiu | Logs da aplicação |
| **504** | App demorou (timeout) | Performance/queries lentas |

Targets *unhealthy*: SG da instância aceita a porta **vindo do SG do ALB**? Path do health check retorna **200**? Serviço de pé?

### 🔴 "ASG cria e destrói instâncias em loop"

Health check (ELB) reprovando tudo → app demora a subir (aumente **grace period**), path errado, ou user data quebrado (olhe o cloud-init-output de uma instância antes de ela morrer).

### 🔴 "Access Denied no console/CLI/Lambda"

- Quem executa? Usuário → política do usuário/grupo. EC2/Lambda → **role** anexada.
- Lembre: **Deny explícito vence tudo**; Learner Lab bloqueia várias ações por design.
- CloudTrail mostra o erro exato da chamada.

### 🔴 "DNS não resolve / aponta errado"

```bash
dig +short site.com          # o que o mundo vê
dig +short site.com @8.8.8.8 # ignora cache local
```
Registro certo na hosted zone? NS do registrador = NS da zona AWS? TTL alto segurando valor antigo? CNAME na raiz (proibido — use Alias)?

### 🔴 "Fatura veio alta do nada"

Cost Explorer → por serviço. Suspeitos de sempre: **NAT Gateway**, **ALB**, **RDS**, **EIP não usado**, EBS/snapshots órfãos, instância esquecida em **outra região** (olhe o seletor!). Rode o checklist de limpeza (arquivo 10).

---

## 3. FAQ relâmpago (respostas de 1 linha)

**Rede/VPC**
- Sub-rede pública é...? → a que tem rota `0.0.0.0/0` para o **IGW**.
- SG vs NACL em 5 palavras? → *Instância-stateful-allow* vs *sub-rede-stateless-allow/deny*.
- NAT Gateway fica em sub-rede...? → **pública**, com Elastic IP.
- Peering é transitivo? → **Não.**
- Quantos IPs a AWS reserva por sub-rede? → **5**.
- Ping falha, SSH funciona? → libere **ICMP** no SG.

**EC2**
- User data roda quando? → **primeiro boot**, como root (log: `cloud-init-output.log`).
- Stop/Start muda o quê? → o **IP público** (a menos que EIP).
- IP fixo? → **Elastic IP** (cobra se ficar ocioso).
- Acessar instância privada sem chave? → **Session Manager** (role SSM).
- Metadados? → `curl http://169.254.169.254/...` (com token IMDSv2).

**Storage**
- Compartilhar disco entre várias EC2? → **EFS** (porta 2049).
- Mover EBS de AZ? → **snapshot** → novo volume.
- Versioning desliga? → não; só **suspende**.
- URL temporária p/ objeto privado? → **presigned URL**.

**Banco**
- HA no RDS? → **Multi-AZ**. Escalar leitura? → **Read Replica**.
- Conectar usa IP ou endpoint? → **endpoint** (sempre).
- Restore sobrescreve o banco? → não; **cria instância nova**.
- DynamoDB: buscar sem chave? → crie um **GSI** (Scan é o último recurso).

**HA/Entrega**
- ALB vs NLB? → camada **7** vs **4** (NLB tem IP fixo).
- 503 no ALB? → **sem targets healthy**.
- Apontar raiz do domínio p/ ALB? → registro **Alias**.
- Cache do CloudFront desatualizado? → **Invalidation `/*`**.
- Cert p/ CloudFront? → ACM em **us-east-1**. P/ ALB? → mesma região do ALB.

**Gestão**
- Auditoria de "quem fez"? → **CloudTrail**. Métricas? → **CloudWatch**.
- Ver RAM da EC2? → precisa do **CloudWatch Agent**.
- Evitar susto na fatura? → **Budget** + limpeza pós-lab.
- Apagar tudo de um lab de uma vez? → crie via **CloudFormation** e delete a stack.

---

## 4. Kit de comandos para colar na prova

```bash
# Servidor web instantâneo (Amazon Linux 2023)
sudo dnf install -y httpd && sudo systemctl enable --now httpd
echo "<h1>ws-web $(hostname)</h1>" | sudo tee /var/www/html/index.html

# Diagnóstico local
sudo systemctl status httpd
sudo ss -tulpn | grep -E ':80|:443'
curl -I http://localhost
sudo tail -50 /var/log/cloud-init-output.log

# Rede
ip a && ip route
ping -c3 8.8.8.8 ; dig +short google.com
nc -zv IP_DO_RDS 3306

# AWS CLI (CloudShell)
aws sts get-caller-identity            # quem sou eu
aws ec2 describe-instances --query "Reservations[].Instances[].[InstanceId,State.Name,PublicIpAddress]" --output table
aws s3 ls
```
