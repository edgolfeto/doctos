# 01 — Fundamentos de Redes (base para tudo na AWS)

> 80% dos erros em provas de cloud são erros de **rede**: CIDR errado, rota faltando, porta bloqueada. Domine este arquivo antes de tudo.

---

## 1. Modelo OSI × TCP/IP (com mapeamento AWS)

| Camada OSI | Nome | Exemplos | Onde aparece na AWS |
|---|---|---|---|
| 7 | Aplicação | HTTP, HTTPS, DNS, SSH, FTP | **ALB** (roteia por URL/host), API Gateway |
| 4 | Transporte | TCP, UDP, portas | **NLB**, regras de Security Group (porta/protocolo) |
| 3 | Rede | IP, roteamento, ICMP | **Tabelas de rota**, NACL, VPC Peering |
| 2 | Enlace | MAC, switch, ARP | ENI (interface de rede da instância) |
| 1 | Física | Cabos, sinais | Datacenter da AWS (não é sua responsabilidade) |

**Frase que resolve prova:** *ALB é camada 7 (entende HTTP), NLB é camada 4 (entende só TCP/UDP e por isso é mais rápido).*

### TCP vs UDP

| | TCP | UDP |
|---|---|---|
| Conexão | Orientado (3-way handshake: SYN → SYN-ACK → ACK) | Sem conexão |
| Garantia de entrega | Sim (retransmite) | Não |
| Velocidade | Menor | Maior |
| Usos | HTTP, SSH, banco de dados | DNS (consultas), streaming, jogos, DHCP |

---

## 2. Endereçamento IPv4

Um IP tem 32 bits divididos em 4 octetos: `192.168.10.25` → `11000000.10101000.00001010.00011001`

### Faixas privadas (RFC 1918) — decore!

| Faixa | CIDR | Uso típico |
|---|---|---|
| 10.0.0.0 – 10.255.255.255 | `10.0.0.0/8` | Redes corporativas grandes, **VPCs na AWS** |
| 172.16.0.0 – 172.31.255.255 | `172.16.0.0/12` | **VPC default da AWS = 172.31.0.0/16** |
| 192.168.0.0 – 192.168.255.255 | `192.168.0.0/16` | Redes domésticas / pequenas |

### IPs especiais

| IP | Significado |
|---|---|
| `127.0.0.1` | Loopback (localhost — a própria máquina) |
| `0.0.0.0/0` | "Qualquer IP" — usado em rotas default e regras abertas |
| `169.254.0.0/16` | Link-local (APIPA). Na AWS: `169.254.169.254` = **serviço de metadados da EC2** |

> IP **privado** não roteia na internet. Para sair, precisa de **NAT** (ou de um IP público).

---

## 3. CIDR e máscara de sub-rede (o coração da prova)

Notação: `IP/prefixo`. O prefixo diz quantos bits são **fixos** (rede); o resto varia (hosts).

**Fórmula:** total de IPs = `2^(32 − prefixo)`

| CIDR | Máscara | Total de IPs | Hosts utilizáveis (rede tradicional)* |
|---|---|---|---|
| /16 | 255.255.0.0 | 65.536 | 65.534 |
| /20 | 255.255.240.0 | 4.096 | 4.094 |
| /24 | 255.255.255.0 | 256 | 254 |
| /25 | 255.255.255.128 | 128 | 126 |
| /26 | 255.255.255.192 | 64 | 62 |
| /27 | 255.255.255.224 | 32 | 30 |
| /28 | 255.255.255.240 | 16 | 14 |

\* Tradicional: descontam-se rede e broadcast (−2). **Na AWS descontam-se 5 IPs por sub-rede** (veja abaixo).

### ⚠️ AWS reserva 5 IPs por sub-rede

Para a sub-rede `10.0.1.0/24`:

| IP | Reservado para |
|---|---|
| 10.0.1.0 | Endereço de rede |
| 10.0.1.1 | Roteador da VPC |
| 10.0.1.2 | DNS da AWS |
| 10.0.1.3 | Uso futuro |
| 10.0.1.255 | Broadcast |

