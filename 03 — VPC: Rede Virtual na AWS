# 03 — VPC: Rede Virtual na AWS ⭐ (capítulo mais importante)

---

## 1. Mapa mental

```
VPC 10.0.0.0/16 ──────────────────────────────────────────────┐
│                                                              │
│  AZ sa-east-1a                    AZ sa-east-1b              │
│  ┌──────────────────┐             ┌──────────────────┐       │
│  │ pública 10.0.1.0/24            │ pública 10.0.2.0/24      │
│  │  [EC2 web] [NAT GW]│           │  [EC2 web]       │       │
│  └────────┬─────────┘             └────────┬─────────┘       │
│  ┌────────┴─────────┐             ┌────────┴─────────┐       │
│  │ privada 10.0.11.0/24           │ privada 10.0.12.0/24     │
│  │  [EC2 app] [RDS] │             │  [RDS standby]   │       │
│  └──────────────────┘             └──────────────────┘       │
│                                                              │
└───────────────[ Internet Gateway ]───────► Internet          │
```

| Componente | Função (1 linha) |
|---|---|
| **VPC** | Sua rede privada isolada na AWS (você escolhe o CIDR) |
| **Sub-rede** | Fatia da VPC presa a **uma** AZ |
| **Internet Gateway (IGW)** | Porta de entrada/saída para a internet (1 por VPC) |
| **Tabela de rotas** | Define para onde o tráfego de cada sub-rede vai |
| **NAT Gateway** | Dá **saída** à internet para sub-redes privadas (💰 cobra por hora) |
| **Security Group** | Firewall **da instância** (stateful) |
| **NACL** | Firewall **da sub-rede** (stateless) |
| **VPC Endpoint** | Acesso privado a serviços AWS sem passar pela internet |

**Definições que caem em prova:**
- Sub-rede **pública** = sua tabela de rotas tem `0.0.0.0/0 → IGW`.
- Sub-rede **privada** = não tem rota para o IGW (pode ter para NAT).

---

## 2. Criar uma VPC completa no Console (jeito rápido ⭐)

**VPC → Create VPC → selecione "VPC and more"** (assistente visual):

1. Name tag: `ws`
2. IPv4 CIDR: `10.0.0.0/16`
3. AZs: **2** | Public subnets: **2** | Private subnets: **2**
4. NAT gateways: `In 1 AZ` (💰 só se precisar de saída nas privadas; senão `None`)
5. VPC endpoints: `S3 Gateway` (grátis e útil)
6. **Create VPC** → o assistente cria sub-redes, IGW, rotas e associações sozinho, com diagrama.

> Na prova, se pedirem nomes específicos de sub-rede, crie manualmente (abaixo) ou renomeie depois.

### Jeito manual (entenda o que o assistente faz)

1. **Your VPCs → Create VPC** → "VPC only" → CIDR `10.0.0.0/16`.
2. **Subnets → Create subnet** → escolha a VPC → 4 sub-redes (defina **AZ** e CIDR de cada uma: 10.0.1.0/24, 10.0.2.0/24, 10.0.11.0/24, 10.0.12.0/24).
3. Nas 2 públicas: **Actions → Edit subnet settings → Enable auto-assign public IPv4** ✔️ (senão a EC2 nasce sem IP público!).
4. **Internet gateways → Create** → depois **Actions → Attach to VPC**.
5. **Route tables → Create** `ws-rt-publica` → **Routes → Edit** → add `0.0.0.0/0 → IGW` → aba **Subnet associations** → associe as 2 públicas.
6. (Se houver NAT) **NAT gateways → Create** → sub-rede **PÚBLICA** ⚠️ → Allocate Elastic IP → Create. Depois crie `ws-rt-privada` com `0.0.0.0/0 → nat-...` e associe as privadas.

**Erro clássico:** criar o NAT Gateway na sub-rede privada. Ele **precisa** estar na pública (é ele quem fala com o IGW).

---

## 3. Security Group × NACL (tabela sagrada 🙏)

| | **Security Group** | **Network ACL** |
|---|---|---|
| Atua em | Instância (ENI) | Sub-rede inteira |
| Estado | **Stateful** — resposta liberada automaticamente | **Stateless** — precisa liberar ida **E** volta |
| Regras | Só **Allow** | **Allow e Deny** |
| Avaliação | Todas as regras (união) | Em ordem numérica; a 1ª que casar decide |
| Padrão | Entrada: nega tudo / Saída: permite tudo | NACL default: permite tudo |
| Uso típico | Firewall do dia a dia | Bloquear IP/faixa específica na sub-rede |

