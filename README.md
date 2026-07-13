# ☁️ Manual AWS — Preparação WorldSkills Cloud Computing

Material de estudo focado no **Console AWS (painel)**, direto ao ponto, para treinamento de competidor WorldSkills — ocupação Cloud Computing.

> Nas edições recentes, a prova de Cloud Computing da WorldSkills usa **AWS**, em formato de desafios práticos (estilo GameDay/Jam): montar arquiteturas, cumprir requisitos exatos e consertar ambientes quebrados — **contra o relógio**.

---

## 📚 Estrutura do material

| # | Arquivo | Conteúdo |
|---|---------|----------|
| 01 | [Fundamentos de Redes](01-fundamentos-de-redes.md) | OSI, IP, CIDR, sub-redes, rotas, NAT, DNS, portas, comandos |
| 02 | [AWS: Visão Geral e IAM](02-aws-visao-geral-e-iam.md) | Regiões, AZs, console, CloudShell, usuários, políticas, roles |
| 03 | [VPC](03-aws-vpc.md) | Sub-redes, IGW, NAT, tabelas de rota, SG vs NACL, endpoints |
| 04 | [EC2](04-aws-ec2.md) | Instâncias, AMI, key pairs, user data, EBS, metadados |
| 05 | [S3, EBS e EFS](05-aws-s3-ebs-efs.md) | Buckets, site estático, versionamento, comparativo de storage |
| 06 | [Bancos de Dados](06-aws-bancos-de-dados.md) | RDS, Multi-AZ vs Read Replica, Aurora, DynamoDB |
| 07 | [ELB e Auto Scaling](07-aws-elb-e-auto-scaling.md) | ALB/NLB, target groups, health checks, ASG |
| 08 | [Route 53 e CloudFront](08-aws-route53-e-cloudfront.md) | DNS, Alias, políticas de roteamento, CDN, ACM |
| 09 | [Serverless](09-aws-serverless.md) | Lambda, API Gateway, SQS/SNS, EventBridge |
| 10 | [Monitoramento e Gestão](10-aws-monitoramento-e-gestao.md) | CloudWatch, CloudTrail, Budgets, CloudFormation, SSM |
| 11 | [Troubleshooting e FAQ](11-troubleshooting-e-faq.md) | Playbooks de diagnóstico + dúvidas rápidas ⭐ |
| — | [Situações-Problema](situacoes-problema/README.md) | Desafios práticos em 4 níveis, prontos para o GitHub |

---

## 🗺️ Trilha de estudo sugerida (4 semanas)

| Semana | Foco | Prática |
|--------|------|---------|
| 1 | Arquivos 01, 02, 03 | Situações-problema Nível 1 |
| 2 | Arquivos 04, 05, 06 | Situações-problema Nível 2 |
| 3 | Arquivos 07, 08, 09 | Situações-problema Nível 3 |
| 4 | Arquivos 10, 11 (revisão) | Nível 4 (troubleshooting) + simulado cronometrado |

---

## 🏆 Regras de ouro para a competição

1. **Leia a prova inteira antes de começar.** Requisitos escondem dependências (a VPC do item 1 é usada no item 7).
2. **Nomes exatos importam.** Se a prova pede `ws-app-server`, não crie `ws-app-server-1`. Avaliação costuma ser automatizada por nome/tag.
3. **Confira a região** no canto superior direito **antes de criar qualquer recurso**. Recurso na região errada = ponto perdido.
4. **Valide cada entrega** (abrir a URL, testar o SSH, rodar o health check) antes de passar ao próximo item.
5. **Use o CloudShell** para comandos rápidos sem configurar nada.
6. **Cronometre os treinos.** Velocidade com precisão é o que diferencia medalhistas.

---

## 💰 Avisos de custo (IMPORTANTE para treinar)

Serviços que **cobram por hora mesmo parados** — sempre delete após o laboratório:

- **NAT Gateway** (vilão nº 1 de fatura surpresa)
- **Load Balancers** (ALB/NLB)
- **RDS** (instância de banco)
- **Elastic IP não associado** a instância em execução
- Instâncias EC2 fora do nível gratuito

✅ Crie um **Budget de US$ 0,01/zero-spend** com alerta por e-mail no primeiro dia (passo a passo no arquivo 10).
✅ Se o SENAI tiver **AWS Academy (Learner Lab)**, use: vem com créditos e ambiente isolado (com algumas restrições, ex.: região limitada e sem criação de usuários IAM).

---

## 🔗 Links oficiais

- Documentação: <https://docs.aws.amazon.com>
- AWS Skill Builder (cursos grátis): <https://skillbuilder.aws>
- Workshops práticos: <https://workshops.aws>
- WorldSkills: <https://worldskills.org>

---

## 📁 Como publicar no GitHub

```bash
git init
git add .
git commit -m "Material de treino AWS - WorldSkills Cloud Computing"
git branch -M main
git remote add origin https://github.com/SEU-USUARIO/aws-worldskills.git
git push -u origin main
```

As dicas e soluções das situações-problema usam blocos `<details>` — no GitHub elas ficam **recolhidas**, então o competidor só vê se clicar. 😉
