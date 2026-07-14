# 🟡 Nível 2 — Intermediário

---

## Desafio 2.1 — A rede da empresa (VPC do zero)

**Contexto:** A software house **CodeBras** vai migrar para a AWS e precisa da fundação de rede, feita **manualmente** (sem o assistente "VPC and more") para provar domínio.

**Requisitos**
1. VPC `codebras-vpc` — CIDR `10.20.0.0/16`.
2. Sub-redes: `pub-a 10.20.1.0/24`, `pub-b 10.20.2.0/24`, `priv-a 10.20.11.0/24`, `priv-b 10.20.12.0/24` (2 AZs diferentes).
3. IGW anexado; tabela `codebras-rt-pub` com rota default para o IGW, associada às públicas.
4. NAT Gateway em `pub-a`; tabela `codebras-rt-priv` com rota default para o NAT, associada às privadas.
5. Auto-assign de IP público **ligado só nas públicas**.

**Critérios de aceitação**
- [ ] EC2 de teste na `pub-a` acessível da internet (HTTP ou ping)
- [ ] EC2 de teste na `priv-a` **sem IP público**, mas `ping 8.8.8.8` funciona (saída via NAT)
- [ ] Da internet, **não** é possível iniciar conexão com a instância privada
- [ ] Diagrama simples (papel/draw.io) com CIDRs e rotas

**Entregáveis:** prints das duas tabelas de rota + testes de conectividade + diagrama.
**Tempo alvo:** 60 min | **Serviços:** VPC, EC2 | 💰 **Delete o NAT no final!**

<details><summary>💡 Dicas</summary>

- Como testar a privada sem SSH público? Use **Session Manager** (role `AmazonSSMManagedInstanceCore` na instância) ou faça o teste 2.2 junto (bastion).
- NAT Gateway: sub-rede **pública** + Elastic IP. A rota `0.0.0.0/0 → nat-...` vai na tabela **das privadas**.
</details>

<details><summary>✅ Solução resumida</summary>

VPC → 4 subnets (atenção às AZs) → IGW attach → RT pública (default→IGW, associar pub-a/pub-b) → habilitar auto-assign nas públicas → NAT em pub-a com EIP → RT privada (default→NAT, associar priv-a/priv-b) → subir 2 EC2s de teste e validar os 3 cenários de conectividade.
</details>

---

## Desafio 2.2 — Cofre com porta única (Bastion host)

**Contexto:** A fintech **PagRápido** exige que servidores de aplicação fiquem em sub-rede privada. O time só pode administrá-los passando por um **bastion** (jump host) na sub-rede pública.

**Requisitos** *(reaproveite a VPC do 2.1)*
1. `pagrapido-bastion` (t3.micro) na sub-rede pública — SG `sg-bastion`: SSH **somente do seu IP**.
2. `pagrapido-app` na sub-rede privada, **sem IP público** — SG `sg-app`: SSH **somente a partir do `sg-bastion`** (referência de SG, não CIDR).
3. Conexão em 2 saltos: você → bastion → app, usando **agent forwarding** (a chave privada não pode ser copiada para o bastion).

**Critérios de aceitação**
- [ ] `ssh` direto da internet para o IP privado da app **falha**
- [ ] Via bastion, o acesso funciona: print do `hostname` da app
- [ ] Prova de que a `.pem` **não está** no bastion (`ls ~/.ssh` limpo)
- [ ] Regra do `sg-app` mostra o **ID do sg-bastion** como origem

**Entregáveis:** prints dos SGs + sessão SSH encadeada.
**Tempo alvo:** 45 min | **Serviços:** EC2, VPC, SG

<details><summary>💡 Dicas</summary>

- Agent forwarding: `ssh -A -i chave.pem ec2-user@IP_BASTION` e de lá `ssh ec2-user@IP_PRIVADO_APP` (no Windows, use `ssh-add` no agente do OpenSSH antes).
- Alternativa moderna para citar na entrevista: `ProxyJump` (`ssh -J`) ou Session Manager (dispensa bastion).
</details>

<details><summary>✅ Solução resumida</summary>

