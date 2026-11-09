# Arquitetura

Esta página apresenta a arquitetura do Sistema GranMestre usando o **C4 Model**, nos níveis 1 (Contexto) e 2 (Containers).

## Nível 1: Diagrama de Contexto

Mostra o sistema no centro, as pessoas que interagem com ele e os sistemas externos.

```mermaid
flowchart TB
    cliente(["👤 <b>Cliente</b><br/>[Pessoa]<br/>Solicita a marmita no balcão"])
    atendente(["👤 <b>Atendente</b><br/>[Pessoa]<br/>Registra e confirma pedidos"])
    cozinheiro(["👤 <b>Cozinheiro</b><br/>[Pessoa]<br/>Prepara a marmita a partir do pedido impresso"])

    sgr["<b>Sistema GranMestre</b><br/>[Sistema de software]<br/>Registra pedidos, gera número único<br/>e envia para impressão na cozinha"]

    impressora["<b>Impressora da Cozinha</b><br/>[Sistema externo / dispositivo]<br/>Imprime os pedidos para o cozinheiro"]

    cliente -- "Solicita pedido (balcão)" --> atendente
    atendente -- "Registra e confirma pedidos<br/>e reimprime em caso de falha" --> sgr
    sgr -- "Envia pedido para impressão" --> impressora
    impressora -. "Pedido impresso" .-> cozinheiro

    classDef person fill:#08427b,stroke:#052e56,color:#fff;
    classDef system fill:#1168bd,stroke:#0b4884,color:#fff;
    classDef ext fill:#999,stroke:#6b6b6b,color:#fff;
    class cliente,atendente,cozinheiro person;
    class sgr system;
    class impressora ext;
```

**Legenda:** azul escuro = pessoas (atores); azul = sistema em construção; cinza = sistema/dispositivo externo.

| Elemento | Tipo | Descrição |
|---|---|---|
| Cliente | Pessoa | Pede a marmita no balcão e retira o pedido pronto |
| Atendente | Pessoa | Registra o pedido (tipo, acompanhamentos, observação), confirma e reimprime se necessário |
| Cozinheiro | Pessoa | Recebe o pedido impresso e prepara a marmita |
| Sistema GranMestre | Sistema | Núcleo da solução, objeto desta documentação |
| Impressora da Cozinha | Sistema externo | Dispositivo que imprime o pedido, evitando que a cozinha dependa de computador ou tela |

## Nível 2: Diagrama de Containers

Detalha as partes fundamentais do sistema e as tecnologias de cada uma.

```mermaid
flowchart TB
    atendente(["👤 <b>Atendente</b><br/>[Pessoa]"])
    cozinheiro(["👤 <b>Cozinheiro</b><br/>[Pessoa]"])

    subgraph sgr ["Sistema GranMestre [Fronteira do sistema]"]
        front["<b>Aplicação Web (Front-end)</b><br/>[Container: React]<br/>Tela de registro, confirmação<br/>e reimpressão de pedidos"]
        api["<b>API de Pedidos</b><br/>[Container: Java / Spring Boot]<br/>Regras de negócio, geração do<br/>numero_pedido, endpoints REST"]
        print["<b>Módulo de Impressão</b><br/>[Container: Java / ESC-POS]<br/>Verifica disponibilidade da<br/>impressora e envia o pedido"]
        db[("<b>Banco de Dados</b><br/>[Container: PostgreSQL]<br/>Armazena pedidos e status")]
    end

    impressora["<b>Impressora da Cozinha</b><br/>[Sistema externo]"]

    atendente -- "Usa [HTTPS]" --> front
    front -- "POST /pedidos<br/>POST /pedidos/{id}/reimprimir<br/>[JSON/HTTPS]" --> api
    api -- "Lê e grava pedidos [JDBC/SQL]" --> db
    api -- "Solicita impressão /<br/>recebe sucesso ou erro" --> print
    print -- "Comandos de impressão [TCP/IP]" --> impressora
    impressora -. "Pedido impresso" .-> cozinheiro

    classDef person fill:#08427b,stroke:#052e56,color:#fff;
    classDef container fill:#438dd5,stroke:#2e6295,color:#fff;
    classDef ext fill:#999,stroke:#6b6b6b,color:#fff;
    class atendente,cozinheiro person;
    class front,api,print,db container;
    class impressora ext;
```

| Container | Tecnologia | Responsabilidade |
|---|---|---|
| Aplicação Web | React | Interface do atendente: registrar pedido, confirmar, ver erros e reimprimir |
| API de Pedidos | Java / Spring Boot (REST) | Valida o pedido, gera `numero_pedido`, persiste e orquestra a impressão |
| Módulo de Impressão | Java / ESC-POS | Verifica a disponibilidade da impressora e envia o pedido; retorna erro sem excluir o pedido |
| Banco de Dados | PostgreSQL | Mantém todos os pedidos salvos, inclusive os que falharam na impressão |

!!! note "Decisões arquiteturais"
    - O pedido é **salvo antes da impressão**, então uma falha na impressora nunca causa perda de dados.
    - A cozinha recebe o pedido **impresso**, sem depender de computador ou tela (RN006).
    - A reimpressão manual retoma o fluxo a partir da etapa de impressão, sem recriar o pedido.
