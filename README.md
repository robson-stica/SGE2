# SafeLine — Governança de Arquitetura e Modelagem (SGE2)
Repositório da Avaliação Final (SGE2) — Disciplina de Modelagem de Software | Sistema SafeLine

## 1. Diagrama de Sequência (Fluxo de Análise Visual e Tratamento de NC)

```mermaid
sequenceDiagram
    autonumber
    actor Eng as Engenheiro de Seguranca
    actor Tec as Tecnico de Seguranca
    actor Sup as Supervisor de Producao
    participant VMS as Sistema de Videomonitoramento (VMS)
    participant CV as Motor de Visao Computacional
    participant Core as SafeLine Core
    participant DB as Base de Dados SafeLine
    participant Notif as Servico de Notificacoes

    Note over Eng, DB: 1 e 2. Configuracao de Posto e Perfil de Seguranca
    Eng->>Core: Define Perfil de Seguranca, Regras de EPI e Ergonomia
    Core->>DB: Guarda configuracoes do Posto e Atividade

    Note over VMS, Core: 3, 4 e 5. Monitoramento, Detecao e Ocorrencia
    VMS->>CV: Fornece stream de video em tempo real
    CV->>CV: Analisa conformidade de EPI e ergonomia
    
    alt Desvio visual detetado pela IA
        CV->>Core: Regista Detecao (frame, posto, tipoDesvio)
        Core->>DB: Cria Ocorrencia (Estado: PENDENTE_VALIDACAO)
        Core->>Notif: Solicita envio de alerta de ocorrencia
        Notif-->>Tec: Notifica ocorrencia pendente de auditoria
    end

    Note over Tec, Notif: 6 e 7. Validacao Humana e Nao Conformidade
    Tec->>Core: Consulta Ocorrencia e evidencia visual
    
    alt Ocorrencia Validada (Desvio Real Confirmado)
        Tec->>Core: Aprova Ocorrencia e gera Nao Conformidade (NC)
        Core->>DB: Atualiza Ocorrencia (VALIDADA) e cria NC (ABERTA)
        Core->>Notif: Solicita alerta de NC para a Producao
        Notif-->>Sup: Notifica Nao Conformidade aberta no posto
        
        Note over Sup, Tec: 8 e 9. Plano de Acao e Validacao da Correcao
        Sup->>Core: Regista Plano de Acao Corretiva
        Sup->>Core: Conclui acao e solicita validacao da correcao
        Core->>DB: Atualiza NC (AGUARDANDO_VALIDACAO)
        Core->>Notif: Notifica Tecnico para validar correcao
        Notif-->>Tec: Alerta de validacao de correcao pendente
        Tec->>Core: Realiza auditoria e valida a correcao
        Core->>DB: Encerra Nao Conformidade (CONCLUIDA)
    else Falso Positivo da IA
        Tec->>Core: Descarta Ocorrencia (Falso Positivo)
        Core->>DB: Atualiza Ocorrencia (DESCARTADA)
    end
```

## 2. Diagrama de Componentes (Arquitetura Interna do SafeLine)

```mermaid
flowchart TB
    Tec["Tecnico de Seguranca<br/>(Auditoria e Validacao)"]
    Eng["Engenheiro de Seguranca<br/>(Perfis, Politicas e Indicadores)"]
    Sup["Supervisor de Producao<br/>(Planos de Acao e Tratamento de NCs)"]

    VMS["Sistema de Videomonitoramento (VMS)<br/>[Sistema Externo]"]
    RH["Sistema Corporativo de Identidade e RH<br/>[Sistema Externo]"]
    Notif["Servico Corporativo de Notificacoes<br/>[Sistema Externo]"]

    subgraph SafeLine ["Sistema SafeLine (Arquitetura de Componentes)"]
        direction TB
        API["API Gateway / Controlador Web<br/>[Componente REST]"]
        
        subgraph Dominio ["Componentes de Dominio e Regras de Negocio"]
            CompPoliticas["Gestor de Perfis e Politicas<br/>Configura postos, EPIs e ergonomia"]
            CompMonitor["Motor de Analise Visual (IA/CV)<br/>Processa frames e deteta desvios"]
            CompOcorrencias["Gestor de Auditoria e Ocorrencias<br/>Exige validacao humana antes de gerar NC"]
            CompNC["Gestor de Nao Conformidades e Planos de Acao<br/>Controla ciclo de vida das NCs"]
            CompIndicadores["Modulo de Historico e Indicadores<br/>Consolida metricas de compliance"]
        end

        DB[("Base de Dados SafeLine<br/>[Persistencia Relacional e Evidencias]")]
    end

    Tec -->|"Valida ocorrencias e correcoes"| API
    Eng -->|"Define perfis e consulta indicadores"| API
    Sup -->|"Trata NCs e atualiza planos de acao"| API

    API --> CompPoliticas
    API --> CompOcorrencias
    API --> CompNC
    API --> CompIndicadores

    VMS -->|"Fornece fluxo de video"| CompMonitor
    CompPoliticas -->|"Fornece regras ativas por posto"| CompMonitor
    CompMonitor -->|"Regista detecao preliminar"| CompOcorrencias
    CompOcorrencias -->|"Promove ocorrencia validada a NC"| CompNC
    
    CompOcorrencias -->|"Solicita envio de alertas"| Notif
    CompNC -->|"Solicita alertas de prazos e acoes"| Notif
    API -->|"Autentica utilizadores e estrutura"| RH

    CompPoliticas --> DB
    CompOcorrencias --> DB
    CompNC --> DB
    CompIndicadores --> DB
```

