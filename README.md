# 🏡 E-Urban AI - Sistema de Vendas Imobiliárias com Agentes Adaptativos

> ⚠️ **Nota de Confidencialidade:** Este repositório contém a documentação técnica (Case Study) e arquitetural do projeto. O código-fonte sensível e os dados de clientes foram omitidos em conformidade com contratos de confidencialidade (NDA).

## 🎯 O Desafio
O cliente (um lançamento imobiliário de alto padrão, *Aldeia dos Lagos*) precisava automatizar o atendimento no WhatsApp para escalar suas vendas. O problema principal não era apenas "responder dúvidas", mas **qualificar e converter leads** com perfis comportamentais completamente diferentes.

Um chatbot linear falhava porque tratava um **Investidor** (focado em ROI/lucro) da mesma forma que uma **Família** (focada em lazer/segurança), resultando em baixa conversão.

## 💡 A Solução: Personas Dinâmicas
Desenvolvi uma arquitetura de **Agentes Especializados** orquestrados via **n8n**. O sistema não segue uma árvore de decisão fixa; ele atua em etapas cognitivas:

1.  **Triagem & Multimodalidade:** Um agente "Recepcionista" recebe o lead. Se for áudio, o sistema transcreve (Whisper). O agente identifica a origem (Instagram, Google, Indicação) e classifica o interesse.
2.  **Roteamento de Personalidade (Router):** Com base na classificação, o chat é transferido para um Agente Especialista com *System Prompt* e Tom de Voz dedicados:
    * 🤵 **Agente Investidor:** Foco em racionalidade, números, escassez e preço de custo.
    * 🏡 **Agente Moradia:** Foco em emocional, realização de sonho e qualidade de vida.
    * 🌳 **Agente Segunda Moradia:** Foco em refúgio, escapismo e tranquilidade.
3.  **Conversão (Agendamento):** O objetivo final é agendar uma visita. O sistema integra com **Supabase** para verificar datas de eventos e reservar vagas.

## 🔔 Sistema de Follow-up de Eventos (Anti-No-Show)
Para resolver o problema de leads que agendam e não comparecem (*No-Show*), foi desenvolvido um módulo de retenção ativa:

* **Régua de Comunicação Automatizada:** O sistema dispara lembretes estratégicos em três momentos críticos: **24h antes**, **1h antes** e **15min antes** do evento.
* **Agendamento Inteligente (CRON):** Jobs configurados para diferenciar a lógica de eventos de meio de semana (Terça/Quinta à noite) e finais de semana (Domingo de manhã).
* **Eficiência de Dados:** Utilização de **RPCs (Stored Procedures)** no banco de dados para filtrar apenas os leads com eventos no dia, otimizando o processamento em lote.

## 🛠️ Stack Tecnológica

| Componente | Tecnologia | Função |
| :--- | :--- | :--- |
| **Cérebro** | OpenAI (GPT-4o) | Inteligência conversacional e adaptação de persona. |
| **Audição** | OpenAI Whisper | Transcrição de áudios longos do WhatsApp em tempo real. |
| **Memória Técnica** | Pinecone (Vector DB) | RAG para consultas jurídicas e técnicas do empreendimento. |
| **Dados & Fila** | Supabase (PostgreSQL) | Gestão de leads, agendamentos e fila de mensagens (Debounce). |
| **Orquestração** | n8n (Self-hosted) | Lógica de fluxo, tratamento de erros, Cron Jobs e integrações. |

---

## 📐 Arquitetura do Sistema

O diagrama abaixo detalha o fluxo de dados completo, incluindo a entrada multimodal, o roteamento de personas e o sistema de follow-up ativo.

```mermaid
graph TD
    %% Estilos (Texto em Preto para legibilidade)
    classDef user fill:#f9f,stroke:#333,stroke-width:2px,color:#000;
    classDef ai fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#000;
    classDef db fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#000;
    classDef logic fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:#000;

    User([👤 Lead WhatsApp]):::user <-->|Texto ou Áudio| Evo[Evolution API]
    Evo -->|Webhook| Queue{⚡ Fila & Debounce<br/>PostgreSQL}:::logic

    subgraph "Módulo Receptivo (Vendas)"
        Queue -->|Se Áudio| Whisper[🎙️ Transcrição Whisper]:::ai
        Queue -->|Se Texto| Recepcionista
        Whisper --> Recepcionista[👩‍💼 Agente Triagem]:::ai
        
        Recepcionista -->|Classificar Perfil| Router{🔀 Router}:::logic
        
        Router -->|Perfil: Lucro| Investidor[🤵 Agente Investidor]:::ai
        Router -->|Perfil: Casa| Moradia[🏡 Agente Moradia]:::ai
        Router -->|Perfil: Lazer| Lazer[🌳 Agente 2ª Moradia]:::ai
        
        Investidor & Moradia & Lazer <-->|RAG / Dúvidas| Pinecone[(🧠 Vector DB)]:::db
        Investidor & Moradia & Lazer <-->|Agendar| Supabase[(🗄️ Supabase)]:::db
    end

    subgraph "Módulo Ativo (Follow-up)"
        Cron[⏰ Cron Jobs<br/>(24h / 1h / 15min)] -->|RPC Call| Supabase
        Supabase -->|Lista de Confirmados| Batch[⚡ n8n Batch Process]:::logic
        Batch -->|Lembrete Personalizado| Evo
    end
    
    Investidor & Moradia & Lazer -->|Resposta Personalizada| Evo
    Supabase -->|Notificar Time Comercial| Humano[👨‍💻 Chatwoot / Humano]:::user
````

## 🚀 Engenharia de Prompt e Segurança

  * **Function Calling:** O sistema utiliza saídas estruturadas (JSON) para garantir que a IA nunca agende um horário indisponível ou invente datas.
  * **Anti-Alucinação (Guardrails):** Regras estritas no *System Prompt* impedem que a IA prometa "lucro garantido" ou use termos juridicamente arriscados (ex: troca "compra e venda" por "adesão à cooperativa").
  * **Debouncing de Mensagens:** Implementação de lógica via banco de dados para tratar mensagens "encavaladas" (várias mensagens seguidas do mesmo usuário), garantindo que a IA responda apenas uma vez ao contexto completo.

-----

*Desenvolvido por [Jeferson Junior](https://www.linkedin.com/in/jeferson-junior-as/)*
