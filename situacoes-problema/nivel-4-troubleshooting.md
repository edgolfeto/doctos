# 🔧 Nível 4 — Troubleshooting (estilo GameDay)

> **Como funciona:** o instrutor prepara e "sabota" o ambiente usando a seção 🔧 (recolhida). O competidor recebe **apenas o chamado do cliente** e precisa diagnosticar, corrigir e explicar a causa raiz.
>
> **Competidor: NÃO abra as seções 🔧 nem ✅.** Pontue-se: causa raiz correta (40%) + correção funcionando (40%) + explicação clara (20%).

---

## Chamado 4.1 — "Ninguém acessa o servidor!"

**Cliente:** *"Nosso administrador saiu da empresa ontem. Hoje ninguém consegue conectar via SSH no servidor `prod-web-01` — dá timeout. O site continua no ar. Precisamos de acesso URGENTE."*

**Missão:** recuperar o acesso SSH, explicar a causa e propor como evitar dependência de uma pessoa só.
**Tempo alvo:** 20 min

<details><summary>🔧 Montagem (instrutor)</summary>

Instância pública com Apache OK. Sabotagem: **remova a regra de entrada da porta 22 do SG** (deixe só a 80). Variante extra: troque a regra 22 para um CIDR aleatório tipo `203.0.113.10/32`.
</details>

<details><summary>💡 Pistas progressivas</summary>

1. Site no ar + SSH timeout = a instância está viva; o problema é **seletivo por porta**.
2. Timeout (e não "permission denied") = pacote nem chega → firewall de rede.
3. Onde vive o firewall por porta na AWS?
</details>

<details><summary>✅ Causa e correção</summary>

Regra de entrada TCP 22 ausente/errada no Security Group. Corrigir: adicionar 22 com origem "My IP". Prevenção: **Session Manager** (sem depender de porta 22/chave pessoal) + acessos por grupo IAM, não por pessoa.
</details>

---

## Chamado 4.2 — "O site caiu de madrugada"

**Cliente:** *"O e-commerce `loja-web` está fora do ar desde as 3h. O navegador fica carregando e dá erro de conexão. Ontem à noite o estagiário 'fez uma manutenção rápida'."*

**Missão:** site no ar novamente + relatório do que o estagiário fez.
**Tempo alvo:** 20 min

<details><summary>🔧 Montagem (instrutor)</summary>

Instância pública com Apache. Sabotagem dupla: `sudo systemctl stop httpd && sudo systemctl disable httpd`. Variante hard: além disso, `sudo mv /var/www/html/index.html /root/`.
</details>

<details><summary>💡 Pistas progressivas</summary>

1. SSH funciona? Então rede OK → o problema é **dentro** da máquina.
2. `curl -I http://localhost` responde? E `sudo ss -tulpn | grep :80`?
3. Se o serviço subir mas der 403/404, cadê o arquivo?
</details>

<details><summary>✅ Causa e correção</summary>

Serviço parado **e desabilitado** (não voltaria nem com reboot): `sudo systemctl enable --now httpd`. Na variante, restaurar o index. Relatório: diferença entre *stop* (agora) e *disable* (no boot) — por isso "reiniciar o servidor" não resolveu.
</details>

---

## Chamado 4.3 — "O servidor interno perdeu a internet"

**Cliente:** *"Nosso servidor de processamento `worker-01` (rede interna, sem IP público — política de segurança) parou de baixar atualizações e não alcança APIs externas. `ping 8.8.8.8` falha. Não mexam na política: ele NÃO pode ter IP público."*

**Missão:** devolver a saída à internet **mantendo a instância privada**.
**Tempo alvo:** 25 min

<details><summary>🔧 Montagem (instrutor)</summary>

VPC com pública+privada, NAT funcionando, worker na privada (com role SSM p/ acesso). Sabotagem (escolha 1): **(a)** apagar a rota `0.0.0.0/0 → NAT` da tabela privada; **(b)** associar a sub-rede privada à tabela de rotas **pública** errada não, melhor: associar à *main* sem default; **(c)** deletar o NAT Gateway.
</details>

<details><summary>💡 Pistas progressivas</summary>

1. Como uma máquina **sem IP público** sai para a internet?
2. Siga o pacote: tabela de rotas da sub-rede → para onde vai `0.0.0.0/0`?
3. O NAT existe? Está *Available*? Está em sub-rede pública?
</details>

<details><summary>✅ Causa e correção</summary>

Caminho privado→NAT→IGW quebrado (rota removida ou NAT ausente). Corrigir a rota default da tabela privada para o NAT (ou recriar o NAT **na pública** + rota). Solução com IP público = **reprovado** (violou a política).
</details>

