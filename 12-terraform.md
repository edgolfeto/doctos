# 12 — Terraform: Infraestrutura como Código (IaC) 🏗️

> **Por que estudar isso?** Nas provas de Cloud Computing é comum receber (ou ter que escrever) infraestrutura como código. Mesmo que a prova cobre só o Console, **saber LER Terraform** te permite entender exatamente o que uma arquitetura faz — e reproduzi-la no painel.

---

## 1. O que é Terraform

| Conceito | 1 linha |
|---|---|
| **IaC** | Descrever a infraestrutura em **arquivos de texto** em vez de clicar no console |
| **Terraform** | Ferramenta da HashiCorp que lê arquivos `.tf` e cria/altera/destrói recursos na nuvem |
| **HCL** | A linguagem dos arquivos `.tf` (HashiCorp Configuration Language) |
| **Declarativo** | Você descreve o **estado final** ("quero 1 VPC e 1 EC2"); o Terraform descobre a ordem e os cliques |
| **State** | Arquivo `terraform.tfstate` onde ele anota tudo que criou (a "memória" dele) |

**Terraform vs CloudFormation** (o outro IaC, nativo da AWS — ver arquivo 10):

| | Terraform | CloudFormation |
|---|---|---|
| Dono | HashiCorp (multi-nuvem) | AWS (só AWS) |
| Linguagem | HCL (`.tf`) | YAML/JSON |
| Onde roda | Sua máquina/CloudShell | Dentro da AWS (Stacks) |
| Memória | `terraform.tfstate` | A própria Stack |

---

## 2. Instalação rápida

**No CloudShell da AWS (jeito mais rápido na prova, nada para configurar):**

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo dnf install -y terraform
terraform -version
```

**No Windows:** baixe o `.exe` em <https://developer.hashicorp.com/terraform/install>, coloque numa pasta do `PATH`. Depois rode `aws configure` (precisa do AWS CLI com uma access key) — no CloudShell isso não é necessário, pois ele já está autenticado. ✅

> 🎓 **AWS Academy / Learner Lab:** as credenciais ficam no botão **AWS Details → AWS CLI**. Cole o conteúdo em `~/.aws/credentials`. A região é limitada a `us-east-1`.

---

## 3. O fluxo de trabalho (decore! ⭐)

```
escrever main.tf
      │
      ▼
terraform init      ← baixa o "plugin" da AWS (1ª vez na pasta)
      │
      ▼
terraform fmt       ← arruma a formatação (opcional)
terraform validate  ← confere a sintaxe (opcional)
      │
      ▼
terraform plan      ← MOSTRA o que vai fazer (não cria nada!)
      │
      ▼
terraform apply     ← cria de verdade (digite "yes")
      │
      ▼
terraform destroy   ← apaga TUDO que ele criou (fim do lab 💰)
```

| Símbolo no `plan` | Significado |
|---|---|
| `+` | vai **criar** |
| `~` | vai **alterar** no lugar |
| `-/+` | vai **destruir e recriar** (cuidado!) |
| `-` | vai **destruir** |

---

## 4. O PROJETO COMPLETO — `main.tf` 📄

O código abaixo cria, sozinho, a arquitetura que você montou "na mão" nos capítulos 03 e 04:

```
Internet ──► IGW ──► Tabela de rotas pública
                          │
        ┌─────────────────┴──────────────────┐
        │ Sub-rede pública A     Sub-rede pública B
        │ 10.0.1.0/24            10.0.2.0/24
        │ [EC2 web + Apache]
        └──────────── VPC 10.0.0.0/16 ────────┘
```

Salve tudo num arquivo chamado **`main.tf`** dentro de uma pasta vazia (ex.: `lab-terraform/`):

```hcl
############################################
# BLOCO 1 — Configurações do Terraform
############################################
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

############################################
# BLOCO 2 — Provider (conexão com a AWS)
############################################
provider "aws" {
  region = var.regiao

  # Etiquetas aplicadas automaticamente a TODOS os recursos
  default_tags {
    tags = local.tags_comuns
  }
}

############################################
# BLOCO 3 — Variáveis (os "campos do formulário")
############################################
variable "regiao" {
  description = "Região AWS (Learner Lab exige us-east-1)"
  type        = string
  default     = "us-east-1"
}

variable "projeto" {
  description = "Prefixo usado no nome de todos os recursos"
  type        = string
  default     = "ws"
}