### Regras de SG — exemplos prontos

**SG do servidor web (`ws-sg-web`):**

| Tipo | Porta | Origem |
|---|---|---|
| HTTP | 80 | `0.0.0.0/0` |
| HTTPS | 443 | `0.0.0.0/0` |
| SSH | 22 | **Seu IP** (`My IP`) — nunca 0.0.0.0/0 em prova |

**SG do banco (`ws-sg-db`):**

| Tipo | Porta | Origem |
|---|---|---|
| MySQL/Aurora | 3306 | **`ws-sg-web`** (o próprio SG do app!) ⭐ |

> **Padrão-ouro:** origem = **outro Security Group**. "Quem tiver o SG web pode falar com o banco." Escala sozinho, sem IP fixo.

### Pegadinha do NACL: portas efêmeras

Se criar regra de NACL permitindo entrada 80, a **resposta** sai por uma porta alta aleatória. Precisa liberar **Outbound 1024–65535** (e vice-versa). Por isso: em prova, prefira mexer em SG e deixe o NACL default, a não ser que peçam explicitamente um bloqueio.

---

## 4. Conectividade avançada

| Recurso | Para quê | Detalhe de prova |
|---|---|---|
| **VPC Peering** | Ligar 2 VPCs (mesma conta ou não) | **Não é transitivo** (A↔B e B↔C ≠ A↔C); CIDRs não podem se sobrepor; precisa adicionar **rotas dos dois lados** e liberar SG |
| **Endpoint Gateway** | S3 e DynamoDB via rede interna | **Grátis**; vira entrada na tabela de rotas |
| **Endpoint Interface** | Demais serviços (SSM, ECR...) | Cria ENI na sub-rede; 💰 cobra/hora |
| **Transit Gateway** | Hub para muitas VPCs/VPN | Citação: "roteador central" |
| **Site-to-Site VPN** | Ligar rede on-premises à VPC | Usa Customer GW + Virtual Private GW |
| **VPC Flow Logs** | Logar tráfego ACCEPT/REJECT | Ativar na VPC → destino CloudWatch Logs; ótimo p/ provar bloqueio |

**Reachability Analyzer** (VPC → Reachability Analyzer): informe origem e destino e ele **aponta exatamente onde o pacote morre** (SG? NACL? rota?). Arma secreta de troubleshooting.

---

## 5. Checklist: "minha EC2 pública precisa funcionar"

- [ ] VPC tem **IGW anexado**
- [ ] Sub-rede associada a tabela de rotas com `0.0.0.0/0 → IGW`
- [ ] Instância tem **IP público** (auto-assign ligado ou Elastic IP)
- [ ] **SG** permite a porta (80/22/443) da origem correta
- [ ] **NACL** da sub-rede não bloqueia (default = OK)
- [ ] Serviço rodando na instância (`sudo systemctl status nginx`)

Memorize a ordem: **Rota → IP → SG → NACL → Serviço.**

---

## 6. Dúvidas comuns (FAQ)

**"Qual a diferença entre a VPC default e uma criada por mim?"**
A default (`172.31.0.0/16`) já vem com IGW, sub-redes públicas em todas as AZs e auto-assign de IP ligado. Boa p/ teste rápido; prova séria = crie a sua.

**"Minha instância privada precisa baixar pacotes (yum/apt). Como?"**
NAT Gateway na sub-rede pública + rota `0.0.0.0/0 → NAT` na tabela da privada. (Alternativa sem custo p/ acessar S3: Endpoint Gateway.)

**"Posso mudar o CIDR da VPC depois?"**
Não dá para editar o principal, mas dá para **adicionar CIDRs secundários**. Sub-rede não muda de CIDR: apague e recrie.

**"Duas sub-redes na mesma AZ podem?"**
Sim. O que não pode é uma sub-rede em duas AZs.

**"Preciso liberar saída no SG?"**
Por padrão a saída já é liberada (All traffic). Só se preocupe se alguém tiver restringido.

**"IGW tem custo? E NAT?"**
IGW é grátis. **NAT Gateway cobra por hora + por GB** — delete depois do lab.

**"Ping não funciona mas SSH sim!"**
Ping é **ICMP**, não TCP. Libere `All ICMP - IPv4` no SG se quiser responder ping.
