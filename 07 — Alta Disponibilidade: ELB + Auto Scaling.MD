# 07 — Alta Disponibilidade: ELB + Auto Scaling ⭐

> O combo **ALB + Auto Scaling Group em 2 AZs** é A arquitetura de prova. Treine até fazer em 15 minutos.

---

## 1. Tipos de Load Balancer

| | **ALB** (Application) | **NLB** (Network) | **GWLB** (Gateway) |
|---|---|---|---|
| Camada | 7 (HTTP/HTTPS) | 4 (TCP/UDP/TLS) | 3 (appliances) |
| Roteia por | Path (`/api`), host, header | Porta/protocolo | — |
| Performance | Alta | Extrema (milhões req/s) | — |
| IP fixo | Não (use DNS) | ✅ Sim (1 por AZ / EIP) | — |
| Uso | Sites e APIs (padrão) ⭐ | Jogos, IoT, IP fixo exigido | Firewalls de terceiros |

*(CLB = Classic, legado — só reconhecer o nome.)*

### Peças do ALB

```
Internet → [Listener :80/:443] → [Regras] → [Target Group] → instâncias/IPs/Lambda
                                                  ↑ Health check decide quem recebe tráfego
```

- **Listener:** porta em que o ALB escuta.
- **Regra:** "se path = /api/* → target group B" (roteamento por caminho/host).
- **Target group:** conjunto de destinos + configuração de **health check**.

---

## 2. Passo a passo: ALB + ASG completo

### 2.1 Pré-requisitos
- VPC com **2 sub-redes públicas em AZs diferentes** (ALB exige ≥2 AZs).
- **AMI pronta** da aplicação **ou** user data que instala tudo (arquivo 04).
- SGs: `ws-sg-alb` (80 de `0.0.0.0/0`) e `ws-sg-web` (80 **a partir de `ws-sg-alb`** ⭐ — cadeia de SGs).

### 2.2 Target Group
**EC2 → Target groups → Create:**
1. Target type: **Instances** | Nome: `ws-tg-web` | Protocolo HTTP:80 | VPC `ws`.
2. **Health check:** HTTP, Path `/` (ou `/health` se a prova pedir). Em *Advanced*: healthy threshold 2, interval 15s (acelera o lab).
3. Não registre alvos agora (o ASG fará isso) → Create.

### 2.3 Load Balancer
**EC2 → Load balancers → Create → ALB:**
1. Nome `ws-alb` | Internet-facing | IPv4.
2. Network mapping: VPC `ws` + **as 2 sub-redes públicas**.
3. SG: `ws-sg-alb`.
4. Listener HTTP:80 → forward para `ws-tg-web` → Create.
5. Anote o **DNS name** (`ws-alb-123.sa-east-1.elb.amazonaws.com`).

### 2.4 Launch Template
**EC2 → Launch templates → Create:**
Nome `ws-lt-web` → AMI, tipo `t3.micro`, key pair, SG `ws-sg-web`, User data do servidor web (que mostre a AZ — assim você VÊ o balanceamento). *(Launch Configuration é legado; sempre template.)*

### 2.5 Auto Scaling Group
**EC2 → Auto Scaling groups → Create:**
1. Nome `ws-asg-web` → Launch template `ws-lt-web`.
2. VPC `ws` → selecione as sub-redes (públicas neste lab; **privadas** na arquitetura "correta" com ALB público na frente).
3. **Attach to an existing load balancer** → target group `ws-tg-web`.
4. ✔️ **Turn on ELB health checks** (além do EC2 check) — se o Apache morrer, o ASG substitui a instância.
5. Capacidade: Desired **2**, Min **2**, Max **4**.
6. Scaling policy: **Target tracking** → Average CPU utilization = **50**.
7. Create → em minutos, 2 instâncias sobem e ficam **healthy** no target group.

### 2.6 Testar
- Abra o DNS do ALB e dê **F5 várias vezes** → a AZ mostrada alterna = balanceando. ✔️
- **Auto-healing:** termine uma instância na mão → o ASG cria outra sozinho.
- **Scaling por carga:** na instância, `sudo dnf install -y stress-ng && stress-ng --cpu 4 --timeout 300` → CPU sobe → nasce instância nova (veja na aba *Activity* do ASG).

---

## 3. Auto Scaling — teoria que cai

| Política | Como age |
|---|---|
| **Target tracking** ⭐ | "Mantenha CPU em 50%" — a AWS calcula tudo (use esta) |
| Step scaling | CPU>70 soma 1; CPU>90 soma 2 |
| Scheduled | "Sexta 18h, desired=10" (Black Friday) |

- **Desired / Min / Max:** o ASG sempre persegue o *desired*, dentro dos limites.
- **Cooldown/warmup:** pausa entre ações p/ instância "esquentar" antes de medir de novo.
- Distribui instâncias **balanceadas entre as AZs** selecionadas.
- **Instance refresh:** troca gradual das instâncias após atualizar o launch template (nova versão da app).

---

## 4. HTTPS no ALB (quando a prova pedir)

1. **ACM (Certificate Manager)** → Request certificate → domínio → validação **DNS** (se a zona está no Route 53, botão *Create records* faz tudo).
2. ALB → Listeners → **Add listener HTTPS:443** → escolher certificado → forward pro target group.
3. (Clássico) Redirecionar 80→443: edite o listener HTTP:80 → ação **Redirect** para HTTPS 443.
4. Certificado do ALB = **mesma região** do ALB (regra do us-east-1 é só p/ CloudFront).

---

## 5. Dúvidas comuns (FAQ)

**"ALB responde 503."**
Nenhum alvo **healthy** no target group. Veja a coluna *Health status* → instâncias unhealthy? (próxima pergunta) → nenhum alvo registrado? (ASG anexado ao TG errado?).

**"Instâncias sempre unhealthy."**
① SG da instância permite a porta **a partir do SG do ALB**? ② O serviço está de pé (`systemctl status httpd`)? ③ O **path** do health check existe e retorna **200**? (`curl -I localhost/health`) ④ Porta certa no TG?

**"502 Bad Gateway"** → a app respondeu "lixo"/caiu no meio (veja logs da app). **"504 Timeout"** → app demorou demais (aumenta timeout ou conserta a app).

**"Acesso pelo IP da instância funciona, pelo ALB não."**
Clássico de SG: liberaram 80 só p/ `0.0.0.0/0` na instância e depois restringiram, ou o SG do ALB não permite saída/entrada correta. Confirme a cadeia: internet→ALB(80) e ALB-SG→instância(80).

**"O ASG fica criando e matando instâncias em loop."**
Health check ELB reprovando (path errado / app demora a subir) → o ASG mata e recria eternamente. Corrija o health check ou aumente o *health check grace period*.

**"Terminei a instância e ela renasce."**
Comportamento correto do ASG! Para desligar tudo: **Desired e Min = 0** (ou delete o ASG).

**"ALB tem IP fixo?"**
Não — sempre use o **DNS name** (ou Route 53 Alias). IP fixo obrigatório = NLB.
