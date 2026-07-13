# 05 — Armazenamento: S3, EBS e EFS

---

## 1. Comparativo (pergunta certa de prova)

| | **S3** | **EBS** | **EFS** |
|---|---|---|---|
| Tipo | Objetos (arquivos via API/HTTP) | Bloco (disco) | Arquivos (NFS) |
| Conecta em | Qualquer lugar (internet/endpoint) | **1 instância por vez**, mesma AZ | **Várias instâncias ao mesmo tempo** (Linux) |
| Escopo | Regional (nome global) | 1 AZ | Multi-AZ |
| Caso de uso | Site estático, backup, mídia, data lake | Disco do servidor, banco | Pasta compartilhada entre servidores web |
| Analogia | Google Drive | HD/SSD do PC | Servidor de arquivos da empresa |

**Frases prontas:** *"Compartilhar arquivos entre várias EC2"* → **EFS**. *"Disco do sistema"* → **EBS**. *"Hospedar imagens/site estático/backup"* → **S3**.

---

## 2. S3 — Simple Storage Service

### Regras básicas

- **Bucket:** nome **globalmente único**, 3–63 caracteres, minúsculas, números e hífen (sem `_`, sem maiúscula).
- Objetos até **5 TB**; "pastas" são só **prefixos** no nome da chave (`fotos/2026/foto.jpg`).
- Durabilidade 99,999999999% (11 noves) — dado replicado em ≥3 AZs.
- Por padrão, **tudo é privado** (Block Public Access ligado).

### Classes de armazenamento

| Classe | Quando usar |
|---|---|
| **Standard** | Acesso frequente (padrão) |
| **Intelligent-Tiering** | Não sei o padrão de acesso — a AWS move sozinha |
| **Standard-IA** | Acesso raro, mas recuperação imediata (backup) |
| **One Zone-IA** | Raro + recriável (fica em 1 AZ só) |
| **Glacier Instant / Flexible / Deep Archive** | Arquivamento (Deep = mais barato, recuperação em horas) |

### Operações no console

- **Criar:** S3 → Create bucket → nome + região → Create.
- **Upload:** entrar no bucket → Upload → arrastar arquivos.
- **Compartilhar temporariamente:** objeto → **Share with a presigned URL** → validade → gera link temporário (ótimo p/ arquivo privado).
- **Versioning:** aba Properties → Bucket Versioning → Enable. ⚠️ Depois de ligado, **não desliga** (só suspende). "Deletar" cria um *delete marker* — o arquivo antigo continua lá (e cobrando).
- **Lifecycle:** aba Management → Create lifecycle rule → ex.: mover p/ Standard-IA após 30 dias, expirar após 365.

### Hospedar site estático (clássico absoluto ⭐)

1. Crie o bucket (ex.: `ws-site-aluno`), envie `index.html` (e `error.html`).
2. **Properties → Static website hosting → Enable** → Index document: `index.html` → Save. Anote o **endpoint** (`http://bucket.s3-website-sa-east-1.amazonaws.com`).
3. **Permissions → Block public access → Edit → desmarcar tudo** → confirmar.
4. **Permissions → Bucket policy → Edit** e colar (troque o nome!):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadGetObject",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::ws-site-aluno/*"
  }]
}
```

5. Abra o endpoint. **Use o endpoint de website** (que tem `s3-website` no meio), não a URL do objeto.

> Site no S3 é **só HTTP**. Quer HTTPS + domínio? CloudFront na frente (arquivo 08).

### Segurança S3 — ordem de verificação de erro 403

1. **Block Public Access** (do bucket **e** da conta) desligado?
2. **Bucket policy** permite `s3:GetObject` no `Resource` com `/*`?
3. Objeto pertence ao dono do bucket? (Object Ownership → *Bucket owner enforced* resolve)
4. Se acesso via IAM: a política do usuário/role permite?

### Extras que caem

- **Criptografia:** SSE-S3 é padrão automático em objetos novos; SSE-KMS quando pedirem chave gerenciada por você.
- **Replicação (CRR/SRR):** Management → Replication rule; exige **versioning ligado nos dois buckets**.
- **CLI (no CloudShell):**

```bash
aws s3 mb s3://ws-site-aluno
aws s3 cp index.html s3://ws-site-aluno/
aws s3 sync ./site s3://ws-site-aluno/
aws s3 ls s3://ws-site-aluno
```

---

## 3. EFS — Elastic File System

Sistema de arquivos NFS compartilhado (Linux). Cresce sozinho, paga pelo uso.

### Criar e montar (passo a passo)

1. **EFS → Create file system** → escolha a VPC → Create (o modo custom permite definir SG dos *mount targets*).
2. Garanta que o **SG do mount target libera a porta 2049 (NFS)** a partir do SG das EC2. ⚠️ Causa nº 1 de "mount travado".
3. Em cada EC2:

```bash
sudo dnf install -y amazon-efs-utils
sudo mkdir /dados
sudo mount -t efs fs-0abc123:/ /dados
echo "teste" | sudo tee /dados/arquivo.txt   # aparece nas outras instâncias!
```

4. Persistir no boot → `/etc/fstab`:
```
fs-0abc123:/ /dados efs _netdev,tls 0 0
```

Uso clássico de prova: `/var/www/html` compartilhado entre instâncias de um Auto Scaling Group.

---

## 4. Dúvidas comuns (FAQ)

**"O nome do bucket já existe, mas não vejo ele na minha conta!"**
Nomes são globais entre TODAS as contas do mundo. Escolha outro (adicione sufixo: `-eduardo-2026`).

**"Fiz upload mas a URL dá AccessDenied."**
A URL de objeto exige permissão pública (policy) OU use presigned URL. Para site: use o **endpoint de website** + policy pública + Block Public Access desligado.

**"Ativei versioning e a conta subiu."**
Cada versão antiga ocupa (e cobra) espaço. Crie lifecycle para expirar *noncurrent versions*.

**"Excluí o arquivo mas ele 'volta'."**
Com versioning, delete cria um marcador. Para sumir de vez: mostrar versões (*Show versions*) e deletar a versão específica.

**"EBS serve para duas instâncias?"**
Regra prática: **não** (1 volume ↔ 1 instância, mesma AZ). Compartilhamento = **EFS** (Linux) ou FSx (Windows). *(Existe o modo raro io1/io2 Multi-Attach, mas não conte com ele em prova.)*

**"S3 é global ou regional?"**
O **nome** é global; **os dados moram na região** que você escolheu ao criar. Latência e requisitos legais dependem disso.
