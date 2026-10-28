# integration-platform

## O problema

Toda integração de e-commerce (Tray, Yampi, e as que vierem depois) precisa das mesmas quatro coisas: conectar a loja, receber webhook, buscar dado na API do provider e traduzir isso pro resto do sistema. Hoje cada integração nova repete esse trabalho do zero, e cada uma tem um jeito ligeiramente diferente de fazer a mesma coisa.

## A ideia central

Toda integração tem exatamente **4 capacidades**:

1. **authentication** — como logar e manter o token válido
2. **webhook** — como validar e entender o que a loja mandou
3. **resources** — como buscar dado na API do provider (um pedido, uma lista de status)
4. **eventMapping** — como traduzir isso pro resto do sistema (flow, CRM)

Cada capacidade é **JSON ou código, nunca os dois**:

- Padrão (OAuth2 normal, webhook com assinatura HMAC, um `GET` simples por id) → só JSON, o motor genérico já sabe executar.
- Torto, só desse provider (handshake de autenticação específico, formato de evento esquisito) → um pedacinho de código só pra essa parte; o resto continua em JSON.

Isso faz a maioria das integrações futuras serem **só um arquivo JSON novo**, sem escrever ou revisar código.

```mermaid
flowchart TB
    subgraph JSON["definitions/*.integration.json"]
        C1[authentication]
        C2[webhook]
        C3[resources]
        C4[eventMapping]
    end

    subgraph CODE["adapter de código\n(só quando o provider é torto)"]
        A1[ex: handshake da Tray]
    end

    subgraph ENGINE["motor genérico (igual pra todas)"]
        E1[resolver: decide JSON ou código por capacidade]
        E2[pipeline de webhook]
        E3[cliente HTTP com retry]
        E4[segredos: por integração e do app]
    end

    C1 --> E1
    C2 --> E1
    C3 --> E1
    C4 --> E1
    A1 --> E1
    E1 --> E2
    E1 --> E3
    E1 --> E4
```

## Exemplo real: Yampi vs Tray

| Capacidade | Yampi | Tray |
|---|---|---|
| authentication | JSON — OAuth2 com redirect (PKCE) | código — handshake próprio da Tray |
| webhook | JSON — assinatura HMAC | JSON — segredo por loja, sem assinatura |
| resources | JSON | JSON (pedido + lista de status paginada) |
| eventMapping | JSON | código — status customizado da loja, telefone com DDI |

A Yampi segue o padrão quase todo; a Tray só precisa de código nos dois pontos onde ela é diferente de todo mundo.

## Conectar uma loja: 3 jeitos, 1 resultado

Nem toda autenticação funciona igual. O motor sabe lidar com três:

```mermaid
flowchart TB
    U[Usuário] --> Q{Como o provider\nautentica?}

    Q -->|OAuth2 com redirect\nex: Yampi| R1[1. pede a authorizationUrl]
    R1 --> R2[2. usuário autoriza no provider]
    R2 --> R3[3. provider devolve code + state pra gente]

    Q -->|OAuth2 sem redirect\nusuário cola o code| D1[1. usuário cola o code\nnum campo do formulário]

    Q -->|Handshake próprio\nex: Tray| C1[1. formulário com os campos\nque esse provider pede]

    R3 --> X[troca por token]
    D1 --> X
    C1 --> X

    X --> S[(integração salva:\ntoken + segredo de webhook\npróprio, só dela)]
```

## Um webhook chegando

```mermaid
flowchart TB
    W[Loja manda webhook] --> F[acha de qual integração é\npelo id da conta do provider]
    F -->|não achou| I[ignora, log interno]
    F -->|achou| AU{é legítimo?\nsegredo da integração\nou assinatura do provider}
    AU -->|não| N[não autêntico, log interno]
    AU -->|sim| EV[descobre o que aconteceu\npedido criado? atualizado?]
    EV -->|já processamos esse evento?| DUP[duplicado, ignora]
    EV -->|evento novo| PUB[publica evento padronizado]
    PUB --> FLOW[flow / CRM busca o pedido\ncompleto e reage]
```

O provider sempre recebe "ok" na resposta, não importa o resultado — erro, duplicado ou sucesso ficam só no log interno, nunca voltam pra loja.

## O que é sempre da plataforma, não do JSON

Não importa a integração, isso é sempre igual e nunca se repete:

- separar por cliente (tenant) e por conta do provider
- segredo de webhook gerado e guardado por integração, não pelo código de cada provider
- não processar o mesmo evento duas vezes (dedupe)
- esconder token e campo sensível de qualquer resposta de API
- log padronizado: qual integração, qual passo, se veio do JSON ou do código

## Onde isso está hoje

Vive em `integration-platform/`, fora do CRM antigo (`modules/integrations`), sem afetar quem já está em produção. Tray e Yampi já estão implementadas nesse novo modelo — conectar, webhook, buscar dado, esconder dado sensível na resposta da API. O que falta é ligar isso ao motor de flow do CRM.
