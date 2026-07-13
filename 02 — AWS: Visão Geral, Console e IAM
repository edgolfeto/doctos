# 02 — AWS: Visão Geral, Console e IAM

---

## 1. Infraestrutura global

| Conceito | Definição | Exemplo |
|---|---|---|
| **Região** | Área geográfica com datacenters independentes | `sa-east-1` (São Paulo), `us-east-1` (N. Virgínia) |
| **AZ (Availability Zone)** | 1+ datacenters isolados dentro da região (energia/rede próprias) | `sa-east-1a`, `sa-east-1b`, `sa-east-1c` |
| **Edge Location** | Ponto de presença para cache (CloudFront) e DNS (Route 53) | Centenas pelo mundo |

**Regras práticas:**
- Alta disponibilidade = recursos em **pelo menos 2 AZs**.
- A maioria dos recursos é **regional** (EC2, VPC, S3*). Exceções **globais**: IAM, Route 53, CloudFront.
- *S3: o nome do bucket é global, mas os dados ficam na região escolhida.
- `us-east-1` é a "região padrão" de muitas provas e a única onde se emitem certificados ACM para CloudFront.

## 2. Modelo de responsabilidade compartilhada

- **AWS cuida DA nuvem:** hardware, datacenter, rede física, hypervisor.
- **Você cuida NA nuvem:** sistema operacional, patches, firewall (SG), dados, IAM, criptografia que você configura.

Pergunta clássica: *"Quem atualiza o SO da EC2?"* → **Você.** *"Quem troca o disco defeituoso?"* → **AWS.**

## 3. Dominando o Console (dicas de competidor)

1. **Barra de busca (Alt+S)** — digite o serviço e vá direto. Não decore menus.
2. **Confira a região** (canto superior direito) ANTES de criar qualquer coisa.
3. **Fixe favoritos** (estrela): EC2, VPC, S3, RDS, IAM, CloudWatch.
4. **CloudShell** (ícone de terminal no topo): shell Linux no navegador, com AWS CLI já autenticada. Perfeito para `aws s3 ls`, testes com `curl`, gerar arquivos.
5. **Tags em tudo**: `Name` é o que aparece nas listas. Provas avaliam por nome/tag.
6. Ícone **"Serviços recentes"** ajuda a navegar rápido entre telas.

---

## 4. IAM — Identity and Access Management

Serviço **global** (não tem região). Controla **quem** pode fazer **o quê** em **qual recurso**.

### Blocos de construção

| Item | O que é | Uso |
|---|---|---|
| **Usuário** | Identidade de uma pessoa/sistema | Login no console, chaves de acesso |
| **Grupo** | Conjunto de usuários | Anexar política 1 vez para vários usuários |
| **Política (Policy)** | Documento JSON de permissões | Diz o que é permitido/negado |
| **Role (Função)** | Identidade **temporária assumível** | EC2 acessar S3 **sem senha/chave** ⭐ |

### Anatomia de uma política JSON

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PermitirLeituraS3",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::meu-bucket",
        "arn:aws:s3:::meu-bucket/*"
      ]
    }
  ]
}
```

- `Effect`: `Allow` ou `Deny`
- `Action`: `serviço:Operação` (curinga: `s3:*`)
- `Resource`: ARN do recurso (`*` = todos)
- Opcional `Condition`: ex. exigir MFA, restringir por IP.

**Regra de avaliação (cai em prova):**
1. Por padrão, tudo é **negado** (implicit deny).
2. Um `Allow` explícito libera.
3. Um **`Deny` explícito SEMPRE vence** qualquer Allow.

### Políticas gerenciadas úteis (AWS managed)

| Política | Dá acesso |
|---|---|
| `AdministratorAccess` | Tudo (use só para admin do lab) |
| `AmazonS3ReadOnlyAccess` | Ler S3 |
| `AmazonEC2FullAccess` | Tudo de EC2 |
| `AmazonSSMManagedInstanceCore` | ⭐ Necessária na **role da EC2** p/ Session Manager |
| `CloudWatchAgentServerPolicy` | EC2 enviar métricas/logs ao CloudWatch |

---

## 5. Passo a passo no Console

### 5.1 Criar grupo + usuário

1. **IAM → User groups → Create group** → nome `ws-devs` → anexe `AmazonS3ReadOnlyAccess` → **Create**.
2. **IAM → Users → Create user** → nome `aluno01`.
3. Marque **Provide user access to the AWS Management Console** → senha customizada → (opcional) exigir troca no 1º login.
4. **Add user to group** → `ws-devs` → **Create user**.
5. Anote a **URL de login da conta** (tem o ID da conta): `https://SEU_ID.signin.aws.amazon.com/console`.

### 5.2 Criar role para EC2 acessar S3 (padrão de prova ⭐)

1. **IAM → Roles → Create role**.
2. Trusted entity: **AWS service** → use case **EC2** → Next.
3. Anexe `AmazonS3ReadOnlyAccess` (e `AmazonSSMManagedInstanceCore` se quiser Session Manager).
4. Nome: `ws-ec2-s3-role` → **Create**.
5. Na EC2: ao lançar, em **Advanced details → IAM instance profile**, escolha a role. (Ou em instância existente: **Actions → Security → Modify IAM role**.)

**Teste dentro da instância:** `aws s3 ls` — funciona **sem** configurar chave nenhuma. Isso é a role em ação.

### 5.3 Ativar MFA

**IAM → Users → aluno01 → Security credentials → Assign MFA device** → Authenticator app → escanear QR com Google Authenticator/Authy → digitar 2 códigos.

---

## 6. Boas práticas (perguntas garantidas)

- ❌ **Nunca** use o usuário **root** no dia a dia (só para billing/fechar conta). Ative MFA nele.
- ❌ **Nunca** coloque access keys dentro de código ou dentro da EC2 → use **roles**.
- ✅ Princípio do **menor privilégio**: dê só o necessário.
- ✅ Grupos para pessoas, roles para serviços.
- ✅ Rotacione/remova chaves antigas.

---

## 7. Dúvidas comuns (FAQ)

**"Usuário ou Role — quando uso cada um?"**
Pessoa que faz login → usuário. Serviço AWS (EC2, Lambda) que precisa acessar outro serviço → **role**.

**"Criei o usuário mas ele não loga."**
Verifique se marcou acesso ao console (usuário pode ser só programático), se está usando a **URL com o ID/alias da conta** e a senha temporária.

**"Anexei Allow mas continua Access Denied."**
Procure um **Deny explícito** em outra política (grupo, SCP do Organizations, boundary) — Deny sempre vence. No Learner Lab da AWS Academy, várias ações de IAM são bloqueadas por design.

**"Onde vejo quem apagou um recurso?"**
**CloudTrail → Event history** (arquivo 10).

**"IAM tem região?"**
Não — usuários, grupos, políticas e roles valem para a conta inteira.

**"O que é ARN?"**
Amazon Resource Name — endereço único de um recurso: `arn:aws:s3:::meu-bucket` ou `arn:aws:ec2:sa-east-1:123456789012:instance/i-0abc...`.
