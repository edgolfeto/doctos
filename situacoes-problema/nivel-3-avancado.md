# 🔴 Nível 3 — Avançado

---

## Desafio 3.1 — Sobrevivendo à Black Friday (ALB + Auto Scaling)

**Contexto:** A loja **MegaPromo** caiu na última Black Friday (servidor único). Exigência da diretoria: o site deve **sobreviver à queda de uma AZ inteira** e **crescer sozinho** com a carga.

**Requisitos**
1. Launch template `megapromo-lt` — Amazon Linux 2023, user data com Apache exibindo **hostname + AZ**.
2. Target group `megapromo-tg` (HTTP:80, health check `/`).
3. ALB `megapromo-alb` internet-facing em **2 sub-redes públicas (AZs diferentes)**.
4. ASG `megapromo-asg`: min 2 / desired 2 / max 4, nas 2 AZs, anexado ao target group, **health check ELB ligado**, target tracking CPU 50%.
5. Cadeia de SGs: internet→ALB(80); instâncias aceitam 80 **somente do SG do ALB**.

**Critérios de aceitação**
- [ ] F5 no DNS do ALB alterna entre AZs (prints das 2 respostas)
- [ ] **Teste do caos:** terminar 1 instância na mão → site continua no ar → ASG repõe sozinho (print da aba Activity)
- [ ] **Teste de carga:** stress em uma instância → nasce instância nova (scale-out) → depois some (scale-in)
- [ ] Acesso direto ao IP público da instância **não** abre a página (só via ALB)

**Entregáveis:** DNS do ALB + prints dos testes de caos/carga + SGs.
**Tempo alvo:** 90 min | 💰 **Delete ALB e ASG no final** | **Serviços:** EC2, ELB, ASG

<details><summary>💡 Dicas</summary>

- O critério "só via ALB" depende do SG da instância ter origem = **SG do ALB** (não 0.0.0.0/0).
- Health check reprovando tudo? Path e porta; e o `grace period` do ASG dá tempo do user data terminar.
- Scale-in demora alguns minutos após a CPU baixar — é o comportamento esperado.
</details>

<details><summary>✅ Solução resumida</summary>

Ordem eficiente: SGs → target group → ALB (2 AZs) → launch template → ASG anexado ao TG com ELB health check + target tracking 50%. Validação tripla: balanceamento (F5), auto-healing (terminate), elasticidade (stress-ng). Roteiro detalhado no arquivo 07 do manual.
</details>

---

## Desafio 3.2 — Site global e blindado (CloudFront + S3 privado)

**Contexto:** A **EduTech Cursos** distribui um site estático para alunos no Brasil, Portugal e Angola. Exigências: HTTPS, baixa latência global e **bucket 100% privado** (ninguém acessa o S3 direto).

**Requisitos**
1. Bucket `edutech-site-SEUNOME` com o site — **Block Public Access LIGADO**, sem website hosting.
2. Distribuição CloudFront com o bucket como origem via **OAC**, `index.html` como *default root object*, HTTP→HTTPS.
3. Bucket policy permitindo leitura **apenas** para a distribuição.
4. Demonstrar atualização de conteúdo com **invalidation**.

**Critérios de aceitação**
- [ ] `https://dxxxx.cloudfront.net` abre o site com cadeado
- [ ] URL direta do objeto no S3 retorna **AccessDenied** (print)
- [ ] Alterar o `index.html`, invalidar `/*` e ver a mudança
- [ ] Print da bucket policy com o *principal* do CloudFront

**Entregáveis:** URL CloudFront + prints acima.
**Tempo alvo:** 60 min | **Serviços:** CloudFront, S3

<details><summary>💡 Dicas</summary>

- Ao criar a origem, escolha o bucket (não o endpoint website) e crie o **OAC**; o console fornece a policy pronta para copiar.
- Esqueceu o *default root object* = erro na raiz mesmo com tudo certo.
- Sem invalidation, o cache segura a versão antiga pelo TTL.
</details>

<details><summary>✅ Solução resumida</summary>

Upload no bucket privado → Create distribution + OAC → colar policy no bucket → default root object → aguardar deploy → testar HTTPS ok e S3 direto negado → editar arquivo → Invalidations `/*` → conferir atualização.
</details>

---

## Desafio 3.3 — Mural de recados sem servidor (API serverless)

**Contexto:** O grêmio da escola **ETEC Norte** quer um "mural de recados" com custo praticamente zero quando ninguém usa. Nada de EC2.