variable "vpc_cidr" {
  description = "Faixa de IPs da VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "tipo_instancia" {
  description = "Tipo da instância EC2 (t2.micro = nível gratuito)"
  type        = string
  default     = "t2.micro"
}

variable "ip_liberado_ssh" {
  description = "Seu IP público com /32 (ex.: 200.100.50.25/32)"
  type        = string
  default     = "0.0.0.0/0" # ⚠️ na prova/vida real, troque pelo SEU IP
}

############################################
# BLOCO 4 — Locals (valores calculados/reaproveitados)
############################################
locals {
  tags_comuns = {
    Projeto      = var.projeto
    GerenciadoPor = "terraform"
  }
}

############################################
# BLOCO 5 — Data sources (CONSULTAS, não criam nada)
############################################
# Lista as AZs disponíveis na região escolhida
data "aws_availability_zones" "disponiveis" {
  state = "available"
}

# Descobre a AMI mais recente do Amazon Linux 2023
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}

############################################
# BLOCO 6 — VPC
############################################
resource "aws_vpc" "principal" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = { Name = "${var.projeto}-vpc" }
}

############################################
# BLOCO 7 — Sub-redes públicas (uma por AZ)
############################################
resource "aws_subnet" "publica_a" {
  vpc_id                  = aws_vpc.principal.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = data.aws_availability_zones.disponiveis.names[0]
  map_public_ip_on_launch = true # ← o famoso "auto-assign public IP"

  tags = { Name = "${var.projeto}-subnet-publica-a" }
}

resource "aws_subnet" "publica_b" {
  vpc_id                  = aws_vpc.principal.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = data.aws_availability_zones.disponiveis.names[1]
  map_public_ip_on_launch = true

  tags = { Name = "${var.projeto}-subnet-publica-b" }
}

############################################
# BLOCO 8 — Internet Gateway
############################################
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.principal.id # ← já cria E anexa na VPC

  tags = { Name = "${var.projeto}-igw" }
}

############################################
# BLOCO 9 — Tabela de rotas pública + associações
############################################
resource "aws_route_table" "publica" {
  vpc_id = aws_vpc.principal.id

  route {
    cidr_block = "0.0.0.0/0"                     # todo o resto da internet...
    gateway_id = aws_internet_gateway.igw.id      # ...sai pelo IGW
  }

  tags = { Name = "${var.projeto}-rt-publica" }
}

resource "aws_route_table_association" "assoc_a" {
  subnet_id      = aws_subnet.publica_a.id
  route_table_id = aws_route_table.publica.id
}

resource "aws_route_table_association" "assoc_b" {
  subnet_id      = aws_subnet.publica_b.id
  route_table_id = aws_route_table.publica.id
}

############################################
# BLOCO 10 — Security Group (firewall da instância)
############################################
resource "aws_security_group" "web" {
  name        = "${var.projeto}-sg-web"
  description = "HTTP para todos e SSH apenas para meu IP"
  vpc_id      = aws_vpc.principal.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.ip_liberado_ssh]
  }

  egress {
    description = "Toda saida liberada"
    from_port   = 0
    to_port     = 0
    protocol    = "-1" # -1 = todos os protocolos
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.projeto}-sg-web" }
}

############################################
# BLOCO 11 — Instância EC2 com Apache (user data)
############################################
resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.tipo_instancia
  subnet_id              = aws_subnet.publica_a.id
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOF
    #!/bin/bash
    dnf install -y httpd
    systemctl enable --now httpd
    TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
      -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
    AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
      http://169.254.169.254/latest/meta-data/placement/availability-zone)
    echo "<h1>Servidor no ar via Terraform! 🚀</h1><p>AZ: $AZ</p>" \
      > /var/www/html/index.html
  EOF

  tags = { Name = "${var.projeto}-web-01" }
}

############################################
# BLOCO 12 — Outputs (o que aparece no final do apply)
############################################
output "id_da_vpc" {
  description = "ID da VPC criada"
  value       = aws_vpc.principal.id
}

output "ip_publico" {
  description = "IP publico da instancia web"
  value       = aws_instance.web.public_ip
}

