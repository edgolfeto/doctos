# Laboratório 4: Arquitetura Resiliente para Plataforma de E-Sports
**Ambiente:** AWS Academy Sandbox (Restrições: *Sem permissões de IAM Roles* e *Sem acesso ao Amazon EFS*).

## Cenário
A *GameByte* é uma plataforma de organização de torneios de E-Sports. O principal recurso do sistema é a "Central de Resultados", onde os jogadores enviam *screenshots* (capturas de tela) que comprovam suas vitórias nas partidas.

Atualmente, o sistema opera em um servidor monolítico, e quando a carga de acessos sobe durante finais de campeonato, o site cai. Além disso, quando o servidor precisa ser reiniciado, há o risco de perda das imagens enviadas. A diretoria da GameByte quer modernizar a arquitetura migrando para múltiplas instâncias em alta disponibilidade, gerenciadas por um balanceador de carga.

## O Desafio e Restrições
Você, como competidor/arquiteto de nuvem, precisa implementar essa infraestrutura automatizada. No entanto, o ambiente atual da sua conta AWS (Sandbox) tem bloqueios rígidos de segurança:
1. Você **não** pode criar *IAM Roles* para acesso aos serviços da AWS.
2. O serviço de armazenamento gerenciado *Amazon EFS* **está bloqueado**.

## Sobre a aplicação
O portal da *GameByte* é desenvolvido em Node.js e está empacotado em um arquivo `.tar.gz`. O arquivo fonte se encontra armazenado em um bucket público do Amazon S3. Quando as instâncias iniciarem, elas devem se provisionar automaticamente sem acesso humano direto.

## Requisitos e Parâmetros de Segurança
1. **Balanceamento e Exposição:** O sistema só pode receber requisições de clientes através do Load Balancer. As instâncias da aplicação não devem possuir IPs públicos nem acesso direto via internet para a porta web.
2. **Armazenamento Customizado:** Como o EFS está desabilitado, você deve projetar e provisionar uma instância EC2 dedicada (bastion/storage) atuando como um Servidor NFS. O *Security Group* desta instância NFS deve permitir o tráfego da porta 2049 estritamente vindo das instâncias da aplicação.
3. **Download sem Credenciais:** Como *IAM Roles* não estão disponíveis, o download do pacote da aplicação a partir do S3 deve ser feito de forma anônima via linha de comando no momento da inicialização.
4. **Zero Touch:** Toda nova instância de aplicação web deve baixar os pacotes do Node.js, descompactar a aplicação, montar o diretório de uploads apontando para o servidor NFS remoto e iniciar o servidor web via *User Data*, sem nenhum passo manual.

## Critérios de Sucesso (Entregáveis)
1. Apresentar a topologia lógica em um diagrama indicando claramente o fluxo do balanceador, as instâncias de aplicação e a instância customizada de armazenamento NFS.
2. Acessar a aplicação através do DNS do Load Balancer.
3. Enviar a *screenshot* de um jogo e comprovar, através dos logs ou atualizando a página, que a imagem foi armazenada no servidor NFS e está sendo lida perfeitamente pelas demais instâncias do cluster web.
4. Mostrar o script *User Data* utilizado em formato legível, evidenciando como as restrições do ambiente sandbox foram tecnicamente contornadas.
