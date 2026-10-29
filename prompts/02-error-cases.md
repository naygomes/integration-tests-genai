# Casos de erro e resiliência (baseado em contrato + few-shot)

**Técnica:** contrato como entrada, exemplo (few-shot) do formato desejado e categorias de falha explícitas.

---

Você é especialista em testes de resiliência e consistência de sistemas distribuídos.

Contrato: [openapi.yaml](../docs/openapi.yaml)

Gere casos de teste de integração NEGATIVOS para POST /pedidos, organizados nas categorias:
A) Falha de dependência (Estoque/Pagamentos retornam erro, caem ou excedem timeout de 3s)
B) Consistência/compensação (falha parcial: uma reserva feita e pagamento falha, e vice-versa)
C) Idempotência (mesmo Idempotency-Key repetido, concorrência)
D) Contrato (resposta da dependência fora do schema, campo ausente)

Para cada caso, informe: como simular a falha (stub/mock/fault injection), resposta esperada de Pedidos,
estado final esperado em Estoque e Pagamentos, e o risco coberto.

Exemplo do formato esperado:
ID: ERR-00 | Categoria: B | Simulação: Pagamentos responde 402 após Estoque reservar 2 itens
Esperado: Pedidos 402; reserva liberada via DELETE /reservas/{id}; pedido com status FALHOU
Risco: estoque preso por pedido nunca pago

Regras: não invente endpoints; marque [HIPÓTESE] quando o contrato for omisso; priorize pelo risco de negócio;
mínimo de 2 casos por categoria.
