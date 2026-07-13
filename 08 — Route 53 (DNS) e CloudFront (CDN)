# 08 — Route 53 (DNS) e CloudFront (CDN)

---

## 1. Route 53

DNS gerenciado da AWS (serviço **global**). Nome vem da porta 53.

### Estruturas

| Item | O que é |
|---|---|
| **Hosted zone pública** | Zona DNS de um domínio visível na internet |
| **Hosted zone privada** | Resolve nomes **só dentro das VPCs associadas** (ex.: `db.interno.local`) |
| Registro | Entradas A, CNAME, MX etc. (tabela no arquivo 01) |

### Alias vs CNAME (pegadinha garantida ⭐)

| | **Alias (AWS)** | CNAME |
|---|---|---|
| Aponta para | Recursos AWS (ALB, CloudFront, S3 website, API GW) e outros registros da zona | Qualquer nome DNS |
| Funciona na **raiz** (`site.com.br`)? | ✅ **Sim** | ❌ Não |
| Custo da consulta | Grátis | Cobrada |

**Regra:** apontar domínio para recurso AWS → **Alias sempre**. `www` pode ser Alias ou CNAME; a **raiz só aceita Alias**.

### Políticas de roteamento

| Política | Comportamento | Caso de uso |
|---|---|---|
| **Simple** | 1 resposta fixa | Padrão |
| **Weighted** | % do tráfego por registro (70/30) | Teste A/B, migração gradual |
| **Latency** | Responde a região mais rápida p/ o usuário | App multi-região |
| **Failover** | Primário; se health check falhar → secundário | DR (site backup no S3!) |
| **Geolocation** | Por país/continente do usuário | Conteúdo por país, restrições |
| **Multivalue** | Até 8 respostas saudáveis | "Mini load balancing" via DNS |

### Passo a passo comum

**Apontar domínio para o ALB:**
Route 53 → Hosted zones → sua zona → **Create record** → Record name vazio (raiz) ou `www` → Type **A** → ligar **Alias** → *Alias to Application Load Balancer* → região → escolher o ALB → Create.

**Failover para página de manutenção no S3:**
1. Crie um **health check** (Route 53 → Health checks) apontando p/ o endpoint primário.
2. Registro A/Alias **Failover-Primary** → ALB (associando o health check).
3. Registro A/Alias **Failover-Secondary** → bucket S3 (website) com página "em manutenção". ⚠️ O bucket precisa ter **exatamente o nome do domínio** p/ Alias de S3 website funcionar.

**Domínio de fora (Registro.br etc.):** crie a hosted zone na AWS → copie os **4 servidores NS** → cadastre-os no painel do registrador. Propagação: minutos a 48h.

---

## 2. CloudFront

CDN: copia (cacheia) seu conteúdo nas **edge locations** pelo mundo → menos latência, menos carga na origem, HTTPS grátis.

### Conceitos

| Termo | Significado |
|---|---|
| **Distribution** | A "instância" do CloudFront (te dá um domínio `dxxxx.cloudfront.net`) |
| **Origin** | De onde vem o conteúdo: S3, ALB, ou qualquer servidor HTTP |
| **Behavior** | Regras por path (`/img/*` → S3; `/api/*` → ALB) |
| **TTL** | Tempo que o objeto fica no cache da edge |
| **Invalidation** | Forçar a limpeza do cache (`/*` limpa tudo) |
| **OAC** (Origin Access Control) | Deixa o bucket **privado**, mas acessível **só pelo CloudFront** ⭐ (substitui o antigo OAI) |

### Passo a passo: site S3 privado + CloudFront (arquitetura correta ⭐)

1. Bucket com o site — **sem** liberar acesso público, **sem** ativar website hosting (fica tudo privado).
2. **CloudFront → Create distribution** → Origin domain: escolha o bucket (use a opção do bucket S3, não o endpoint website).
3. **Origin access → Origin access control settings** → Create OAC → o console mostra um botão/aviso para **copiar a policy** → cole em *Permissions → Bucket policy* do bucket (dá `s3:GetObject` só ao CloudFront).
4. Viewer protocol policy: **Redirect HTTP to HTTPS**.
5. **Default root object: `index.html`** ⚠️ (sem isso, a raiz dá erro).
6. Create (implanta em ~5 min) → teste `https://dxxxx.cloudfront.net`.

**Domínio próprio no CloudFront:** peça certificado no **ACM na região us-east-1** (regra fixa do CloudFront!) → na distribution: *Alternate domain names (CNAMEs)* + certificado → Route 53 Alias → CloudFront.

**Atualizei o site e não mudou:** cache! → aba **Invalidations → Create → `/*`**.

---

## 3. Dúvidas comuns (FAQ)

**"CNAME na raiz do domínio dá erro."**
Comportamento correto do DNS. Use **Alias** (Route 53) para a raiz.

**"Quanto tempo para o DNS propagar?"**
Depende do **TTL** e dos caches pelo caminho. Registro novo em zona AWS: quase imediato. Troca de NS no registrador: até 24–48h. Teste com `dig +short seu-dominio.com`.

**"CloudFront devolve 403 com origem S3."**
① Policy do OAC colada no bucket? ② `Default root object` definido? ③ O objeto existe com esse nome exato (case-sensitive)? ④ Block Public Access pode ficar **ligado** (o OAC não precisa de público — isso é o certo).

**"Preciso de HTTPS no site do S3."**
Endpoint website do S3 é só HTTP. HTTPS = **CloudFront na frente** (com certificado ACM us-east-1 se for domínio próprio).

**"Health check do Route 53 marca unhealthy, mas o site abre pra mim."**
Os checkers vêm de IPs públicos da AWS → o SG/firewall da origem precisa aceitar tráfego público na porta verificada; confira também o path (precisa retornar 2xx/3xx).

**"CloudFront ou ALB? Qual vem primeiro?"**
São camadas diferentes: CloudFront = cache global na borda; ALB = balanceia entre suas instâncias. Arquitetura completa: **Route 53 → CloudFront → ALB → ASG (EC2) → RDS**.
