# Backlog — Módulo de Checkout

**Referência:** [PRD_CHECKOUT.md](PRD_CHECKOUT.md)  
**Data:** 2026-05-12

---

## User Stories

### US-001 — Confirmar carrinho e iniciar checkout

**Como** comprador autenticado,  
**quero** revisar os itens do meu carrinho e confirmar a compra,  
**para que** o sistema valide meu pedido e me apresente as opções de pagamento.

**Critério de prioridade:** Sem essa história nenhum outro fluxo de checkout começa.

---

### US-002 — Gerar cobrança Pix

**Como** comprador autenticado com pedido criado,  
**quero** receber um QR Code e um código copia-e-cola Pix,  
**para que** eu possa efetuar o pagamento pelo meu banco em menos de 2 segundos de espera.

**Critério de prioridade:** É o único meio de pagamento aceito na v1 — bloqueia a receita inteira se falhar.

---

### US-003 — Confirmar pagamento via webhook

**Como** sistema (order-service),  
**quero** receber e processar o webhook do provedor Pix com assinatura válida,  
**para que** o pedido seja marcado como pago de forma automática, segura e sem duplicidade.

**Critério de prioridade:** Garante a integridade financeira — um pedido não pode ser confirmado sem pagamento real.

---

### US-004 — Expirar cobrança sem pagamento

**Como** sistema (order-service),  
**quero** expirar automaticamente cobranças Pix não pagas após 30 minutos,  
**para que** o estoque reservado seja liberado e o pedido encerrado sem intervenção manual.

**Critério de prioridade:** Sem expiração, o estoque fica preso indefinidamente e outros compradores são bloqueados.

---

### US-005 — Bloquear checkout por estoque insuficiente

**Como** comprador,  
**quero** ser informado imediatamente quando um item do carrinho não tiver estoque disponível,  
**para que** eu não pague por algo que não pode ser entregue.

**Critério de prioridade:** Protege o comprador e evita estornos custosos para o vendedor.

---

## Critérios de Aceitação (BDD)

As duas histórias escolhidas são **US-002** e **US-003** por representarem o núcleo do fluxo de pagamento: uma gera a cobrança e a outra confirma o dinheiro — falha em qualquer uma delas paralisa a receita.

---

### US-002 — Gerar cobrança Pix

```gherkin
Feature: Geração de cobrança Pix
  Como comprador autenticado com pedido em status pending_payment
  Quero receber um QR Code e um código copia-e-cola Pix
  Para efetuar o pagamento pelo meu aplicativo bancário

  Background:
    Given o comprador está autenticado com JWT válido
    And o carrinho contém 1 unidade do produto "Tênis Running X" (R$ 299,90)
    And o product-service confirma estoque disponível

  Scenario: Geração bem-sucedida da cobrança Pix
    When o comprador confirma o pedido
    Then o order-service cria o pedido com status "pending_payment"
    And o provedor Pix retorna QR Code e código copia-e-cola
    And a resposta inclui o campo "pix_qr_code" não vazio
    And a resposta inclui o campo "pix_copy_paste" no formato EMV válido
    And a resposta inclui o campo "expires_at" com timestamp 30 minutos no futuro
    And o tempo de resposta total é menor que 2000ms

  Scenario: Tentativa de gerar cobrança para pedido já existente
    Given o comprador já possui um pedido com status "pending_payment" para o mesmo carrinho
    When o comprador tenta confirmar o pedido novamente
    Then o sistema retorna HTTP 409
    And a mensagem de erro é "Já existe uma cobrança ativa para este pedido"
    And nenhuma nova cobrança é criada no provedor Pix

  Scenario: Provedor Pix indisponível no momento da geração
    Given o provedor Pix está retornando timeout
    When o comprador confirma o pedido
    Then o sistema retorna HTTP 503
    And a mensagem de erro é "Serviço de pagamento temporariamente indisponível. Tente novamente em instantes."
    And o pedido não é criado
    And o estoque reservado é liberado
```

---

### US-003 — Confirmar pagamento via webhook

```gherkin
Feature: Confirmação de pagamento via webhook do provedor Pix
  Como sistema (order-service)
  Quero processar o webhook do provedor com segurança e idempotência
  Para marcar o pedido como pago apenas quando o pagamento for real e válido

  Background:
    Given existe um pedido "ORD-001" com status "pending_payment"
    And o pedido possui uma cobrança Pix ativa com ID "PIX-ABC123"
    And a cobrança não está expirada

  Scenario: Webhook válido confirma o pedido
    When o provedor envia POST /webhooks/pix com body:
      """
      {
        "event": "payment.confirmed",
        "payment_id": "PIX-ABC123",
        "order_id": "ORD-001",
        "amount": 29990
      }
      """
    And o header "X-Signature" contém assinatura HMAC-SHA256 válida
    Then o sistema retorna HTTP 200
    And o pedido "ORD-001" tem status atualizado para "paid"
    And o vendedor recebe notificação do novo pedido pago
    And o evento é registrado no log com correlation-ID

  Scenario: Webhook com assinatura inválida é rejeitado
    When o provedor envia POST /webhooks/pix com assinatura HMAC-SHA256 inválida no header "X-Signature"
    Then o sistema retorna HTTP 401
    And o pedido "ORD-001" permanece com status "pending_payment"
    And nenhuma notificação é enviada ao vendedor
    And o incidente é registrado no log de segurança

  Scenario: Webhook duplicado não processa o pedido duas vezes
    Given o pedido "ORD-001" já está com status "paid"
    When o provedor envia novamente o webhook com "payment_id": "PIX-ABC123"
    And o header "X-Signature" contém assinatura HMAC-SHA256 válida
    Then o sistema retorna HTTP 200
    And o status do pedido permanece "paid" sem alteração
    And nenhuma notificação duplicada é enviada ao vendedor

  Scenario: Webhook para pedido expirado
    Given o pedido "ORD-001" está com status "expired"
    When o provedor envia webhook de confirmação para "PIX-ABC123"
    And o header "X-Signature" contém assinatura HMAC-SHA256 válida
    Then o sistema retorna HTTP 422
    And a mensagem de erro é "Cobrança expirada — pagamento não pode ser confirmado"
    And o time financeiro é alertado para análise manual de estorno
```