→ Um /24 na AWS tem **251 IPs utilizáveis** (256 − 5). Um /28 tem só **11**.

Limites na AWS: VPC aceita CIDR de **/16 até /28**; sub-rede mínima **/28**.

---

## 4. Exemplos práticos de subnetting (faça no papel!)

### Exemplo 1 — Dividir um /24 em 4 sub-redes iguais

Rede: `192.168.0.0/24`. Preciso de 4 sub-redes → pego 2 bits emprestados (2² = 4) → prefixo vira **/26** (64 IPs cada).

| Sub-rede | Faixa | Broadcast |
|---|---|---|
| 192.168.0.0/26 | .0 – .63 | .63 |
| 192.168.0.64/26 | .64 – .127 | .127 |
| 192.168.0.128/26 | .128 – .191 | .191 |
| 192.168.0.192/26 | .192 – .255 | .255 |

**Truque do salto:** 256 − 192 (último octeto da máscara) = **64** → as sub-redes "pulam" de 64 em 64.

### Exemplo 2 — Plano de VPC padrão de prova

VPC `10.0.0.0/16` com 2 AZs, sub-redes públicas e privadas:

```
10.0.1.0/24  → publica-a   (AZ sa-east-1a)
10.0.2.0/24  → publica-b   (AZ sa-east-1b)
10.0.11.0/24 → privada-a   (AZ sa-east-1a)
10.0.12.0/24 → privada-b   (AZ sa-east-1b)
```

Deixar "espaço" entre públicas (1, 2) e privadas (11, 12) facilita crescer depois. Decore esse esqueleto — serve para 90% das provas.

### Exemplo 3 — Em qual sub-rede está o IP?

O IP `10.0.5.130` pertence a `10.0.5.128/26`?
Salto do /26 = 64 → blocos: .0, .64, **.128**, .192. Como 130 está entre 128 e 191 → **SIM**. ✔️

### Exemplo 4 — Dimensionamento

"Preciso de uma sub-rede para 100 servidores na AWS."
- /25 = 128 − 5 = 123 utilizáveis ✔️ (o /26 daria 59, insuficiente)

---

## 5. Roteamento

- **Tabela de rotas** = lista de "para chegar em X, saia por Y".
- Regra do **prefixo mais longo vence**: rota `10.0.0.0/16 → local` ganha de `0.0.0.0/0 → internet` para tráfego interno.
- `0.0.0.0/0` = **rota padrão** ("todo o resto vai para cá").

Exemplo de tabela de rotas de sub-rede **pública** na AWS:

| Destino | Alvo |
|---|---|
| 10.0.0.0/16 | local (dentro da VPC) |
| 0.0.0.0/0 | igw-xxxx (Internet Gateway) |

Sub-rede **privada** com saída via NAT:

| Destino | Alvo |
|---|---|
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | nat-xxxx (NAT Gateway) |

---

## 6. NAT (Network Address Translation)

Traduz IPs privados ↔ público. É o que permite sua casa inteira navegar com 1 IP público.

- **SNAT (saída):** muitos privados saem por 1 público. Na AWS = **NAT Gateway** (só saída; ninguém de fora inicia conexão para dentro).
- **DNAT / port forwarding (entrada):** redireciona porta do público para um privado específico.

**Analogia:** NAT Gateway é a portaria do prédio — moradores (instâncias privadas) podem sair e receber o que pediram, mas visitantes não entram sozinhos.

---

## 7. DNS — Domain Name System

Tradução nome → IP. Fluxo de uma consulta a `www.loja.com.br`:

```
Navegador → resolver do provedor → servidor raiz (.) → TLD (.br / .com.br)
→ servidor autoritativo da zona "loja.com.br" → responde o IP → cache (TTL)
```

### Tipos de registro (decore!)

