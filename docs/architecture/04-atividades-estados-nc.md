# Fluxo de Atividades e Máquina de Estados — Ciclo de Não Conformidade (SafeLine)

Este artefato complementa a modelagem comportamental do **SafeLine**, conectando os contêineres definidos no [Diagrama C4 Nível 2](01-auditoria-c4-nivel2.md) ao ciclo de vida completo de uma Ocorrência até o encerramento da Não Conformidade (NC).

## 1. Diagrama de Atividades com Raias e Fork/Join

```mermaid
flowchart TB
    Start(["Inicio"])

    subgraph VMS_CV ["Motor de Visao Computacional / VMS"]
        A1["Capturar stream de video do posto"]
        A2{"Desvio de EPI ou<br/>ergonomia detetado?"}
        A3["Publicar evento DetecaoRegistrada"]
    end

    subgraph Core_MQ ["API Gateway / SafeLine Core & Broker de Mensageria"]
        A4["Consumir evento DetecaoRegistrada"]
        A5["Persistir Ocorrencia na Base de Dados SafeLine"]
        Fork1(["Fork - Processamento Paralelo"])
        A6["Publicar evento NotificacaoSolicitada"]
        A7["Atualizar metricas no Modulo de Indicadores"]
        Join1(["Join"])
    end

    subgraph Worker_Notif ["Worker de Notificacoes e Retentativas"]
        A8["Enviar alerta ao Tecnico de Seguranca"]
    end

    subgraph Auditoria ["Tecnico de Seguranca & Supervisor de Producao"]
        A9{"Tecnico valida<br/>ocorrencia?"}
        A10["Descartar Ocorrencia - Falso Positivo"]
        A11["Gerar Nao Conformidade ABERTA"]
        A12["Supervisor regista Plano de Acao Corretiva"]
        A13{"Tecnico aprova<br/>correcao?"}
        A14["Encerrar NC como CONCLUIDA"]
    end

    End(["Fim"])

    Start --> A1 --> A2
    A2 -->|"Nao"| A1
    A2 -->|"Sim"| A3 --> A4 --> A5 --> Fork1
    Fork1 --> A6 --> A8 --> A9
    Fork1 --> A7 --> Join1
    A9 -->|"Falso Positivo"| A10 --> Join1
    A9 -->|"Confirmado"| A11 --> A12 --> A13
    A13 -->|"Reprovado"| A12
    A13 -->|"Aprovado"| A14 --> Join1
    Join1 --> End
```

---

## 2. Máquina de Estados (Ocorrência e Não Conformidade)

```mermaid
stateDiagram-v2
    [*] --> DetecaoGerada : Motor de Visao Computacional publica DetecaoRegistrada
    DetecaoGerada --> OcorrenciaPendente : SafeLine Core persiste PENDENTE_VALIDACAO
    DetecaoGerada --> AguardandoReprocessamento : Timeout DB > 3000ms (retry-db-queue)
    AguardandoReprocessamento --> OcorrenciaPendente : Reprocessamento assincrono concluido

    OcorrenciaPendente --> AnaliseTecnica : Tecnico de Seguranca inicia auditoria
    AnaliseTecnica --> OcorrenciaDescartada : Falso Positivo confirmado
    OcorrenciaDescartada --> [*]

    AnaliseTecnica --> NaoConformidadeAberta : Desvio confirmado (gera NC ABERTA)
    NaoConformidadeAberta --> PlanoAcaoEmAndamento : Supervisor define Plano de Acao
    PlanoAcaoEmAndamento --> AguardandoValidacao : Acao corretiva concluida
    AguardandoValidacao --> PlanoAcaoEmAndamento : Tecnico reprova correcao
    AguardandoValidacao --> NaoConformidadeConcluida : Tecnico aprova correcao
    NaoConformidadeConcluida --> [*]
```
