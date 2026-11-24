# Passo 1: Auditoria Agêntica do C4 Nível 2 (Contêineres) & Detecção de Gargalos

## Análise Arquitetural (Topologia SafeLine)

1. **Pontos de Falha Única (SPOF):** Na arquitetura inicial, o `SafeLine Core` concentrava todas as responsabilidades síncronas (API REST, ingestão de detecções de vídeo, persistência e disparo de alertas) conectado diretamente a uma única `Base de Dados SafeLine`, tornando o núcleo e o banco gargalos críticos em picos de detecção de câmeras.
2. **Acoplamento Síncrono Excessivo:** O `Motor de Visao Computacional` enviava detecções de forma síncrona para o núcleo, e o núcleo chamava diretamente o `Servico Corporativo de Notificacoes`. Qualquer lentidão na rede, no banco de dados ou indisponibilidade do serviço de e-mail bloqueava o processamento de frames em tempo real.
3. **Otimizações Aplicadas no C4 L2:**
   - Introdução de um **Broker de Mensageria (`RabbitMQ`)** para desacoplar a ingestão de detecções visuais e o envio de notificações.
   - Separação de um **Worker de Notificacoes e Retentativas** dedicado ao consumo assíncrono e tratamento de *Dead Letter Queue (DLQ)*.
   - Adição de um **Cache de Perfis e Sessoes (`Redis`)** para leitura de regras de EPI/ergonomia com baixa latência e suporte a *Circuit Breaker* / fallback.

---

## Diagrama C4 Nível 2 — Contêineres (Otimizado)

```mermaid
C4Container
    title Diagrama de Conteineres (C4 Nivel 2) - SafeLine (Otimizado)

    Person(eng, "Engenheiro de Seguranca", "Define perfis de seguranca, regras de EPI e analisa indicadores")
    Person(tec, "Tecnico de Seguranca", "Realiza auditorias, valida ocorrencias e correcoes")
    Person(sup, "Supervisor de Producao", "Executa planos de acao sobre Nao Conformidades")

    System_Ext(vms, "Sistema de Videomonitoramento (VMS)", "Fornece streams de cameras industriais em tempo real")
    System_Ext(rh, "Sistema Corporativo de Identidade e RH", "Gere acessos, cargos e setores")
    System_Ext(notif, "Servico Corporativo de Notificacoes", "Envia alertas por e-mail e mensagens")

    Container_Boundary(safeline, "Sistema SafeLine") {
        Container(spa, "SafeLine Web App", "React / TypeScript", "Interface web para auditoria, gestao de NCs e paineis")
        Container(api, "API Gateway / SafeLine Core", "Java / Spring Boot", "Orquestra regras de negocio, perfis, ocorrencias e NCs")
        Container(cv, "Motor de Visao Computacional", "Python / OpenCV / PyTorch", "Processa streams de video e deteta desvios de EPI e ergonomia")
        Container(worker, "Worker de Notificacoes e Retentativas", "Spring Boot Worker", "Consome eventos da fila e realiza envios assincronos com retentativa")
        ContainerQueue(mq, "Broker de Mensageria", "RabbitMQ", "Desacopla detecoes visuais, eventos de NC e alertas assincronos")
        ContainerDb(db, "Base de Dados SafeLine", "PostgreSQL", "Armazena perfis, ocorrencias, NCs, planos de acao e evidencias")
        ContainerDb(cache, "Cache de Perfis e Sessoes", "Redis", "Cache distribuido para regras ativas por posto e fallback")
    }

    Rel(eng, spa, "Configura perfis e consulta indicadores", "HTTPS")
    Rel(tec, spa, "Audita ocorrencias e valida correcoes", "HTTPS")
    Rel(sup, spa, "Regista planos de acao corretiva", "HTTPS")

    Rel(spa, api, "Consome endpoints REST", "JSON/HTTPS")
    Rel(api, rh, "Autentica utilizadores e consulta estrutura", "HTTPS/OIDC")

    Rel(vms, cv, "Transmite stream de video", "RTSP/WebRTC")
    Rel(cv, mq, "Publica evento de detecao visual", "AMQP")
    Rel(mq, api, "Entrega evento de detecao para criacao de ocorrencia", "AMQP")

    Rel(api, cache, "Le regras ativas de posto com baixa latencia", "TCP")
    Rel(api, db, "Le e persiste perfis, ocorrencias e NCs", "JDBC/TLS")
    Rel(api, mq, "Publica eventos de alerta e notificacao", "AMQP")
    Rel(mq, worker, "Entrega mensagens de alerta e DLQ", "AMQP")
    Rel(worker, notif, "Dispara alertas de ocorrencias e NCs", "HTTPS/SMTP")
```
