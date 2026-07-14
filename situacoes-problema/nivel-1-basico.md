# 🟢 Nível 1 — Básico

---

## Desafio 1.1 — Portfólio no ar (S3)

**Contexto:** A pizzaria **Bella Massa** quer uma página institucional simples (nome, cardápio, telefone). Sem servidor, custo mínimo.

**Requisitos**
1. Bucket chamado `bellamassa-site-SEUNOME` na região `us-east-1`.
2. Página `index.html` (crie um HTML simples) e uma `error.html`.
3. Hospedagem de site estático habilitada, acessível publicamente pelo **endpoint de website**.

**Critérios de aceitação**
- [ ] Endpoint do site abre no navegador de **qualquer** rede (teste no 4G do celular)
- [ ] URL inexistente (ex.: `/xyz`) mostra a `error.html`
- [ ] Objetos continuam **não listáveis** (acessar a raiz do bucket via URL de API não lista arquivos)

**Entregáveis:** URL do endpoint + print da bucket policy.
**Tempo alvo:** 30 min | **Serviços:** S3

<details><summary>💡 Dicas</summary>

- A ordem que funciona: habilitar *Static website hosting* → desligar *Block Public Access* → colar a *bucket policy* de leitura pública (`s3:GetObject` em `arn:aws:s3:::bucket/*`).
- Endpoint de website ≠ URL do objeto. Procure em Properties → Static website hosting.
</details>

<details><summary>✅ Solução resumida</summary>

Create bucket → Upload dos HTMLs → Properties: Static website hosting (index/error) → Permissions: Block Public Access OFF → Bucket policy pública de GetObject → testar endpoint. A policy permite só `GetObject` (não `ListBucket`), por isso a listagem continua bloqueada. ✔️
</details>

---

## Desafio 1.2 — Primeiro servidor web (EC2 + user data)

**Contexto:** A autopeças **Garagem do Zé** quer uma página "Em breve" servida por um servidor Linux de verdade, que suba **já configurado**, sem ninguém acessar via SSH para instalar nada.

**Requisitos**
1. Instância `t3.micro`, Amazon Linux 2023, nome `ze-web-01`, na VPC default.
2. Apache instalado e iniciado **via user data** (proibido instalar manualmente).
3. Página inicial deve exibir o **hostname ou a AZ** da instância.
4. SG `ze-sg-web`: HTTP liberado para todos; SSH **somente do seu IP**.

**Critérios de aceitação**
- [ ] `http://IP_PUBLICO` abre a página com o hostname/AZ
- [ ] Reprovar: SSH aberto para 0.0.0.0/0
- [ ] `sudo systemctl status httpd` = active (running)

**Entregáveis:** print da página + print das regras do SG + conteúdo do user data.
**Tempo alvo:** 40 min | **Serviços:** EC2, SG

<details><summary>💡 Dicas</summary>

- User data começa com `#!/bin/bash` e roda como root no primeiro boot.
- AZ via metadados: `http://169.254.169.254/latest/meta-data/placement/availability-zone` (IMDSv2 usa token — receita no arquivo 04 do manual).
- Não abriu? `sudo cat /var/log/cloud-init-output.log`.
</details>

<details><summary>✅ Solução resumida</summary>

Launch instance → user data instala `httpd`, habilita com `systemctl enable --now`, gera `/var/www/html/index.html` com `$(hostname)`/AZ dos metadados → SG com 80/0.0.0.0/0 e 22/My IP → validar pelo IP público.
</details>

---

## Desafio 1.3 — Organizando a equipe (IAM)

**Contexto:** A **Clínica Vida** contratou você para organizar os acessos AWS: 2 desenvolvedores (podem mexer em EC2 e S3), 1 financeiro (só visualizar custos/billing) e 1 estagiário (somente leitura geral). Hoje todos usam o root. 😱

**Requisitos**
1. Grupos: `vida-devs`, `vida-financeiro`, `vida-leitura` com políticas gerenciadas adequadas.
2. Usuários: `dev01`, `dev02`, `fin01`, `estag01`, cada um no grupo certo, com acesso ao console e senha temporária.
3. MFA ativado em pelo menos 1 usuário (demonstração).
4. Nenhuma access key criada sem necessidade.

**Critérios de aceitação**
- [ ] Login com `estag01` consegue **ver** instâncias EC2 mas **não** consegue criar/terminar
- [ ] Login com `dev01` cria um bucket com sucesso
- [ ] `fin01` acessa páginas de Billing, mas não consegue criar EC2
- [ ] Print do MFA ativo

**Entregáveis:** prints dos grupos com políticas + teste de negação do estagiário.
**Tempo alvo:** 45 min | **Serviços:** IAM

<details><summary>💡 Dicas</summary>

- Políticas úteis: `AmazonEC2FullAccess` + `AmazonS3FullAccess` (devs), `ReadOnlyAccess` (estagiário), `AWSBillingReadOnlyAccess` (financeiro).
- Para o financeiro enxergar billing, pode ser preciso ativar *IAM user/role access to Billing information* nas configurações da conta (root).
- URL de login: `https://ID-DA-CONTA.signin.aws.amazon.com/console`.
</details>

<details><summary>✅ Solução resumida</summary>

Criar 3 grupos com as managed policies acima → criar 4 usuários com console access marcando o grupo → testar em janela anônima cada perfil → ativar MFA virtual num deles. A prova do "não pode" (AccessDenied do estagiário ao terminar instância) vale tanto quanto o "pode".
</details>

---

## Desafio 1.4 — Backup que se cuida (S3 versioning + lifecycle)

**Contexto:** O escritório **ContaCerta** guarda backups diários de planilhas num bucket. Já sobrescreveram arquivo errado e querem: histórico de versões + economia automática (arquivos antigos custam menos) + limpeza após 1 ano.

**Requisitos**
1. Bucket `contacerta-backup-SEUNOME` com **versionamento** habilitado.
2. Regra de ciclo de vida: objetos vão para **Standard-IA após 30 dias** e **expiram após 365 dias**; versões antigas (noncurrent) expiram após **90 dias**.
3. Demonstrar recuperação: subir `relatorio.txt`, sobrescrever com conteúdo diferente, e **recuperar a versão original**.

**Critérios de aceitação**
- [ ] Print de *Show versions* exibindo 2+ versões do `relatorio.txt`
- [ ] Versão original recuperada (conteúdo conferido)
- [ ] Regra de lifecycle visível com as 3 transições/expirações

**Entregáveis:** prints das versões, da regra e do arquivo restaurado.
**Tempo alvo:** 30 min | **Serviços:** S3

<details><summary>💡 Dicas</summary>

- Recuperar versão antiga: com *Show versions* ligado, você pode baixar a versão específica ou **deletar o delete marker/versão nova** para a antiga voltar a ser a atual.
- Lifecycle: aba **Management** do bucket → *Create lifecycle rule* → marque ações para *current* e *noncurrent versions*.
</details>

<details><summary>✅ Solução resumida</summary>

Criar bucket com versioning ON → upload v1, upload v2 (mesmo nome) → Show versions mostra as duas → baixar/restaurar v1 → criar lifecycle com transição 30d (Standard-IA), expiração 365d e *noncurrent* 90d.
</details>