---

## Chamado 4.4 — "Error establishing a database connection"

**Cliente:** *"Nosso blog WordPress mostra 'Error establishing a database connection' desde hoje cedo. Ninguém alterou senha. O RDS aparece como Available no console."*

**Missão:** blog funcionando + causa raiz documentada.
**Tempo alvo:** 25 min

<details><summary>🔧 Montagem (instrutor)</summary>

Ambiente do desafio 2.3 funcionando. Sabotagem (escolha 1): **(a)** no SG do RDS, trocar a origem da regra 3306 do sg-web para um SG inexistente/aleatório; **(b)** trocar no `wp-config.php` o host para um IP fixo antigo qualquer; **(c)** regra 3306 removida.
</details>

<details><summary>💡 Pistas progressivas</summary>

1. Separe as hipóteses: credencial errada responde rápido ("Access denied for user"); **timeout** = rede.
2. Da EC2: `nc -zv ENDPOINT 3306` — abre?
3. Quem decide se a EC2 pode falar com o RDS na 3306?
</details>

<details><summary>✅ Causa e correção</summary>

Timeout de rede: regra 3306 do SG do banco não referencia mais o SG da web (ou host errado no wp-config). Corrigir a regra origem=sg-web (ou host=endpoint DNS). Lição: **endpoint, nunca IP** + origem por SG.
</details>

---

## Chamado 4.5 — "O balanceador diz 503"

**Cliente:** *"Colocamos um load balancer ontem e hoje o site responde `503 Service Temporarily Unavailable`. As duas instâncias estão 'running' no EC2. O que há?"*

**Missão:** tráfego fluindo pelo ALB + explicação do 503.
**Tempo alvo:** 25 min

<details><summary>🔧 Montagem (instrutor)</summary>

Ambiente do desafio 3.1 (pode ser sem ASG: 2 instâncias registradas). Sabotagem (escolha 1): **(a)** mudar o path do health check para `/status-nao-existe`; **(b)** no SG das instâncias, trocar a origem da 80 do sg-alb para "My IP"; **(c)** desregistrar os targets.
</details>

<details><summary>💡 Pistas progressivas</summary>

1. 503 no ALB tem UM significado clássico. Qual?
2. Target group → aba Targets: qual o *Health status* e o **motivo** ao lado?
3. O health check consegue chegar (SG) e o path devolve 200 (`curl -I localhost/PATH` na instância)?
</details>

<details><summary>✅ Causa e correção</summary>

503 = **nenhum target healthy**. Diagnóstico pela coluna de status: *Request timed out* → SG não permite ALB→instância; *Health checks failed with 404* → path errado. Corrigir SG (origem=sg-alb) ou path `/`. Explicar a cadeia internet→ALB→instância.
</details>

---

## Chamado 4.6 — "O domínio sumiu do ar (mas o site existe!)"

**Cliente:** *"Desde a migração de ontem, `www.nossaloja.com.br` não abre — 'não foi possível encontrar o endereço'. Se eu acesso o DNS do load balancer direto, o site funciona perfeitamente!"*

**Missão:** domínio resolvendo para o lugar certo + explicação de DNS para leigo.
**Tempo alvo:** 20 min

<details><summary>🔧 Montagem (instrutor)</summary>

Zona no Route 53 (pode usar domínio fictício + teste com `dig @ns-da-zona`). Sabotagem (escolha 1): **(a)** deletar o registro `www`; **(b)** apontar o A/Alias para um ALB deletado ou IP inválido; **(c)** criar o registro como CNAME na raiz (erro proposital de configuração para ele identificar por que não salva/funciona).
</details>

<details><summary>💡 Pistas progressivas</summary>

1. "Site funciona pelo DNS do ALB" isola o problema em qual camada?
2. `dig www.nossaloja.com.br` retorna o quê? E consultando direto o NS da zona (`dig @ns-xxx.awsdns...`)?
3. Que tipo de registro aponta um nome para um ALB? E na raiz do domínio?
</details>

<details><summary>✅ Causa e correção</summary>

Registro ausente/apontando para alvo morto. Recriar `www` como **A-Alias → ALB** (raiz do domínio também Alias — CNAME é proibido no apex). Explicação leiga: DNS é a "agenda de contatos" da internet; o contato foi apagado, o telefone (servidor) continuava funcionando.
</details>

---

## 🏁 Rodada final sugerida (dia de simulado)

O instrutor escolhe **3 chamados aleatórios**, monta os ambientes com antecedência e dá **60 minutos** para o competidor resolver todos, anotando causa raiz de cada um. Ganha medalha quem fechar 3/3 com explicações corretas. 🥇