| Registro | Aponta para | Exemplo |
|---|---|---|
| **A** | Nome → IPv4 | `www → 54.233.10.20` |
| **AAAA** | Nome → IPv6 | `www → 2600:1f1e::1` |
| **CNAME** | Nome → outro nome | `blog → hospedagem.site.com` (⚠️ não pode na raiz do domínio) |
| **MX** | Servidor de e-mail | `10 mail.loja.com.br` |
| **TXT** | Texto (SPF, verificações) | `"v=spf1 include:..."` |
| **NS** | Servidores de nome da zona | `ns-123.awsdns...` |
| **SOA** | Autoridade da zona | (criado automaticamente) |
| **Alias** | Exclusivo AWS: nome → recurso AWS | raiz do domínio → ALB/CloudFront/S3 ✔️ |

**TTL** = quanto tempo a resposta fica em cache. TTL alto = propagação lenta de mudanças; TTL baixo = mais consultas.

---

## 8. Portas e protocolos essenciais

| Porta | Protocolo | Serviço |
|---|---|---|
| 20/21 | TCP | FTP |
| **22** | TCP | **SSH / SFTP** |
| 23 | TCP | Telnet (inseguro) |
| 25 | TCP | SMTP |
| **53** | TCP/UDP | **DNS** |
| 67/68 | UDP | DHCP |
| **80** | TCP | **HTTP** |
| 123 | UDP | NTP (hora) |
| **443** | TCP | **HTTPS** |
| 1433 | TCP | SQL Server |
| 2049 | TCP | NFS (**EFS** usa essa!) |
| **3306** | TCP | **MySQL / MariaDB / Aurora** |
| **3389** | TCP | **RDP (Windows)** |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 8080 | TCP | HTTP alternativo |

---

## 9. Comandos de diagnóstico (Linux)

| Comando | Para quê | Exemplo |
|---|---|---|
| `ping` | Alcança o host? (ICMP) | `ping 8.8.8.8` |
| `traceroute` | Caminho até o destino | `traceroute google.com` |
| `nslookup` / `dig` | Resolver DNS | `dig www.site.com` |
| `curl -I` | Testar HTTP e ver cabeçalhos | `curl -I http://10.0.1.50` |
| `nc -zv` | Porta aberta? (telnet moderno) | `nc -zv 10.0.2.30 3306` |
| `ip a` | Meus IPs/interfaces | `ip a` |
| `ip route` | Minha tabela de rotas | `ip route` |
| `ss -tulpn` | Portas em escuta na máquina | `sudo ss -tulpn` |
| `cat /etc/resolv.conf` | Qual DNS estou usando | — |

### Sequência de diagnóstico clássica ("de dentro para fora")

```bash
ip a                        # 1. Tenho IP?
ip route                    # 2. Tenho rota default?
ping 10.0.1.1               # 3. Alcanço o gateway?
ping 8.8.8.8                # 4. Alcanço a internet por IP?
dig google.com              # 5. DNS resolve?
curl -I https://google.com  # 6. Aplicação funciona?
```

Se o passo 4 funciona e o 5 falha → **problema é DNS**, não conectividade.

---

## 10. Mini-exercícios (respostas no fim)

1. Quantos IPs utilizáveis tem um `/27` na AWS?
2. `172.20.0.0/16` é IP privado?
3. Divida `10.10.0.0/24` em 2 sub-redes. Quais os CIDRs?
4. Cliente acessa `ping 8.8.8.8` OK, mas `ping google.com` falha. Qual a causa provável?
5. Qual registro DNS uso para apontar a **raiz** `minhaloja.com.br` para um Load Balancer da AWS?
6. Que porta libero no firewall para um servidor MySQL?

<details>
<summary>✅ Respostas</summary>

1. 32 − 5 = **27 IPs**
2. Sim — está dentro de `172.16.0.0/12` (172.16 a 172.31)
3. `10.10.0.0/25` (.0–.127) e `10.10.0.128/25` (.128–.255)
4. **DNS** (conectividade IP está OK)
5. **Alias** (CNAME não pode na raiz do domínio)
6. **3306/TCP**
</details>