output "url_do_site" {
  description = "Abra no navegador apos o apply"
  value       = "http://${aws_instance.web.public_ip}"
}
```

**Para rodar:**

```bash
cd lab-terraform
terraform init
terraform plan     # confira: "Plan: 10 to add"
terraform apply    # digite yes e aguarde ~2 min
# abra a url_do_site no navegador 🎉
terraform destroy  # SEMPRE ao terminar o lab 💰
```

---

## 5. Anatomia de um bloco (a gramática do HCL)

```hcl
resource "aws_subnet" "publica_a" {
   │          │             │
   │          │             └─ APELIDO local (só existe no seu código;
   │          │                é como você referencia esse recurso)
   │          └─ TIPO do recurso na AWS (a "tela" do console)
   └─ TIPO de bloco (resource, data, variable, output, provider...)

  vpc_id = aws_vpc.principal.id
   │            │
   │            └─ REFERÊNCIA: tipo.apelido.atributo
   └─ ARGUMENTO (o "campo do formulário" daquela tela)
}
```

**Regras de ouro da sintaxe:**

- `argumento = valor` → texto entre `"aspas"`, números sem aspas, listas entre `[ ]`, mapas entre `{ }`.
- `var.nome` → lê uma variável. `local.nome` → lê um local. `data.tipo.apelido.atributo` → lê uma consulta.
- `"${var.projeto}-vpc"` → **interpolação**: cola o valor da variável dentro do texto (resultado: `ws-vpc`).
- `#` inicia comentário.
- ⚠️ Dentro do `user_data`, `$TOKEN` (shell) funciona normal, mas **`${...}` é interpretado pelo Terraform**. Se precisar de `${VAR}` do shell, escreva `$${VAR}`.

**Como o Terraform sabe a ORDEM de criação?** Pelas referências! Quando a sub-rede usa `aws_vpc.principal.id`, ele entende: *"preciso da VPC antes da sub-rede"*. Ele monta um grafo de dependências e cria tudo na ordem certa (e destrói na ordem inversa). **No console, o grafo de dependências é você.** 😄

---

## 6. Explicação bloco a bloco 🔍

| Bloco | O que faz | Detalhes que caem em prova |
|---|---|---|
| **1 `terraform {}`** | Exige versão mínima do Terraform e declara o **provider** AWS (plugin baixado no `init`, fica na pasta oculta `.terraform/`) | `~> 5.0` = aceita 5.x, recusa 6.0 |
| **2 `provider "aws"`** | Define **em qual região** tudo será criado + `default_tags` etiqueta todos os recursos automaticamente | Equivale a escolher a região no canto superior direito do console **antes** de clicar em qualquer coisa |
| **3 `variable`** | Entradas do código, com `default`. Sobrescreva sem editar o arquivo: `terraform apply -var="tipo_instancia=t3.micro"` | São os "campos que você digitaria no formulário" |
| **4 `locals`** | Valores calculados/reaproveitados internamente (não dá para sobrescrever por fora) | Aqui, o mapa de tags comum |
| **5 `data`** | **Consulta** coisas que já existem, não cria nada. `aws_ami` acha a AMI mais recente do AL2023 (AMI muda por região — isso evita ID quebrado!); `aws_availability_zones` lista as AZs | `names[0]` e `names[1]` = primeira e segunda AZ da região |
| **6 `aws_vpc`** | Cria a VPC com o CIDR da variável. Os dois `enable_dns_*` ligam a resolução de nomes interna (RDS e endpoints dependem disso) | `tags = { Name = ... }` é o nome que aparece no console |
| **7 `aws_subnet` ×2** | Uma sub-rede /24 em cada AZ. `map_public_ip_on_launch = true` é o checkbox **"Enable auto-assign public IPv4"** — sem ele a instância nasce sem IP público (erro clássico! cap. 03) | Sub-rede pertence a **uma** AZ |
| **8 `aws_internet_gateway`** | Cria o IGW **já anexado** à VPC (o argumento `vpc_id` faz o attach) | No console são 2 passos: Create + Attach |
| **9 `aws_route_table` + `association`** | Tabela com a rota `0.0.0.0/0 → IGW` (isso é o que **torna a sub-rede pública**) e a associação dela às duas sub-redes | Sem a association, a sub-rede fica na tabela *main* (sem internet) |
| **10 `aws_security_group`** | Firewall stateful: entra 80 de qualquer lugar, 22 só do seu IP; sai tudo (`protocol = "-1"`) | `ingress`/`egress` são **sub-blocos** repetíveis |
| **11 `aws_instance`** | A EC2: AMI vem do `data`, entra na sub-rede A, usa o SG, e o `user_data` (script do 1º boot, cap. 04) instala o Apache e mostra a AZ via metadados IMDSv2 | `<<-EOF ... EOF` = texto de várias linhas (heredoc) |
| **12 `output`** | Valores impressos ao final do `apply` (reveja com `terraform output`) | É o "você copiando o IP público na tela do console" |

