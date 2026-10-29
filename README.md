# Testes de Integração com GenAI – Serviço de Pedidos

Repositório da atividade do fórum da Unidade III do módulo 13 - Testes de Software Automatizados com Inteligência Artificial Generativa. Mostra o uso de diferentes tipos de prompt para gerar casos de teste de integração entre serviços REST.

## Cenário

Fluxo de criação de pedido em um e-commerce:

```
Cliente/Front-end --> Pedidos API --> Estoque API   (reserva de itens)
                              \-----> Pagamentos API (autorização do pagamento)
```

Contrato resumido em [docs/openapi.yaml](docs/openapi.yaml).

## Tipos de prompt utilizados

| #   | Tipo de prompt                                             | Objetivo                                         | Prompts                                                                    | Casos gerados                                                  |
| --- | ---------------------------------------------------------- | ------------------------------------------------ | -------------------------------------------------------------------------- | -------------------------------------------------------------- |
| 1   | Geração de cenários (role + contexto + formato)            | Cobrir fluxo feliz e alternativos entre serviços | [prompts/01-integration-scenarios.md](prompts/01-integration-scenarios.md) | [tests/01-integration-cases.md](tests/01-integration-cases.md) |
| 2   | Casos de erro/resiliência (baseado em contrato + few-shot) | Falhas, timeouts, inconsistência e idempotência  | [prompts/02-error-cases.md](prompts/02-error-cases.md)                     | [tests/02-error-cases.md](tests/02-error-cases.md)             |
| 3   | Dados de teste (restrições + saída estruturada)            | Massa de dados válida, limite e inválida         | [prompts/03-test-data.md](prompts/03-test-data.md)                         | [data/orders.json](data/orders.json)                           |

## Estrutura

```
docs/           contrato OpenAPI usado como contexto nos prompts
prompts/        prompts executados
tests/          casos de teste gerados e revisados
data/           massa de dados gerada
```
