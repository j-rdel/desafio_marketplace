# PRD — Módulo de Checkout

**Status:** Rascunho  
**Data:** 2026-05-12  
**Autor:** Jardel Urban  
**Versão:** 1.0

---

## Objetivo

Permitir que um comprador autenticado finalize a compra dos itens do carrinho por meio de pagamento via Pix, gerando um pedido confirmado e notificando o vendedor — tudo dentro do fluxo já existente dos serviços `user-service`, `product-service` e `order-service`.

O módulo de checkout não é um serviço novo: é o fluxo orquestrado pelo `order-service` que vai do carrinho até a confirmação do pagamento.

---

## Personas

### Comprador
- Usuário cadastrado e autenticado na plataforma.
- Tem itens no carrinho e quer concluir a compra rapidamente.
- Já usa Pix no dia a dia e espera um QR Code ou copia-e-cola imediato.
- Abandona o fluxo se a experiência for lenta ou confusa.

### Vendedor
- Cadastrou produtos no `product-service`.
- Quer ser notificado quando um pedido for pago para separar e despachar.
- Não interage diretamente com o checkout — recebe o resultado.

### Operador / Admin
- Monitora pedidos com pagamento pendente ou expirado.
- Precisa de visibilidade sobre status de pagamento para suporte.

---

## Escopo

### Dentro do escopo (v1)
- Geração de cobrança Pix (QR Code + copia-e-cola) ao confirmar o pedido.
- Validação de estoque antes de emitir a cobrança (chamada ao `product-service`).
- Expiração automática da cobrança após 30 minutos sem pagamento.
- Confirmação do pedido via webhook do provedor de pagamento.
- Atualização de status do pedido: `pending_payment` → `paid` → `processing`.
- Estorno automático de estoque em cobranças expiradas ou canceladas.

### Fora do escopo (v1)
- Cartão de crédito, boleto ou outros meios de pagamento.
- Parcelamento.
- Gestão de devoluções/reembolso (fase futura).
- Split de pagamento entre múltiplos vendedores.

---

## Regras de Negócio

### Pagamento

| Regra | Detalhe |
|---|---|
| Meio aceito | Somente Pix |
| Validade da cobrança | 30 minutos após geração |
| Valor mínimo | R$ 1,00 |
| Tentativas | 1 cobrança por pedido — nova tentativa exige novo pedido |
| Confirmação | Somente via webhook autenticado do provedor |
| Falha no webhook | Reprocessamento automático com até 3 tentativas (backoff exponencial) |

### Estoque

- O estoque é **reservado** no momento da criação do pedido, antes de gerar a cobrança Pix.
- Se a cobrança expirar ou for cancelada, o estoque reservado é **liberado automaticamente**.
- Se o `product-service` retornar estoque insuficiente, o checkout é **bloqueado** com erro 422.

### Status do Pedido

```
cart → pending_payment → paid → processing → shipped (fora do escopo v1)
                      ↘ expired (sem pagamento em 30 min)
                      ↘ cancelled (cancelado pelo comprador antes do pagamento)
```

---

## Fluxo Principal

```
Comprador confirma carrinho
  → order-service valida estoque (product-service)
  → order-service cria pedido com status pending_payment
  → order-service solicita cobrança Pix ao provedor
  → provedor retorna QR Code + copia-e-cola
  → comprador paga
  → provedor dispara webhook para order-service
  → order-service atualiza pedido para paid
  → order-service notifica vendedor
```

---

## Requisitos Não-Funcionais

| Atributo | Meta |
|---|---|
| Latência de geração do QR Code | < 2 s (p95) |
| Disponibilidade | 99,5% |
| Confirmação de pagamento | ≤ 5 s após webhook recebido |
| Segurança do webhook | Validação de assinatura HMAC-SHA256 |
| Idempotência | Webhook com mesmo ID não processa pedido duas vezes |

---

## Integrações

| Sistema | Responsabilidade |
|---|---|
| `user-service` | Valida JWT do comprador |
| `product-service` | Confirma disponibilidade e reserva estoque |
| Provedor Pix (ex: Efí, Asaas, Pagar.me) | Gera cobrança, dispara webhook de confirmação |

---

## Métricas de Sucesso

- Taxa de conversão do checkout ≥ 60% (pedidos pagos / pedidos iniciados).
- Taxa de expiração de cobrança ≤ 25%.
- Tempo médio entre geração do QR Code e pagamento ≤ 10 min.
- Zero pedidos confirmados sem pagamento real (integridade crítica).