## 3. Diagrama de Estados (Ciclo de Vida da Deteção à Não Conformidade)

```mermaid
stateDiagram-v2
    [*] --> DetecaoGerada : IA identifica desvio no video
    DetecaoGerada --> OcorrenciaPendente : Registo automatico no SafeLine
    
    OcorrenciaPendente --> AnaliseTecnica : Tecnico de Seguranca inicia auditoria
    
    AnaliseTecnica --> OcorrenciaDescartada : Falso Positivo (Descartado)
    OcorrenciaDescartada --> [*]

    AnaliseTecnica --> NaoConformidadeAberta : Desvio Confirmado (Gera NC)
    NaoConformidadeAberta --> PlanoAcaoEmAndamento : Supervisor define Plano de Acao
    PlanoAcaoEmAndamento --> AguardandoValidacao : Supervisor conclui acao corretiva
    
    AguardandoValidacao --> NaoConformidadeConcluida : Tecnico aprova correcao
    AguardandoValidacao --> PlanoAcaoEmAndamento : Tecnico reprova correcao
    
    NaoConformidadeConcluida --> [*]
```

## 4. Diagrama C4 Nível 1 (Contexto do Sistema)

```mermaid
flowchart TB
    Tec["👤 Tecnico de Seguranca<br/>[Utilizador]<br/>Realiza auditorias e valida ocorrencias"]
    Eng["👤 Engenheiro de Seguranca<br/>[Utilizador]<br/>Configura perfis de seguranca e analisa indicadores"]
    Sup["👤 Supervisor de Producao<br/>[Utilizador]<br/>Executa planos de acao sobre Nao Conformidades"]

    SafeLine["🛡️ SafeLine<br/>[Sistema de Software]<br/>Monitora conformidade de EPIs e ergonomia em linhas de producao"]

    VMS["📹 Sistema de Videomonitoramento (VMS)<br/>[Sistema Externo]<br/>Fornece streams de cameras industriais"]
    RH["🏢 Sistema Corporativo de Identidade e RH<br/>[Sistema Externo]<br/>Gere acessos, cargos e setores"]
    Notif["🔔 Servico Corporativo de Notificacoes<br/>[Sistema Externo]<br/>Envia alertas por e-mail/mensagens"]

    Tec -->|"Valida ocorrencias e correcoes"| SafeLine
    Eng -->|"Define regras e consulta relatorios"| SafeLine
    Sup -->|"Gere planos de acao corretiva"| SafeLine

    VMS -->|"Transmite fluxo de video"| SafeLine
    SafeLine -->|"Consulta dados organizacionais"| RH
    SafeLine -->|"Dispara alertas de ocorrencias e NCs"| Notif
```

## 5. Auditoria de Arquitetura, Resiliência e ADR (Laboratório Agêntico)

Documentação gerada na auditoria arquitetural do **SafeLine**:

- **Passo 1 — Auditoria C4 Nível 2 (Contêineres):** [docs/architecture/01-auditoria-c4-nivel2.md](docs/architecture/01-auditoria-c4-nivel2.md)
- **Passo 2 — Diagrama de Sequência com Chaos e Resiliência:** [docs/architecture/02-sequencia-resiliencia-chaos.md](docs/architecture/02-sequencia-resiliencia-chaos.md)
- **Passo 3 — ADR 0001 (Mensageria Assíncrona e Resiliência):** [docs/adr/0001-escolha-tecnologica.md](docs/adr/0001-escolha-tecnologica.md)
- **Complemento — Desacoplamento por Interfaces e DI:** [docs/architecture/03-componentes-interfaces-di.md](docs/architecture/03-componentes-interfaces-di.md)

### Passo 4 — Checklist de Auditoria Final
- [x] **Consistência Nomes/Métodos:** Os nomes dos contêineres e serviços no C4 L2 (`Motor de Visao Computacional`, `Broker de Mensageria`, `API Gateway / SafeLine Core`, `Cache de Perfis e Sessoes`, `Base de Dados SafeLine`, `Worker de Notificacoes e Retentativas`, `Servico Corporativo de Notificacoes`) são exatamente os mesmos usados no Diagrama de Sequência.
- [x] **Sintaxe Compilável:** Todos os blocos Mermaid (`C4Container`, `sequenceDiagram`, `classDiagram`, `flowchart`, `stateDiagram-v2`) seguem sintaxe oficial compatível com GitHub e Mermaid Live Editor.
- [x] **Desacoplamento por Interfaces:** O Diagrama de Classes em `docs/architecture/03-componentes-interfaces-di.md` demonstra uso de interfaces (`IOcorrenciaService`, `IOcorrenciaRepository`, `INotificacaoPublisher`, `IPerfilCache`) e Injeção de Dependência via construtor.
- [x] **Rastreabilidade no Git:** Artefatos organizados em `/docs/architecture/` e `/docs/adr/`.
