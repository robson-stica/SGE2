# ADR 0002: Adoção de Cache Distribuído (Redis) e Circuit Breaker para Tolerância a Falhas

## Título
Uso de Redis (`Cache de Perfis e Sessoes`) e padrão Circuit Breaker para mitigar latência e falhas na `Base de Dados SafeLine`

## Status
Aceito

## Contexto
Durante a análise de Chaos e Resiliência no fluxo principal do **SafeLine**, identificou-se que o `API Gateway / SafeLine Core` depende da `Base de Dados SafeLine` (PostgreSQL) tanto para consultar regras ativas de EPI/ergonomia por posto quanto para persistir novas ocorrências geradas pelo `Motor de Visao Computacional`.

Caso a base relacional sofra degradação de desempenho (latência superior a `3000ms`) ou indisponibilidade temporária, novas requisições poderiam esgotar o pool de conexões da API, causando indisponibilidade em cascata na plataforma web e perda de eventos de segurança.

## Decisão
Decidimos adotar o **Redis** (`Cache de Perfis e Sessoes`) combinado com o padrão **Circuit Breaker** e **Fallback**:
- **Leitura em baixa latência:** perfis de segurança e regras por posto ficam armazenados em cache no Redis via interface `IPerfilCache`, reduzindo a carga de leitura no PostgreSQL.
- **Timeout explícito e Circuit Breaker:** chamadas de escrita à `Base de Dados SafeLine` possuem timeout máximo de `3000ms`. Ao detectar falhas recorrentes, o *Circuit Breaker* abre temporariamente para proteger o banco.
- **Fallback e reprocessamento:** enquanto o circuito está aberto, o payload da ocorrência é salvo temporariamente no Redis (`salvarFallbackOcorrencia`) e reenfileirado na `retry-db-queue` do `Broker de Mensageria` para persistência assíncrona após recuperação.

## Consequências Positivas
- **Proteção contra falha em cascata:** evita esgotamento de threads e conexões quando o PostgreSQL está lento ou instável.
- **Menor latência nas consultas frequentes:** regras de postos de trabalho são servidas em memória pelo Redis.
- **Preservação de eventos críticos:** nenhuma detecção visual é perdida durante janelas curtas de instabilidade do banco.

## Consequências Negativas (Trade-offs)
- **Invalidação de cache:** alterações feitas pelo Engenheiro de Segurança nos perfis exigem estratégia de invalidação ou TTL adequado para evitar leitura de regras desatualizadas.
- **Infraestrutura adicional:** requer provisionamento e monitoramento de uma instância Redis além do PostgreSQL e do RabbitMQ.
- **Consistência eventual em modo fallback:** ocorrências enfileiradas na `retry-db-queue` levam alguns segundos extras até ficarem visíveis para auditoria.
