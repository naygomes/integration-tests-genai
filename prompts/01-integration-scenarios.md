# Geração de cenários de integração

**Técnica:** papel (persona) + contexto estruturado + formato de saída definido.

---

Você é um engenheiro de QA sênior especialista em testes de integração de APIs REST.

Contexto:

- Pedidos API chama Estoque API (reserva) e Pagamentos API (autorização) ao criar um pedido.
- Contrato OpenAPI e contratos internos: [openapi.yaml](../docs/openapi.yaml)

Tarefa:
Gere cenários de teste de INTEGRAÇÃO (interação entre serviços, não validação isolada de campos) para
POST /pedidos, GET /pedidos/{id} e POST /pedidos/{id}/cancelamento.

Cobertura obrigatória:

- fluxo feliz; fluxos alternativos; ordem das chamadas entre serviços;
- efeitos colaterais (estado em Estoque e Pagamentos após cada fluxo);
- cancelamento (compensação).

Regras:

- Use somente status codes e campos presentes no contrato. Se precisar assumir algo, marque como [HIPÓTESE].
- Não repita cenários que só diferem em valor de dado.
- Cada caso deve ser verificável por asserções objetivas.

Formato (tabela Markdown): ID | Título | Pré-condições | Passos | Resultado esperado (resposta + efeitos nos serviços) | Prioridade (Alta/Média/Baixa)
Gere, no mínimo, 10 casos.
