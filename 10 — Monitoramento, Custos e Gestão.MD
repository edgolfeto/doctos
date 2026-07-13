# 10 — Monitoramento, Custos e Gestão

---

## 1. CloudWatch — olhos da infraestrutura

| Recurso | Função |
|---|---|
| **Metrics** | Números ao longo do tempo (CPU, rede, requisições...) |
| **Alarms** | "Se métrica passar de X por Y minutos → ação" |
| **Logs** | Centraliza logs (Lambda vai sozinho; EC2 precisa de agente) |
| **Dashboards** | Painéis com os gráficos que você escolher |

### Fatos que caem em prova ⭐

- EC2 envia métricas a cada **5 min** (grátis). **Detailed monitoring** = 1 min (pago).
- **CPU, rede e disco (IO)** vêm por padrão. **Memória RAM e espaço em disco NÃO** → precisam do **CloudWatch Agent** instalado na instância (com role `CloudWatchAgentServerPolicy`). Pergunta clássica: *"por que não vejo memória?"*
- Estados do alarme: `OK`, `ALARM`, `INSUFFICIENT_DATA` (normal nos primeiros minutos ou métrica sem dados).

### Alarme de CPU com e-mail (passo a passo)

1. (Pré) Tópico SNS `ws-alertas` com sua inscrição de e-mail **confirmada** (arquivo 09).
2. **CloudWatch → Alarms → Create alarm → Select metric** → EC2 → Per-Instance → sua instância → **CPUUtilization**.
3. Threshold: `Greater than 80` | período 5 min (Static).
4. Notification: estado **In alarm** → tópico `ws-alertas` → Next → nome `ws-alarme-cpu` → Create.
5. Testar: `stress-ng --cpu 4 --timeout 300` na instância → e-mail chega. ✔️

---

## 2. CloudTrail — quem fez o quê

Auditoria de **chamadas de API** na conta (console, CLI, SDK): quem, quando, de onde, o quê.

- **Event history**: últimos **90 dias**, grátis, sem configurar nada.
- Uso clássico: *"quem terminou a instância?"* → CloudTrail → Event history → filtrar **Event name = `TerminateInstances`**.
- Trail (opcional): entrega contínua p/ S3, para guardar além de 90 dias.

**CloudWatch × CloudTrail (pegadinha):** CloudWatch = **desempenho** (métricas/logs da app). CloudTrail = **auditoria** (ações de usuários/API).
*(AWS Config = terceiro irmão: histórico de configuração e conformidade dos recursos.)*

---

## 3. Custos — sobreviva ao lab 💰

### Budget com alerta (faça no dia 1)

**Billing and Cost Management → Budgets → Create budget:**
- Template **Zero spend budget** (avisa em qualquer gasto acima do free tier) **ou** Monthly cost budget com valor (ex.: US$ 5).
- E-mail de alerta → Create. Pronto: nada de susto na fatura.

### Ferramentas

| Ferramenta | Serve para |
|---|---|
| **Cost Explorer** | Gráficos: gasto por serviço/dia — descubra o vilão |
| **Budgets** | Alertas por limite |
| **Free Tier (página)** | Quanto do nível gratuito você já consumiu |

### Checklist de limpeza pós-lab (decore!)

- [ ] Auto Scaling Group (desired/min = 0 ou deletar) — senão as EC2 renascem!
- [ ] Instâncias EC2 → Terminate
- [ ] **Load balancers** e Target groups
- [ ] **NAT Gateways** (vilão nº 1)
- [ ] **RDS** (delete; sem snapshot final se for só treino)
- [ ] **Elastic IPs** → Release
- [ ] Volumes EBS órfãos e snapshots velhos
- [ ] Endpoints Interface, CloudFront (disable → delete)

---

## 4. CloudFormation — infraestrutura como código (noções)

Você descreve os recursos em **YAML/JSON** (template) → a AWS cria tudo como uma **stack**. Deletou a stack = deletou tudo junto (ótimo p/ labs!).

### Template mínimo (EC2 web)

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Servidor web simples
Parameters:
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: ami-0abcdef1234567890   # troque pela AMI da região
      KeyName: !Ref KeyName
      Tags:
        - Key: Name
          Value: ws-cfn-web
Outputs:
  IPPublico:
    Value: !GetAtt WebServer.PublicIp
```

**Console:** CloudFormation → Create stack → *Upload a template file* → Next → parâmetros → Create. Acompanhe na aba **Events** (erros aparecem lá; stack com erro faz **rollback** automático).

Vocabulário: `Parameters` (entradas), `Resources` (obrigatório), `Outputs` (saídas), `!Ref`/`!GetAtt` (referências), **Drift detection** (alguém mexeu na mão?).

---

## 5. Systems Manager (SSM) — canivete suíço

| Recurso | Uso |
|---|---|
| **Session Manager** ⭐ | Terminal na instância **sem porta 22, sem key pair** — inclusive em sub-rede privada. Requisito: role com `AmazonSSMManagedInstanceCore` (+ saída p/ internet ou VPC endpoints) |
| **Parameter Store** | Guardar configs/segredos (ex.: senha do banco) em vez de hardcode |
| **Run Command** | Executar comando em N instâncias de uma vez |

---

## 6. Serviços para citar (1 linha cada)

- **Elastic Beanstalk:** faça upload do código (PHP, Node, Python...) e ele cria EC2+ALB+ASG sozinho — PaaS.
- **ECS / Fargate:** containers Docker gerenciados; Fargate = sem gerenciar servidor.
- **EKS:** Kubernetes gerenciado.
- **Trusted Advisor:** recomendações de custo, segurança e limites.
- **Well-Architected Framework:** 6 pilares (excelência operacional, segurança, confiabilidade, eficiência de performance, otimização de custos, sustentabilidade).
- **AWS Backup:** políticas centralizadas de backup (EBS, RDS, EFS...).
- **Secrets Manager:** segredos com **rotação automática** (vs Parameter Store, mais simples).

---

## 7. Dúvidas comuns (FAQ)

**"Criei o alarme e ficou INSUFFICIENT_DATA."**
Normal nos primeiros minutos. Se persistir: métrica/dimensão errada (instância certa?) ou período maior que a frequência da métrica.

**"O alarme disparou mas não recebi e-mail."**
Inscrição SNS **não confirmada** (olhe o spam) — status precisa ser *Confirmed*.

**"Não vejo memória RAM no CloudWatch."**
Instale o **CloudWatch Agent** (memória/disco não são visíveis pelo hypervisor).

**"Quem deletou meu bucket?!"**
CloudTrail → Event history → Event name `DeleteBucket`.

**"Minha stack do CloudFormation falhou."**
Aba **Events** da stack mostra o recurso e o motivo exato (ex.: AMI inexistente na região, nome de SG duplicado). Corrija o template e recrie.

**"Conta gerou custo e não sei de onde."**
Cost Explorer → agrupar por **Service** → normalmente: NAT Gateway, ALB, RDS, EIP solto ou EBS órfão. Rode o checklist de limpeza.
