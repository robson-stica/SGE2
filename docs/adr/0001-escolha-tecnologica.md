# ADR 0001: Adoção de Mensageria Assíncrona com RabbitMQ e Padrões de Resiliência no SafeLine

## Título
Adoção de RabbitMQ para desacoplamento assíncrono entre Visão Computacional, SafeLine Core e Notificações

## Status
Aceito

## Contexto
No sistema **SafeLine**, o `Motor de Visao Computacional` processa streams contínuos de câmeras industriais (VMS) para detectar desvios de EPI e ergonomia em tempo real. Na arquitetura inicial, o fluxo entre detecção, persistência na `Base de Dados SafeLine` e disparo de alertas pelo `Servico Corporativo de Notificacoes` ocorria de forma síncrona.

Durante a auditoria arquitetural (C4 Nível 2 e Análise de Chaos), foram identificados dois riscos críticos:
1. **Acoplamento síncrono e efeito cascata:** se o banco de dados apresentar latência superior a 3 segundos ou se o serviço corporativo de notificações ficar indisponível (HTTP 503 / timeout), as requisições ficam retidas, degradando o `API Gateway / SafeLine Core` e interrompendo a ingestão de novas detecções.
2. **Perda de eventos de segurança:** falhas transitórias de rede poderiam resultar no descarte silencioso de ocorrências pendentes de auditoria pelo Técnico de Segurança.

## Decisão
Decidimos adotar o **RabbitMQ** como `Broker de Mensageria` central para comunicação assíncrona baseada em eventos (AMQP), combinado com:
- **Desacoplamento na ingestão:** o `Motor de Visao Computacional` publica eventos `DetecaoRegistrada` no RabbitMQ, que são consumidos pelo `API Gateway / SafeLine Core`.
- **Worker dedicado de notificações:** criação do `Worker de Notificacoes e Retentativas` para consumir eventos `NotificacaoSolicitada`, isolando falhas do `Servico Corporativo de Notificacoes`.
- **Políticas de resiliência:** uso de timeouts explícitos (`3000ms` no banco de dados), *Circuit Breaker*, filas de retentativa com *backoff* exponencial (`retry-db-queue` e `retry-notif-queue`) e *Dead Letter Queue* (`dlq-notificacoes`) para mensagens que excederem o limite de tentativas.
- **Inversão de dependência:** o domínio comunica-se exclusivamente por interfaces (`INotificacaoPublisher`), permitindo substituição ou testes isolados sem dependência direta do broker.

## Consequências Positivas
- **Alta resiliência e tolerância a falhas:** indisponibilidades no serviço de notificações ou picos de latência no banco não derrubam o núcleo nem perdem eventos de segurança.
- **Menor acoplamento e melhor escalabilidade:** o `Motor de Visao Computacional`, o `API Gateway / SafeLine Core` e o `Worker de Notificacoes e Retentativas` podem escalar de forma independente conforme a carga das linhas de produção.
- **Rastreabilidade de falhas:** mensagens com erro persistente ficam retidas na `dlq-notificacoes` para auditoria e reprocessamento seguro.

## Consequências Negativas (Trade-offs)
- **Aumento da complexidade operacional:** exige provisionamento, monitoramento e manutenção de um cluster RabbitMQ, além de filas de retry e DLQ.
- **Consistência eventual:** entre a detecção visual e a persistência ou envio da notificação existe um pequeno atraso assíncrono em vez de resposta imediata na mesma transação.
- **Necessidade de idempotência:** os consumidores (`API Gateway / SafeLine Core` e `Worker de Notificacoes e Retentativas`) devem tratar mensagens duplicadas de forma idempotente em cenários de reentrega.
