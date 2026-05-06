# ADR-001: Arquitetura de Microserviços para Plataforma de E-commerce

**Status:** Aceito  
**Data:** 2026-05-05  
**Autores:** Arquitetura de Software  

---

## Contexto

A plataforma de e-commerce precisa suportar três domínios funcionais distintos: gerenciamento de usuários, catálogo de produtos e processamento de pedidos. Cada domínio possui ciclos de vida, frequências de mudança e requisitos de escala independentes.

O volume esperado de tráfego é assimétrico:
- Leitura de produtos: alto volume, picos em promoções
- Criação de pedidos: moderado, exige consistência e rastreabilidade
- Usuários: baixo volume relativo, mas crítico para autenticação

O time é composto por squads que podem atuar de forma independente por domínio. A plataforma deve ser capaz de evoluir cada serviço sem deploys globais e sem impactar contratos já estabelecidos.

---

## Decisão

Adotar **arquitetura de microserviços** com os seguintes pilares:

- **3 serviços independentes:** `user-service`, `product-service`, `order-service`
- **Stack:** Python + Django Rest Framework (DRF) por serviço
- **Bancos de dados isolados:** cada serviço possui seu próprio PostgreSQL — sem compartilhamento de schema ou tabelas
- **Comunicação síncrona:** HTTP REST para chamadas em tempo real (ex: order-service consultando user-service e product-service)
- **Preparação para mensageria:** contratos desenhados para ser publicáveis via eventos (RabbitMQ/Kafka), sem implementação nesta fase
- **Autenticação centralizada:** JWT emitido pelo user-service, validado localmente pelos demais serviços via chave pública compartilhada

---

## Alternativas Consideradas

### Alternativa 1: Monólito (Django único)

**Prós:**
- Simples de desenvolver e testar localmente
- Sem overhead de rede entre domínios
- Transações ACID nativas entre entidades
- Menor complexidade operacional

**Contras:**
- Acoplamento crescente entre domínios conforme o sistema evolui
- Deploy total obrigatório a cada mudança
- Dificuldade de escalar componentes individualmente
- Times diferentes precisam coordenar releases

**Quando faria sentido:** MVP inicial com time único e < 5k usuários ativos.

---

### Alternativa 2: Serverless (AWS Lambda / GCP Cloud Functions)

**Prós:**
- Escala automática por função
- Custo zero em idle
- Sem gerenciamento de infraestrutura de servidor

**Contras:**
- Cold starts impactam latência em operações críticas (ex: checkout)
- Estado difícil de gerenciar entre funções
- Dificulta debugging e rastreabilidade distribuída
- Vendor lock-in elevado
- Django não é otimizado para execução em Lambda sem adaptadores (Mangum/Zappa)

**Quando faria sentido:** Workloads com padrão de uso muito esporádico ou funções de processamento batch.

---

## Trade-offs da Decisão Adotada

| Dimensão | Ganho | Custo |
|---|---|---|
| **Escalabilidade** | Escala independente por serviço | Infraestrutura mais complexa |
| **Deploy** | Deploy isolado por domínio | CI/CD por serviço, não global |
| **Consistência** | Domínios com contratos claros | Sem transações distribuídas nativas |
| **Desenvolvimento** | Times independentes por squad | Overhead de comunicação síncrona |
| **Observabilidade** | Métricas por serviço | Exige correlação de traces distribuídos |
| **Latência** | Cada serviço responde por seu domínio | Chamadas encadeadas somam latência |

---

## Consequências

### Positivas

- Cada serviço pode ser versionado, deployado e escalado de forma independente
- Falhas em product-service não derrubam user-service
- Domínios com linguagem e frameworks distintos podem ser adotados por serviço no futuro
- Testabilidade por domínio com contratos bem definidos (contract testing possível com Pact)

### Negativas e Mitigações

| Problema | Mitigação |
|---|---|
| Consistência eventual em pedidos | Validação síncrona de estoque no momento da criação do pedido |
| Serviço indisponível bloqueia fluxo | Circuit breaker (fase futura via Resilience4j ou tenacity) |
| Rastreabilidade entre serviços | Correlation-ID propagado via HTTP headers em todos os requests |
| Dados replicados (ex: nome do usuário no pedido) | Desnormalização intencional — order-service armazena snapshot de dados no momento do pedido |

---

## Quando esta Arquitetura NÃO deve ser usada

- **Time pequeno (< 3 devs):** o overhead operacional supera os benefícios
- **Domínios fortemente acoplados:** se os serviços precisam de transações ACID entre si com frequência, microserviços viram um problema
- **MVP de validação de mercado:** monólito modular entrega o mesmo valor com menos custo
- **Sem infraestrutura de observabilidade:** operar microserviços sem logs centralizados, tracing e alertas por serviço é cego e perigoso
- **Sem estratégia de API versioning:** quebras de contrato entre serviços derrubam o sistema inteiro

---

## Requisitos para Operacionalizar esta Arquitetura

- [ ] API Gateway (nginx, Kong ou AWS API Gateway) como ponto de entrada único
- [ ] Service Discovery básico (variáveis de ambiente em fase inicial, Consul/Kubernetes em produção)
- [ ] Logs estruturados (JSON) com correlation-ID em todos os serviços
- [ ] Health check endpoints (`/health`) em todos os serviços
- [ ] Estratégia de versionamento de API (`/api/v1/`)
- [ ] Contratos de API documentados (OpenAPI/Swagger por serviço)

---

## Referências

- [Martin Fowler — Microservices](https://martinfowler.com/articles/microservices.html)
- [Sam Newman — Building Microservices (2nd ed.)]
- [Django REST Framework — Official Docs](https://www.django-rest-framework.org/)
- [The Twelve-Factor App](https://12factor.net/)
