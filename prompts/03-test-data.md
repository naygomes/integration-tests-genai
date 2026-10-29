# Dados de teste (restrições + saída estruturada)

**Técnica:** restrições do schema + classes de equivalência/valor limite + saída em JSON.

---

Gere massa de dados para POST /pedidos com base no schema PedidoRequest: [openapi.yaml](../docs/openapi.yaml).

Técnicas: partição de equivalência e análise de valor limite.
Classes: VALIDO, LIMITE_VALIDO, INVALIDO.
Cubra: quantidade (0,1,10,11), itens (0,1,20,21), sku (padrão correto/incorreto), metodo (CARTAO/PIX/outro),
tokenCartao ausente com CARTAO, valor diferente do total (preço unitário: 10.00 para todo SKU).

Restrições:

- Dados fictícios e claramente falsos (sem CPF/cartão reais; tokens como tok_teste_xxx).
- Saída: array JSON; cada item com "id", "classe", "descricao", "request" e "esperado" (status HTTP do contrato).
- Não repita registros equivalentes.
