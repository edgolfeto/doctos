# 06 — Bancos de Dados: RDS, Aurora e DynamoDB

---

## 1. Qual banco usar? (decisão em 5 segundos)

| Pedido da prova | Serviço |
|---|---|
| MySQL/PostgreSQL gerenciado, aplicação web tradicional | **RDS** |
| Compatível com MySQL/PG, alta performance/HA nativa AWS | **Aurora** |
| NoSQL chave-valor, latência de ms, serverless, escala infinita | **DynamoDB** |
| Cache em memória (Redis/Memcached) | **ElastiCache** |
| Data warehouse / BI analítico | **Redshift** |

"Gerenciado" = AWS cuida de instalação, patch, backup e failover. Você cuida de schema, queries e usuários do banco.

---

## 2. RDS — Relational Database Service

Engines: MySQL, MariaDB, PostgreSQL, Oracle, SQL Server (e Aurora à parte).

### Multi-AZ × Read Replica (A pergunta nº 1 de prova ⭐)

| | **Multi-AZ** | **Read Replica** |
|---|---|---|
| Objetivo | **Alta disponibilidade** (failover) | **Performance de leitura** |
| Replicação | Síncrona | Assíncrona |
| A cópia atende consultas? | ❌ Não (standby invisível)* | ✅ Sim (só leitura) |
| Failover | Automático (DNS aponta p/ standby) | Manual (dá p/ **promover** a banco independente) |
| Outra região? | Não | ✅ Pode (DR / leitura local) |

\* A variação "Multi-AZ DB **Cluster**" (2 standbys legíveis) existe, mas na dúvida responda o clássico: *standby do Multi-AZ não recebe leitura*.

**Frase pronta:** *"O site caiu quando a AZ caiu"* → faltou **Multi-AZ**. *"O banco está sobrecarregado de SELECTs"* → **Read Replica**.

### Criar um MySQL no console (passo a passo)

**RDS → Databases → Create database:**

1. Standard create → Engine **MySQL**.
2. Templates: **Free tier** (treino) ou Dev/Test.
3. DB instance identifier: `ws-db` | Master username: `admin` | senha forte (anote!).
4. Instance: `db.t3.micro` | Storage: 20 GB gp3.
5. **Connectivity:**
   - VPC `ws` → **DB subnet group** (crie antes em RDS → Subnet groups com as **2 sub-redes privadas**).
   - **Public access: No** (padrão de prova!)
   - SG: `ws-sg-db` (3306 liberado **a partir do SG do app**).
6. Create (leva ~5–10 min). Anote o **Endpoint** (`ws-db.xxxx.sa-east-1.rds.amazonaws.com`).

### Conectar a partir da EC2

```bash
sudo dnf install -y mariadb105   # cliente MySQL (Amazon Linux 2023)
mysql -h ws-db.xxxx.sa-east-1.rds.amazonaws.com -u admin -p
```

```sql
CREATE DATABASE loja;
SHOW DATABASES;
```

> **Sempre use o ENDPOINT (DNS), nunca o IP** — no failover do Multi-AZ o IP muda, o endpoint não.

### Backups e snapshots

- **Backup automático:** diário + logs de transação → restaura **point-in-time**; retenção 1–35 dias.
- **Snapshot manual:** dura até você apagar.
- ⚠️ **Restaurar cria uma NOVA instância** (novo endpoint) — não sobrescreve a atual. Depois aponte a aplicação p/ o novo endpoint (ou renomeie).
- Esqueceu a senha master? **Modify → New master password** (não precisa recriar).

---

## 3. Aurora (resumo de prova)

- Compatível com **MySQL e PostgreSQL**, até ~5x/3x mais rápido; storage replica **6 cópias em 3 AZs** e cresce sozinho.
- Estrutura de **cluster**: 1 instância *writer* + até 15 *readers*.
- **Endpoints:** *Cluster (writer)* p/ escrita, *Reader* p/ balancear leitura — use cada um no lugar certo.
- **Aurora Serverless v2**: capacidade em ACUs que escala sozinha — bom p/ carga imprevisível.

---

## 4. DynamoDB — NoSQL serverless

Sem servidor para gerenciar, latência de milissegundos, escala automática. Modelo **chave-valor/documento**.

### Conceitos

| Termo | Significado |
|---|---|
| **Tabela / Item / Atributo** | ≈ tabela / linha / campo (schema flexível) |
| **Partition key (PK)** | Obrigatória — define a distribuição |
| **Sort key (SK)** | Opcional — ordena itens dentro da mesma PK |
| **Query** | Busca **pela chave** — rápida e barata ✔️ |
| **Scan** | Varre a tabela toda — lenta e cara ❌ |
| **On-demand** | Paga por requisição (imprevisível / labs) ⭐ |
| **Provisioned** | Define RCU/WCU (carga estável) |
| **GSI** | Índice p/ buscar por outro atributo |
| **Streams** | Fluxo de mudanças → dispara Lambda |

### No console

**DynamoDB → Create table** → Table name `ws-clientes` → Partition key `cliente_id` (String) → (opcional Sort key `data_pedido`) → default settings (on-demand) → Create.
Inserir/ver dados: **Explore table items → Create item**.

Exemplo de modelagem (pedidos): PK = `cliente_id`, SK = `pedido_id` → "todos os pedidos do cliente X" é **uma Query**.

---

## 5. ElastiCache (citação rápida)

Cache em memória gerenciado. **Redis** (estruturas ricas, persistência, HA) vs **Memcached** (simples, multi-thread). Padrão de arquitetura: app consulta cache antes do RDS (*cache-aside*) p/ aliviar o banco.

---

## 6. Dúvidas comuns (FAQ)

**"Não conecto no RDS a partir da EC2 (timeout)."**
Na ordem: ① SG do RDS libera **3306 a partir do SG da EC2**? ② mesma VPC (ou peering c/ rotas)? ③ usando o **endpoint**, não IP? ④ RDS *Available*? ⑤ NACL default?

**"Não conecto do meu PC de casa."**
RDS foi criado com **Public access: No** (correto!). Acesse via EC2/bastion ou Session Manager. Tornar público em prova = quase sempre errado.

**"O que acontece no failover do Multi-AZ?"**
AWS promove o standby e atualiza o DNS do endpoint (30s–2min). A aplicação só precisa reconectar — por isso: endpoint, nunca IP.

**"Read Replica serve como backup?"**
Não substitui: replica é assíncrona (replica inclusive o `DROP TABLE` que você fizer 😅). Backup = snapshots + retenção automática.

**"Storage do RDS encheu."**
Modify → aumentar storage (ou ligar *storage autoscaling*). Banco não trava se autoscaling estiver ligado.

**"Query vs Scan no DynamoDB?"**
Query usa a chave (eficiente); Scan lê tudo (evite). Se precisa buscar por outro campo sempre → crie um **GSI**.