Duas instâncias conforme requisitos → sg-app com inbound 22 tendo **Source = sg-bastion** → `ssh-add chave.pem` → `ssh -A` no bastion → `ssh` interno para o IP privado. O pulo do gato avaliado: origem por SG e ausência da chave no bastion.
</details>

---

## Desafio 2.3 — Blog institucional (WordPress: EC2 + RDS)

**Contexto:** A imobiliária **MoradaBoa** quer um WordPress. Regra da matriz: **o banco não pode morar no mesmo servidor** da aplicação e não pode ser acessível pela internet.

**Requisitos**
1. RDS MySQL `moradaboa-db` (db.t3.micro, 20 GB) nas **sub-redes privadas** (DB subnet group), *Public access: No*.
2. SG do banco: 3306 **somente a partir do SG da instância web**.
3. EC2 `moradaboa-web` (pública) com Apache + PHP + WordPress apontando para o **endpoint** do RDS.
4. Concluir o instalador do WordPress no navegador (site com 1 post publicado).

**Critérios de aceitação**
- [ ] Blog abre pelo IP público e exibe um post
- [ ] `wp-config.php` usa o **endpoint DNS** do RDS (print, pode ocultar a senha)
- [ ] Tentativa de `mysql -h endpoint` **do seu PC** falha (banco não é público)
- [ ] Tentativa a partir da EC2 funciona

**Entregáveis:** URL do blog + prints dos SGs + testes de conexão.
**Tempo alvo:** 90 min | **Serviços:** EC2, RDS, VPC | 💰 **Delete o RDS no final!**

<details><summary>💡 Dicas</summary>

- Instalação (Amazon Linux 2023): `dnf install -y httpd php php-mysqlnd wget`, baixe `https://wordpress.org/latest.tar.gz`, extraia em `/var/www/html`, ajuste dono (`chown -R apache:apache`).
- Crie o database antes: a partir da EC2, `mysql -h ENDPOINT -u admin -p` → `CREATE DATABASE wordpress;`.
- "Error establishing a database connection" = 90% SG do RDS ou endpoint/senha errados.
</details>

<details><summary>✅ Solução resumida</summary>

DB subnet group (privadas) → RDS MySQL não público com sg-db(3306←sg-web) → EC2 pública com stack LAMP via user data ou manual → `CREATE DATABASE` → instalador web do WP com host = endpoint → publicar post. Validar o "não conecto de fora".
</details>

---

## Desafio 2.4 — Vigia 24h (CloudWatch + SNS + Budget)

**Contexto:** Depois de um servidor travado num sábado, o e-commerce **TudoBarato** exige: alerta por e-mail se a CPU passar de 70% e alerta de gastos da conta.

**Requisitos**
1. Tópico SNS `tudobarato-alertas` com sua inscrição de e-mail **confirmada**.
2. Alarme `tudobarato-cpu-alta`: CPUUtilization > 70% por 5 minutos → notifica o tópico.
3. Provocar o alarme com teste de stress e **receber o e-mail**.
4. Budget mensal de US$ 5 com alerta em 80%.

**Critérios de aceitação**
- [ ] Print do alarme em estado **In alarm** (histórico)
- [ ] Print do e-mail recebido
- [ ] Print do budget configurado
- [ ] Alarme volta para **OK** após o stress acabar

**Entregáveis:** prints acima.
**Tempo alvo:** 40 min | **Serviços:** CloudWatch, SNS, Budgets, EC2

<details><summary>💡 Dicas</summary>

- Stress: `sudo dnf install -y stress-ng && stress-ng --cpu 4 --timeout 420`.
- Sem e-mail? A inscrição SNS está *Pending confirmation* (olhe o spam).
- `t3.micro` em modo unlimited pode "aguentar" — 4 workers por 7 min resolvem.
</details>

<details><summary>✅ Solução resumida</summary>

SNS topic + subscription confirmada → CloudWatch alarm na métrica CPUUtilization da instância (>70, 1 datapoint de 5 min) com ação In alarm → SNS → stress-ng → e-mail chega → Budgets template mensal US$ 5/80%.
</details>
