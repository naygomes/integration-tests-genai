# Casos de teste – Cenários de integração (saída do prompt v2, revisada)

| ID | Título | Pré-condições | Passos | Resultado esperado | Prioridade |
|----|--------|---------------|--------|--------------------|-----------|
| INT-01 | Criar pedido com cartão (fluxo feliz) | SKU `ABC-0001` com saldo ≥ 2; Pagamentos aprova | 1. POST /pedidos (2x ABC-0001, CARTAO, valor 20.00) | 201, status `CONFIRMADO`. Pedidos chamou Estoque `POST /reservas` e depois Pagamentos `POST /autorizacoes` (nessa ordem). Saldo do estoque reduzido em 2; 1 autorização de 20.00 | Alta |
| INT-02 | Retry do Estoque com sucesso na 2ª tentativa | Estoque falha por timeout (>3s) na 1ª chamada e responde 201 na 2ª | POST /pedidos (valor 20.00) | 201 `CONFIRMADO`; Estoque recebeu 2 chamadas `POST /reservas`; 1 única reserva efetiva; Pagamentos chamado 1 vez após a reserva | Média |
| INT-03 | Pedido com vários SKUs | Saldo suficiente para 3 SKUs distintos | POST /pedidos com 3 itens | 201; 3 reservas (uma por SKU); 1 única autorização com o total somado | Alta |
| INT-04 | Estoque insuficiente em um dos itens | SKU `ABC-0001` com saldo 1 | POST /pedidos com 2x ABC-0001 | 409; Pagamentos **não** é chamado; nenhuma autorização criada; pedido não confirmado | Alta |
| INT-05 | Pagamento recusado | Estoque OK; Pagamentos retorna 402 | POST /pedidos | 402; reservas liberadas (`DELETE /reservas/{id}`); saldo volta ao valor inicial; pedido `FALHOU` | Alta |
| INT-06 | Consultar pedido após criação | Pedido criado em INT-01 | GET /pedidos/{id} | 200; `status=CONFIRMADO`; `total=20.00` consistente com a autorização | Média |
| INT-07 | Consultar pedido inexistente | UUID aleatório | GET /pedidos/{id} | 404; nenhuma chamada a Estoque/Pagamentos | Baixa |
| INT-08 | Cancelar pedido confirmado | Pedido `CONFIRMADO` | POST /pedidos/{id}/cancelamento | 200, `CANCELADO`; reserva removida (saldo restaurado); `POST /autorizacoes/{id}/estorno` chamado 1 vez | Alta |
| INT-09 | Cancelar pedido já cancelado | Pedido `CANCELADO` | POST cancelamento novamente | 409; sem novo estorno nem nova liberação de estoque | Alta |
| INT-10 | Cancelar pedido inexistente | UUID aleatório | POST cancelamento | 404; sem efeitos nos serviços | Baixa |
| INT-11 | Cancelar pedido que falhou | Pedido com status `FALHOU` (ex.: INT-05) | POST cancelamento | 409 [HIPÓTESE]; sem estorno e sem `DELETE /reservas` adicionais | Média |
| INT-12 | Repetição com mesma `Idempotency-Key` | Pedido criado em INT-01 | Reenviar POST /pedidos com mesma chave e mesmo corpo | Mesma resposta (mesmo `id`, 201) [HIPÓTESE]; Estoque com 1 reserva e Pagamentos com 1 autorização no total | Alta |

## Hipóteses levantadas pelo prompt (a esclarecer com o time)

- [HIPÓTESE] Pedido com 2+ SKUs: se a reserva do 2º falhar, a do 1º é liberada (contrato não detalha).
- [HIPÓTESE] Ordem Estoque → Pagamentos (inferida do fluxo descrito, não explícita no contrato).
- [HIPÓTESE] Cancelamento de pedido `FALHOU` retorna 409 (INT-11).
- [HIPÓTESE] Repetição com mesma `Idempotency-Key` devolve a resposta original (INT-12).
