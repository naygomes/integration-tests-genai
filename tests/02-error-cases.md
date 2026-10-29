# Casos de teste – Erros e resiliência (saída do prompt v2, revisada)

## A) Falha de dependência

**ERR-01 | Categoria: A | Estoque indisponível**

- Simulação: Estoque com conexão recusada (container parado / toxiproxy).
- Esperado (Pedidos): 503 após 1 retry (2 tentativas em `POST /reservas`); nenhum pedido `CONFIRMADO`.
- Estado final: Estoque sem reservas; Pagamentos sem chamadas e sem autorizações.
- Risco: cascata de falha / pedido parcial.

**ERR-02 | Categoria: A | Timeout do Estoque (> 3s)**

- Simulação: WireMock com delay de 4s em `POST /reservas`.
- Esperado (Pedidos): exatamente 1 retry (2 chamadas no total); se persistir, 503 em ≈ 6s, sem bloqueio indefinido.
- Estado final: Estoque sem reservas efetivas [HIPÓTESE: reservas tardias da 1ª tentativa são descartadas ou expiram]; Pagamentos sem chamadas.
- Risco: threads presas, retry excessivo, reserva duplicada por retry.

**ERR-03 | Categoria: A | Timeout do Pagamentos (> 3s)**

- Simulação: delay de 4s em `POST /autorizacoes`, com Estoque reservando normalmente.
- Esperado (Pedidos): **sem retry** (política só para Estoque); 503; pedido `FALHOU` [HIPÓTESE: status gravado].
- Estado final: Estoque com reservas liberadas (`DELETE /reservas/{id}`); Pagamentos com 1 única tentativa [HIPÓTESE: autorização tardia é estornada, pois o contrato não define cancelamento por timeout].
- Risco: cobrança duplicada por retry indevido / cobrança sem pedido.

**ERR-04 | Categoria: A | Pagamentos responde 500**

- Simulação: stub 500 em `POST /autorizacoes`.
- Esperado (Pedidos): 503 [HIPÓTESE: o contrato só lista 402 e 503 para falha de pagamento]; corpo sem detalhes internos.
- Estado final: Estoque com reservas liberadas; Pagamentos sem autorização.
- Risco: vazamento de erro interno e estoque preso.

## B) Consistência e compensação

**ERR-05 | Categoria: B | Pagamento recusado após reserva**

- Simulação: Estoque 201; Pagamentos 402.
- Esperado (Pedidos): 402; `DELETE /reservas/{id}` para cada reserva; pedido `FALHOU` [HIPÓTESE: status gravado].
- Estado final: Estoque com saldo igual ao inicial; Pagamentos sem autorização ativa.
- Risco: estoque preso por pedido nunca pago.

**ERR-06 | Categoria: B | Falha ao liberar reserva na compensação**

- Simulação: Pagamentos 402; Estoque `DELETE /reservas/{id}` retorna 500.
- Esperado (Pedidos): 402 ao cliente [HIPÓTESE]; falha de compensação registrada em log/fila de reprocessamento [HIPÓTESE: existe mecanismo de reprocessamento].
- Estado final: Estoque com reserva ainda ativa (pendente de reprocessamento); Pagamentos sem autorização.
- Risco: inconsistência permanente de estoque.

**ERR-07 | Categoria: B | Pagamento autorizado, falha ao persistir pedido**

- Simulação: Pagamentos 201; falha de banco no próprio Pedidos (fault injection) antes de confirmar o pedido.
- Esperado (Pedidos): 503 [HIPÓTESE: o contrato não define código para falha interna]; `POST /autorizacoes/{id}/estorno` chamado 1 vez; reservas liberadas.
- Estado final: Pagamentos com autorização estornada; Estoque com saldo igual ao inicial.
- Risco: cobrança sem pedido (maior impacto financeiro).

**ERR-08 | Categoria: B | Segunda reserva falha em pedido multi-SKU**