---

## 7. E se eu quiser fazer TUDO ISSO no Console? 🖱️

É exatamente o mesmo processo dos capítulos 03 e 04 — o Terraform só automatiza os cliques. Tabela de tradução:

| Bloco no código | Onde fazer no Console | Manual |
|---|---|---|
| `terraform {}` / `provider` | Fazer login + **escolher a região** no canto superior direito | 02 |
| `variable` / `locals` | Não existem — são os valores que você digita nos formulários | — |
| `data aws_ami` | EC2 → Launch instance → Quick Start → **Amazon Linux 2023** (o console já seleciona a AMI mais recente) | 04 |
| `data aws_availability_zones` | O dropdown de AZ no formulário da sub-rede | 03 |
| `aws_vpc` | VPC → Your VPCs → **Create VPC** → *VPC only* → CIDR `10.0.0.0/16` → depois Actions → **Edit VPC settings** → marcar *Enable DNS hostnames* | 03 §2 |
| `aws_subnet` ×2 | VPC → Subnets → **Create subnet** (escolha a VPC, AZ e CIDR de cada) → depois selecione cada uma → Actions → **Edit subnet settings** → ✅ *Enable auto-assign public IPv4* | 03 §2 |
| `aws_internet_gateway` | VPC → Internet gateways → **Create** → Actions → **Attach to VPC** (2 passos!) | 03 §2 |
| `aws_route_table` + `route` | VPC → Route tables → **Create** (na VPC) → aba Routes → **Edit routes** → Add `0.0.0.0/0` → target *Internet Gateway* | 03 §3 |
| `aws_route_table_association` ×2 | Na mesma tabela → aba **Subnet associations** → Edit → marcar as 2 sub-redes | 03 §3 |
| `aws_security_group` | EC2 → Security Groups → **Create**: inbound HTTP 80 *Anywhere* + SSH 22 *My IP*; outbound deixa o padrão (all) | 03 §4 |
| `aws_instance` + `user_data` | EC2 → **Launch instance**: nome `ws-web-01`, AMI AL2023, `t2.micro`, *Proceed without key pair* (acesse via Instance Connect), **Network settings → Edit**: selecionar SUA VPC, sub-rede A, SG existente `ws-sg-web` → **Advanced details → User data**: colar o mesmo script bash | 04 §3 |
| `output` | Você mesmo: EC2 → Instances → copiar o **Public IPv4 address** e abrir `http://IP` | 04 |

**Ordem no console (siga a mesma do Terraform):**
1️⃣ Região → 2️⃣ VPC → 3️⃣ Sub-redes A e B (+ auto-assign IP) → 4️⃣ IGW (criar **e** anexar) → 5️⃣ Tabela de rotas (+ rota 0.0.0.0/0 → IGW) → 6️⃣ Associar as sub-redes → 7️⃣ Security Group → 8️⃣ Lançar a EC2 com user data → 9️⃣ Testar `http://IP-público` no navegador.

**Para APAGAR no console** é na **ordem inversa** (EC2 → SG → desassociar/excluir rotas → desanexar e excluir IGW → sub-redes → VPC). O `terraform destroy` faz essa ordem inversa sozinho — mais uma vantagem do IaC. 💡

---

## 8. O arquivo de estado (`terraform.tfstate`) ⚠️

- É a **memória** do Terraform: lista de tudo que ele criou e os IDs reais.
- **Nunca edite na mão. Nunca delete.** Se deletar, o Terraform "esquece" os recursos — eles continuam existindo (e cobrando 💰), mas órfãos.
- **Não misture console + Terraform no mesmo recurso**: se você alterar no console algo criado pelo TF, o próximo `plan` mostra a diferença e tenta "consertar" de volta.
- Contém dados sensíveis → **não suba o `.tfstate` para o GitHub** (crie um `.gitignore` com `*.tfstate*` e `.terraform/`).

---

## 9. Erros comuns e soluções 🚑

