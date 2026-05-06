# Desafio Marketplace — Arquitetura de Microserviços

Projeto de design arquitetural de uma plataforma de e-commerce baseada em microserviços. O foco do desafio é a **decisão arquitetural, modelagem e documentação** — sem implementação de código backend.

---

## Sobre o Desafio

O objetivo é projetar um sistema de e-commerce simples com três serviços independentes que se comunicam via HTTP REST:

- **user-service** — cadastro e autenticação de usuários
- **product-service** — catálogo e controle de estoque
- **order-service** — criação e consulta de pedidos

Cada serviço possui banco de dados próprio (PostgreSQL), sem compartilhamento de schema entre eles.

---

## Estrutura do Repositório

```
desafio_marketplace/
├── services/
│   ├── user-service/       ← Domínio de usuários e autenticação
│   ├── product-service/    ← Domínio de produtos e estoque
│   └── order-service/      ← Domínio de pedidos
├── docs/
│   ├── ADR.md              ← Decisão arquitetural documentada
│   └── DIAGRAMAS.md        ← Diagramas ER, fluxo, arquitetura e APIs
├── frontend/
│   └── index.html          ← Mockup de UI (HTML + Tailwind, dados mockados)
└── README.md
```

---

## Documentação

### ADR.md — Architecture Decision Record

Documenta a decisão de usar microserviços com DRF, incluindo:

- Contexto e motivação
- Alternativas consideradas (Monólito e Serverless) com prós e contras
- Trade-offs da decisão adotada
- Quando essa arquitetura **não** deve ser usada

### DIAGRAMAS.md — Diagramas e Contratos de API

Contém todos os diagramas em Mermaid e a definição completa das APIs:

- **Diagrama ER** — entidades e relacionamentos entre os serviços
- **Diagrama de sequência** — fluxo de criação de pedido de ponta a ponta
- **Diagrama de arquitetura** — visão geral dos serviços, bancos e comunicação
- **Contratos de API** — todos os endpoints com request/response em JSON

---

## Stack Definida

| Camada | Tecnologia |
|---|---|
| Backend (por serviço) | Python + Django REST Framework |
| Banco de dados | PostgreSQL (um por serviço) |
| Comunicação | HTTP REST |
| Autenticação | JWT (emitido pelo user-service) |
| Mensageria (fase futura) | RabbitMQ / Kafka |

---

## Fluxo Principal

```
Cliente → Login (user-service)
       → Busca produto (product-service)
       → Cria pedido (order-service)
            → Valida usuário (user-service)
            → Valida estoque (product-service)
            → Registra pedido e retorna confirmação
```

---

## Como visualizar o frontend

Abra o arquivo diretamente no navegador:

```bash
open frontend/index.html
```

O mockup inclui busca de produtos, filtro por categoria, carrinho funcional e simulação de checkout — tudo com dados estáticos, sem dependência de backend.

---

## Como visualizar os diagramas

Os diagramas estão escritos em [Mermaid](https://mermaid.js.org/). Para renderizá-los:

- **VS Code:** instale a extensão [Mermaid Preview](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid)
- **GitHub:** renderiza Mermaid nativamente em arquivos `.md`
- **Online:** cole o código em [mermaid.live](https://mermaid.live)
