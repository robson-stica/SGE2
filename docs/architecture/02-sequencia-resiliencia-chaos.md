# Passo 2: Engenharia de Chaos e Resiliência no Diagrama de Sequência

Este diagrama evolui o fluxo principal do **SafeLine** (detecção visual de desvio de EPI/ergonomia, registro de ocorrência e envio de notificação) injetando cenários realistas de falha:
1. **Latência/Timeout na Base de Dados (`> 3s`)**: tratado com *Timeout*, *Circuit Breaker*, fallback em cache e reenfileiramento assíncrono.
2. **Indisponibilidade do Serviço Externo de Notificações**: tratado com retentativas assíncronas via mensageria (*backoff* exponencial) e *Dead Letter Queue (DLQ)*.

> Todos os nomes dos participantes correspondem exatamente aos contêineres e sistemas externos definidos no [Diagrama C4 Nível 2](01-auditoria-c4-nivel2.md).

```mermaid
sequenceDiagram
    autonumber
    actor Tec as Tecnico de Seguranca
    participant VMS as Sistema de Videomonitoramento (VMS)
    participant CV as Motor de Visao Computacional
    participant MQ as Broker de Mensageria
    participant Core as API Gateway / SafeLine Core
    participant Cache as Cache de Perfis e Sessoes
    participant DB as Base de Dados SafeLine
    participant Worker as Worker de Notificacoes e Retentativas
    participant Notif as Servico Corporativo de Notificacoes

    Note over VMS, CV: 1. Monitoramento e Detecao Visual
    VMS->>CV: Fornece stream de video em tempo real
    CV->>CV: Analisa conformidade de EPI e ergonomia

    alt Desvio visual detetado pela IA
        CV->>MQ: Publica evento DetecaoRegistrada (frame, posto, tipoDesvio)
        MQ->>Core: Entrega evento DetecaoRegistrada

        Note over Core, DB: 2. Persistencia com Timeout de 3s (Chaos: Lentidao no Banco)
        Core->>DB: Persiste Ocorrencia (timeout = 3000ms)

        alt Base de Dados responde em ate 3 segundos
            DB-->>Core: Ocorrencia criada (Estado: PENDENTE_VALIDACAO)
            Core->>MQ: Publica evento NotificacaoSolicitada
        else Timeout maior que 3 segundos ou falha de conexao
            DB--xCore: TimeoutException (> 3000ms)
            Core->>Cache: Grava payload temporario em fallback
            Core->>MQ: Reenfileira em fila de retentativa (retry-db-queue)
            Note over Core, MQ: Circuit Breaker ativado - reprocessamento assincrono agendado
        end

        Note over MQ, Notif: 3. Envio Assincrono de Alerta (Chaos: Servico de Notificacoes Indisponivel)
        MQ->>Worker: Consome evento NotificacaoSolicitada
        Worker->>Notif: Solicita envio de alerta de ocorrencia

        alt Servico de Notificacoes disponivel (200 OK)
            Notif-->>Worker: Confirmacao de envio
            Worker-->>Tec: Notifica ocorrencia pendente de auditoria
        else Servico de Notificacoes indisponivel (503 / Timeout)
            Notif--xWorker: Erro 503 Service Unavailable
            Worker->>MQ: NACK e publica em fila de retentativa (retry-notif-queue)

            alt Retentativa bem-sucedida apos backoff exponencial
                MQ->>Worker: Reentrega evento NotificacaoSolicitada
                Worker->>Notif: Reenvia alerta de ocorrencia
                Notif-->>Worker: 200 OK
                Worker-->>Tec: Notifica ocorrencia pendente de auditoria
            else Excedeu limite maximo de retentativas
                MQ->>MQ: Encaminha para Dead Letter Queue (dlq-notificacoes)
                Note over MQ, Worker: Alerta operacional registado para reprocessamento posterior
            end
        end
    end
```