| Erro | Causa | Solução |
|---|---|---|
| `No valid credential sources found` | AWS CLI sem credenciais | `aws configure` (ou rode no CloudShell; no Learner Lab, copie de *AWS Details*) |
| `UnauthorizedOperation` | Usuário/role sem permissão | Ajustar política IAM (cap. 02) |
| Recursos "sumiram" no console | Região errada selecionada no painel | Conferir a região da `var.regiao` no canto superior direito |
| `InvalidAMIID.NotFound` | AMI fixa de outra região | Use o `data "aws_ami"` (como no nosso código) |
| `Error acquiring the state lock` | Outro `apply` travado/rodando | Espere; em último caso `terraform force-unlock <ID>` |
| `plan` quer recriar algo que você mudou no console | Drift (mundo real ≠ código) | Volte a mudança no console **ou** atualize o código |
| Fatura após o lab | Esqueceu o `destroy` | `terraform destroy` + conferir no console (cap. 10, checklist de limpeza) |

---

## 10. Kit de comandos 🧰

| Comando | Para quê |
|---|---|
| `terraform init` | Baixa providers (1ª vez / novo provider) |
| `terraform fmt` | Formata o código |
| `terraform validate` | Checa a sintaxe |
| `terraform plan` | Prévia (não altera nada) |
| `terraform apply` | Executa (pede `yes`) |
| `terraform apply -auto-approve` | Executa sem perguntar (⚠️ prova/lab apenas) |
| `terraform output` | Reexibe os outputs |
| `terraform state list` | Lista o que está sob gestão do TF |
| `terraform show` | Detalha o estado atual |
| `terraform destroy` | Apaga tudo que o TF criou |
| `terraform console` | Testa expressões HCL interativamente |

---

## 11. Como a prova pode cobrar Terraform 🎯

1. **Ler e prever:** "dado este `main.tf`, quantas sub-redes serão criadas? A instância terá IP público?" → treine lendo o código do §4 sem olhar a explicação.
2. **Completar lacunas:** entregam o código com `_____` no lugar de `gateway_id` ou `vpc_id`.
3. **Consertar:** um argumento errado (ex.: rota apontando para o ID errado) — o `terraform plan`/`validate` é seu amigo.
4. **Provisionar:** rodar `init/plan/apply` e provar que o site abre.

---

## 12. Mini-desafios 💪

**1)** Adicione HTTPS (porta 443, aberto para todos) no Security Group e aplique.

<details><summary>Ver solução</summary>

Adicione mais um sub-bloco no `aws_security_group.web`:

```hcl
ingress {
  description = "HTTPS"
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
```

`terraform apply` → o plan mostra `~ update in-place` no SG (não recria nada).
</details>

**2)** Crie uma segunda instância `ws-web-02` na sub-rede B e um output com o IP dela.

<details><summary>Ver solução</summary>

Copie o bloco `aws_instance` com apelido novo, trocando `subnet_id = aws_subnet.publica_b.id` e a tag `Name = "${var.projeto}-web-02"`. Depois:

```hcl
output "ip_publico_web02" {
  value = aws_instance.web02.public_ip
}
```

Plan: `2 to add` (instância + nada mais — o resto é reaproveitado).
</details>

**3)** Rode o projeto com instância `t3.micro` **sem editar o arquivo**.

<details><summary>Ver solução</summary>

```bash
terraform apply -var="tipo_instancia=t3.micro"
```

O plan mostra `~ update in-place` em `instance_type` (a instância para e volta com o novo tipo).
</details>

**4)** Prove que o `destroy` limpou tudo.

<details><summary>Ver solução</summary>

`terraform destroy` (yes) → `terraform state list` deve voltar **vazio** → no console: VPC → Your VPCs (só deve restar a VPC *default*) e EC2 → Instances (instância `terminated`). Zero custo residual. ✅
</details>

---

## ✅ Checklist do capítulo

- [ ] Sei o fluxo `init → plan → apply → destroy` e o que cada símbolo do plan significa
- [ ] Sei ler um bloco: tipo, apelido, argumentos e referências (`tipo.apelido.atributo`)
- [ ] Entendo que `data` consulta e `resource` cria
- [ ] Sei traduzir cada bloco do `main.tf` para os cliques do console (§7)
- [ ] Nunca edito/deleto o `terraform.tfstate` e nunca subo ele pro GitHub
- [ ] Sempre rodo `terraform destroy` no fim do laboratório 💰

**Próximo nível (estudo extra):** `count`/`for_each` (criar N recursos com 1 bloco), dividir o código em `variables.tf`/`outputs.tf`, módulos e state remoto no S3.