**Requisitos**
1. Tabela DynamoDB `etec-recados` — PK `id` (String), modo on-demand.
2. Lambda `etec-api` (Python) com duas operações: **POST** grava recado `{id, autor, texto}`; **GET** lista recados.
3. HTTP API no API Gateway com rotas `GET /recados` e `POST /recados` integradas à Lambda.
4. Execution role com permissão **mínima** (PutItem/Scan **naquela tabela**, não `dynamodb:*` em `*`).

**Critérios de aceitação**
- [ ] `curl -X POST .../recados -d '{"id":"1","autor":"Ana","texto":"Olá!"}'` retorna 201
- [ ] `curl .../recados` devolve o recado em JSON
- [ ] Item visível no console do DynamoDB
- [ ] Print da policy da role restrita ao ARN da tabela

**Entregáveis:** Invoke URL + prints dos curls e da tabela.
**Tempo alvo:** 90 min | **Serviços:** Lambda, API Gateway, DynamoDB, IAM

<details><summary>💡 Dicas</summary>

- Código-base de Lambda com boto3 está no arquivo 09 do manual (adapte nomes).
- Métodos diferentes na mesma função: leia `event["requestContext"]["http"]["method"]`.
- Erro 500? CloudWatch Logs da função conta a verdade (falta de permissão aparece lá como AccessDenied).
</details>

<details><summary>✅ Solução resumida</summary>

Tabela on-demand → Lambda com if método POST (put_item) / GET (scan) → HTTP API com as 2 rotas para a mesma função → editar a role: inline policy com `dynamodb:PutItem` e `dynamodb:Scan` no ARN `.../table/etec-recados` → testar via curl/CloudShell.
</details>

---

## Desafio 3.4 — 🏆 PROVA SIMULADA: Arquitetura completa (3 camadas)

**Contexto:** Você é o arquiteto da **VotaFácil**, app de enquetes que será notícia nacional amanhã. Monte a arquitetura completa de produção. **Cronometre: 150 min.**

**Requisitos (pontuação)**
| # | Item | Pts |
|---|---|---|
| 1 | VPC `votafacil-vpc 10.30.0.0/16`, 2 AZs, 2 públicas + 2 privadas, IGW, NAT | 15 |
| 2 | RDS MySQL Multi-AZ? **Não** — para economizar no treino: single-AZ `votafacil-db` privado, mas **explique por escrito** o que mudaria com Multi-AZ | 15 |
| 3 | Launch template com app (pode ser a página PHP abaixo) + ASG (2–4) **nas privadas** | 20 |
| 4 | ALB público → target group das instâncias privadas | 20 |
| 5 | Cadeia de SGs: ALB←internet:80 / app←ALB:80 / db←app:3306 | 15 |
| 6 | Alarme CPU>70 → SNS e-mail | 10 |
| 7 | Tags `Projeto=VotaFacil` em TODOS os recursos | 5 |

**App de teste (user data, após instalar httpd+php+php-mysqlnd):**
```php
<?php // /var/www/html/index.php
$c = @mysqli_connect("ENDPOINT_RDS","admin","SENHA","mysql");
echo "<h1>VotaFácil - ".gethostname()."</h1>";
echo $c ? "<p>✅ Banco conectado</p>" : "<p>❌ Sem banco: ".mysqli_connect_error()."</p>";
```

**Critérios de aceitação**
- [ ] DNS do ALB abre a página com "✅ Banco conectado", alternando hostname no F5
- [ ] Instâncias **sem IP público**; ainda assim instalaram pacotes (NAT funcionando)
- [ ] Terminar uma instância não derruba o site
- [ ] Texto de 5 linhas: papel do Multi-AZ + o que mais faltaria para produção (HTTPS? CloudFront? Backup?)

**Entregáveis:** prints por item + autoavaliação da pontuação (aprovação: ≥ 80/100).
💰 **Ambiente caro (NAT+ALB+RDS): monte, valide, printe e DESTRUA.**

<details><summary>💡 Dicas</summary>

- Ordem que economiza tempo: VPC completa → SGs (os 3, já referenciados) → RDS (demora ~10 min: dispare cedo!) → template → TG+ALB → ASG → alarme.
- Instância privada sem SSH: valide com Session Manager ou confie no health check + página.
- O erro mais comum aqui: esquecer `php-mysqlnd` ou o SG do banco apontar para o SG errado.
</details>

<details><summary>✅ Gabarito de arquitetura</summary>

Internet → ALB (públicas) → ASG/EC2 (privadas, saída via NAT) → RDS (privadas). SGs em cadeia por referência. Multi-AZ: standby síncrono em outra AZ com failover automático de endpoint — protege o banco da queda de AZ (a camada web já está protegida pelo ASG em 2 AZs). Produção ainda pediria: HTTPS (ACM no ALB), domínio (Route 53 Alias), CloudFront, backups/retention no RDS, CloudTrail e IaC (CloudFormation).
</details>