- Simulação: SKU1 reservado (201); SKU2 retorna 409.
- Esperado (Pedidos): 409; Pagamentos não chamado.
- Estado final: Estoque com reserva de SKU1 liberada (saldo inicial); Pagamentos sem autorização.
- Risco: reserva parcial.

## C) Idempotência

**ERR-09 | Categoria: C | Mesma `Idempotency-Key` repetida**

- Simulação: enviar POST /pedidos duas vezes com mesma chave e mesmo corpo.
- Esperado (Pedidos): 2ª resposta idêntica à 1ª (mesmo `id`) [HIPÓTESE: o contrato exige o header, mas não descreve o replay].
- Estado final: Estoque com 1 reserva por item; Pagamentos com 1 autorização.
- Risco: cobrança duplicada em retry do cliente.

**ERR-10 | Categoria: C | Requisições concorrentes com mesma chave**

- Simulação: 5 POST simultâneos com mesma chave e mesmo corpo.
- Esperado (Pedidos): apenas 1 pedido criado; demais retornam o mesmo resultado ou 409 [HIPÓTESE].
- Estado final: Estoque com 1 reserva por item; Pagamentos com 1 autorização.
- Risco: race condition com cobrança e reserva duplicadas.

**ERR-11 | Categoria: C | Mesma chave com corpo diferente**

- Simulação: reutilizar a chave alterando `quantidade`.
- Esperado (Pedidos): 422 [HIPÓTESE: 422 está no contrato para regra de negócio, mas esta regra não é descrita].
- Estado final: sem novas chamadas; Estoque e Pagamentos inalterados em relação ao 1º pedido.
- Risco: reutilização indevida de chave.

## D) Contrato

**ERR-12 | Categoria: D | Estoque responde 201 sem `reservaId`**

- Simulação: stub de `POST /reservas` com corpo `{}`.
- Esperado (Pedidos): 503 [HIPÓTESE: erro de contrato tratado como dependência indisponível]; não chama Pagamentos.
- Estado final: Estoque pode ter reserva sem identificador (não liberável) [HIPÓTESE]; Pagamentos sem chamadas.
- Risco: reserva impossível de liberar.

**ERR-13 | Categoria: D | Pagamentos responde 201 sem `autorizacaoId`**

- Simulação: stub de `POST /autorizacoes` com corpo `{}`.
- Esperado (Pedidos): 503 [HIPÓTESE]; pedido não confirmado.
- Estado final: Estoque com reservas liberadas; Pagamentos com autorização sem id (não estornável) [HIPÓTESE].
- Risco: impossibilidade de estorno futuro.

**ERR-14 | Categoria: D | Estoque responde corpo com tipo inválido**

- Simulação: stub de `POST /reservas` com 201 e `{"reservaId": 123}` (número, não string).
- Esperado (Pedidos): 503 [HIPÓTESE] ou aceitação documentada; sem erro 500 não tratado.
- Estado final: Pagamentos sem chamadas; Estoque conforme stub.
- Risco: falha de desserialização que derruba o fluxo com 500.

**ERR-15 | Categoria: D | Pagamentos responde 402 sem corpo/schema esperado**

- Simulação: stub de `POST /autorizacoes` com 402 e corpo vazio.
- Esperado (Pedidos): 402 ao cliente (o contrato interno só exige o status).
- Estado final: Estoque com reservas liberadas; Pagamentos sem autorização.
- Risco: erro de parsing mascara a recusa e deixa estoque preso.

## Prioridade (por risco de negócio)

1. ERR-07, ERR-09, ERR-10, ERR-03 (dinheiro)
2. ERR-05, ERR-08, ERR-06 (estoque/consistência)
3. ERR-02, ERR-12, ERR-13, ERR-15
4. ERR-01, ERR-04, ERR-11, ERR-14

Observação: o header `Idempotency-Key` ausente/inválido (400) é validação de campo isolada e fica fora deste escopo de integração.
