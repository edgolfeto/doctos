# 09 — Serverless: Lambda, API Gateway, SQS/SNS e EventBridge

> "Serverless" = servidor existe, mas **não é problema seu**. Você entrega código/config; a AWS cuida de provisionar, escalar e cobrar por uso.

---

## 1. Lambda

Roda **funções** em resposta a **eventos**. Paga por requisição + tempo de execução (ms).

### Limites e fatos de prova

| Item | Valor |
|---|---|
| Tempo máximo de execução | **15 minutos** |
| Memória | 128 MB – 10 GB (CPU cresce junto) |
| Disco temporário | `/tmp` (512 MB – 10 GB, efêmero) |
| Runtimes | Python, Node.js, Java, .NET, Go... |
| Permissões da função | **Execution role** (IAM role) ⭐ |
| Logs | Vão sozinhos para o **CloudWatch Logs** |

**Gatilhos (triggers) clássicos:** API Gateway (HTTP), S3 (arquivo criado), EventBridge (agendamento cron), SQS (fila), DynamoDB Streams, SNS.

### Criar a primeira função (console)

1. **Lambda → Create function** → *Author from scratch* → Nome `ws-ola` → Runtime **Python 3.12** → Create.
2. Cole no editor e **Deploy**:

```python
import json

def lambda_handler(event, context):
    nome = event.get("queryStringParameters", {}).get("nome", "mundo") if event.get("queryStringParameters") else "mundo"
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"mensagem": f"Olá, {nome}!"})
    }
```

3. Botão **Test** → crie um evento de teste `{}` → Invoke → veja resposta e logs.
4. Configuration → General: ajuste **timeout** (padrão é só 3s!) e memória se precisar.

### Exemplo com S3 (padrão de prova)

Trigger: *Add trigger → S3 → bucket → event type: PUT*. A função recebe no `event` o bucket e a key do arquivo criado:

```python
def lambda_handler(event, context):
    for r in event["Records"]:
        print("Novo arquivo:", r["s3"]["bucket"]["name"], r["s3"]["object"]["key"])
```

⚠️ Para a função **ler** o arquivo, a **execution role** precisa de `s3:GetObject` (Configuration → Permissions → role → attach policy). "Access Denied" no Lambda = quase sempre role sem permissão.

---

## 2. API Gateway

Porta de entrada HTTP para o Lambda (ou outros backends): URL pública, rotas, autenticação, throttling.

| | **HTTP API** ⭐ | REST API |
|---|---|---|
| Perfil | Simples, barata, rápida | Completa (API keys, cache, validação) |
| Para prova/labs | **Use esta** | Se pedirem recursos avançados |

### Passo a passo: API → Lambda

1. **API Gateway → Create API → HTTP API → Build**.
2. Add integration: **Lambda** → escolha `ws-ola` → API name `ws-api` → Next.
3. Route: método **GET**, path `/ola` → Next → Stage `$default` (auto-deploy) → Create.
4. Copie a **Invoke URL** e teste: `https://xxxx.execute-api.sa-east-1.amazonaws.com/ola?nome=Aluno`.

**CORS** (front-end no navegador chamando a API): HTTP API → aba **CORS** → Origin `*` (lab), métodos e headers necessários. Erro de CORS aparece **no console do navegador**, não na API.

### Fluxo serverless clássico completo

```
Navegador → API Gateway → Lambda → DynamoDB
```

No Lambda (Python/boto3):

```python
import boto3, json
tabela = boto3.resource("dynamodb").Table("ws-clientes")

def lambda_handler(event, context):
    if event["requestContext"]["http"]["method"] == "POST":
        item = json.loads(event["body"])
        tabela.put_item(Item=item)
        return {"statusCode": 201, "body": json.dumps({"ok": True})}
    resp = tabela.scan(Limit=25)
    return {"statusCode": 200, "body": json.dumps(resp["Items"], default=str)}
```

*(Role precisa de `dynamodb:PutItem`/`Scan` na tabela.)*

---

## 3. SQS, SNS e EventBridge (mensageria)

| | **SQS** (fila) | **SNS** (tópico) | **EventBridge** |
|---|---|---|---|
| Modelo | 1 produtor → **1 consumidor processa** cada msg | 1 publicação → **todos os assinantes** recebem (fan-out) | Barramento de eventos + **agendador (cron)** |
| Uso típico | Desacoplar (pedidos → processador), absorver picos | Notificações (e-mail!), disparar vários sistemas | "Toda 6ª às 18h rode a Lambda X" |
| Detalhes de prova | Standard (pelo menos 1x, ordem não garantida) vs **FIFO** (ordem exata, sufixo `.fifo`); **DLQ** guarda msgs com erro; *visibility timeout* esconde a msg enquanto processa | Assinaturas: e-mail (**confirmar no inbox!**), SMS, Lambda, SQS | Regras por padrão de evento ou schedule |

**Padrão fan-out (cai em prova):** SNS → várias filas SQS → cada sistema consome no seu ritmo.

**SNS por e-mail em 1 minuto:** SNS → Create topic (Standard) `ws-alertas` → Create subscription → Email → **confirmar o e-mail recebido** → usado pelos alarmes do CloudWatch (arquivo 10).

---

## 4. Dúvidas comuns (FAQ)

**"Minha Lambda dá timeout."**
Padrão é **3 segundos**. Configuration → aumente o timeout. Se acessa recurso em VPC (RDS), precisa estar **na VPC** com SG/sub-rede certos — e aí perde internet a menos que haja NAT.

**"AccessDenied dentro da Lambda."**
Execution **role** sem a permissão (S3/DynamoDB/etc.). Não é o seu usuário — é a role da função.

**"Onde vejo os prints/erros?"**
**Monitor → View CloudWatch logs** → log group `/aws/lambda/nome-da-funcao`.

**"A API retorna 500."**
Erro dentro do Lambda (veja os logs) ou resposta fora do formato (`statusCode` + `body` string). 403 = rota/método errado ou autorização.

**"Chamei a API e o navegador bloqueia (CORS)."**
Configure CORS na API **e** garanta que a resposta Lambda inclua os headers quando necessário.

**"Lambda pode substituir a EC2?"**
Para tarefas curtas e orientadas a evento, sim (e sai mais barato). Processos longos (>15 min), estado local, servidores tradicionais → EC2/containers.
