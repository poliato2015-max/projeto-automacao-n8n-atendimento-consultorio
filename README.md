# Automacao N8N — Atendimento Consultorio Odontologico

> Fluxo de atendimento automatico via WhatsApp para consultorios odontologicos — com IA Generativa, transcricao de audio, buffer de mensagens picadas, historico de conversas e gerenciamento de agendamentos via Google Calendar.

[![N8N](https://img.shields.io/badge/Automacao-N8N-orange)](https://n8n.io)
[![WhatsApp](https://img.shields.io/badge/WhatsApp-API%20Oficial%20Meta-25D366?logo=whatsapp)](https://developers.facebook.com/docs/whatsapp)
[![OpenAI](https://img.shields.io/badge/LLM-GPT--4o%20mini-412991?logo=openai)](https://openai.com)
[![Supabase](https://img.shields.io/badge/Backend-Supabase-3ECF8E?logo=supabase)](https://supabase.com)
[![Redis](https://img.shields.io/badge/Buffer-Redis%20via%20Upstash-DC382D?logo=redis)](https://upstash.com)
[![Google Calendar](https://img.shields.io/badge/Agenda-Google%20Calendar-4285F4?logo=googlecalendar)](https://calendar.google.com)
[![Status](https://img.shields.io/badge/Status-Em%20Desenvolvimento-yellow)]()

---

## 📋 Sobre o projeto

Este projeto foi desenvolvido como parte da minha imersao pratica em **Inteligencia Artificial aplicada a automacao de processos**, utilizando o N8N como orquestrador de um fluxo completo de atendimento automatico para consultorios odontologicos via WhatsApp.

O fluxo substitui o atendimento manual de recepcao para tarefas rotineiras como agendamento, reagendamento e cancelamento de consultas, alem de responder duvidas frequentes dos pacientes — tudo de forma automatica, humanizada e disponivel 24 horas.

> *"O maior aprendizado foi entender que automacao com IA nao e sobre substituir pessoas — e sobre liberar a equipe para o que realmente importa: o cuidado com o paciente."*

---

## 🎯 Problema que resolve

Consultorios odontologicos enfrentam diariamente:

- Alto volume de mensagens no WhatsApp fora do horario comercial
- Tempo da recepcao consumido em agendamentos manuais repetitivos
- Pacientes que enviam mensagens fragmentadas e esperam respostas rapidas
- Dificuldade em manter historico organizado de conversas por paciente
- Ausencia de integracao entre atendimento e agenda do consultorio

---

## 🏗️ Arquitetura do fluxo

![Fluxo Completo](https://raw.githubusercontent.com/poliato2015-max/imagens/main/projeto-automacao-n8n-atendimento-consultorio-completo.png)

O fluxo e composto por 8 grupos de nodes que trabalham em sequencia:

```
1. Receber Mensagens        — WhatsApp Trigger via API Meta
2. Seleciona Campos         — Normalizacao dos dados de entrada
3. Banco de Dados           — Verificacao e cadastro do paciente
4. Tratativa da Mensagem    — Classificacao e transcricao (texto ou audio)
5. Buffer (Banco Redis)     — Agrupamento de mensagens picadas
6. Agente de IA             — Processamento e decisao com GPT-4o mini
7. Gerenciamento de Agendamentos — Google Calendar via MCP
8. Divisao de Mensagens     — Envio humanizado da resposta
```

---

## 🛠️ Tecnologias utilizadas

| Tecnologia | Uso no projeto |
|---|---|
| **N8N** | Orquestrador do fluxo de automacao |
| **WhatsApp Cloud API (Meta)** | Recepcao e envio de mensagens |
| **OpenRouter** | Gateway de acesso aos modelos de LLM |
| **GPT-4o mini (OpenAI)** | Modelo de IA conversacional em producao |
| **Groq Whisper** | Transcricao de mensagens de audio para texto |
| **Supabase (PostgreSQL)** | Banco de dados de pacientes e historico |
| **Redis via Upstash** | Buffer de mensagens picadas |
| **Google Calendar** | Gerenciamento de agendamentos via MCP |

---

## 📂 Estrutura do repositorio

```
repositorio/
├── README.md
├── workflow/
│   └── fluxo_n8n_consultorio.json
└── docs/
    ├── system_prompt_agente.md
    ├── prompt_listar_eventos.md
    ├── prompt_criar_evento.md
    ├── prompt_reagendar_evento.md
    └── prompt_deletar_evento.md
```

---

## ⚙️ Pre-requisitos e Configuracao

### Servicos necessarios
- Conta no N8N (self-hosted ou cloud)
- Conta de desenvolvedor Meta com WhatsApp Business API configurada
- Conta no OpenRouter com credito ativo
- Conta no Groq (gratuita)
- Projeto no Supabase
- Conta no Upstash (Redis gratuito)
- Google Calendar com credenciais OAuth configuradas

### Criacao da tabela de pacientes no Supabase

O fluxo inclui um node utilitario isolado chamado **"Criar Tabela Dados Paciente"** — localizado acima do fluxo principal no canvas do N8N. Basta configurar suas credenciais do Supabase neste node e execute-lo manualmente uma unica vez antes de ativar o fluxo principal.

```sql
create table pacientes (
    id bigserial primary key,
    created_at TIMESTAMPTZ,
    telefone text,
    nome_paciente text,
    nome_confirmado text
);
```

| Campo | Tipo | Funcao |
|---|---|---|
| `id` | bigserial (PK) | Identificador unico do paciente |
| `created_at` | TIMESTAMPTZ | Data e hora do primeiro contato |
| `telefone` | text | Numero do WhatsApp do paciente |
| `nome_paciente` | text | Nome capturado durante a conversa |
| `nome_confirmado` | text | Nome confirmado pelo paciente |

### Tabela de historico de conversas

A tabela `n8n_chat_histories` e criada **automaticamente** pelo N8N na primeira execucao do fluxo — nenhuma configuracao manual e necessaria no Supabase.

| Campo | Tipo | Funcao |
|---|---|---|
| `id` | int4 | Identificador do registro |
| `session_id` | varchar | Numero de telefone do paciente (Session ID) |
| `message` | jsonb | Conteudo da mensagem com tipo (human/ai) |

---

## 🔄 Descricao detalhada do fluxo

### Etapa 1 — Receber Mensagens

O fluxo e iniciado pelo node **WhatsApp Trigger**, um webhook que monitora continuamente a chegada de mensagens dos pacientes via API oficial da Meta. A cada nova mensagem recebida, o fluxo e disparado automaticamente.

---

### Etapa 2 — Seleciona Campos

O node **Agrega campos** filtra e normaliza os dados do payload recebido da API Meta, extraindo apenas as informacoes relevantes para os proximos nodes do fluxo — numero de telefone, tipo de mensagem e conteudo.

---

### Etapa 3 — Banco de Dados

Apos normalizar os dados de entrada, o fluxo consulta o Supabase pelo numero de telefone do paciente:

- **Buscar Cliente** — executa uma query para verificar se o contato ja existe na base
- **Telefone Cadastrado?** — node condicional que avalia o resultado:
  - `true` — paciente ja cadastrado, segue diretamente para a proxima etapa
  - `false` — paciente novo, o node **Gravar Lead na Base** registra automaticamente o novo paciente no banco antes de continuar

---

### Etapa 4 — Tratativa da Mensagem

O node **Classifica mensagens** identifica o tipo de mensagem recebida e direciona para o caminho correto:

- **Audio** — o node **Download audio** faz o download do arquivo via API da Meta e o envia para o node **Transcricao audio**, que converte a fala em texto via Groq Whisper
- **Texto** — o node **Normaliza Mensagem** padroniza o conteudo diretamente
- **Outros tipos** (imagem, documento, sticker) — o node **Mensagem invalida** responde automaticamente ao paciente: *"Oi! Tudo bem? Nao conseguimos abrir arquivos anexados, so aceitamos texto ou audio"*

Ao final, o node **Mensagem Final** unifica ambos os caminhos (audio transcrito e texto), garantindo que o proximo node sempre receba uma mensagem em formato texto padronizado.

---

### Etapa 5 — Buffer (Banco Redis)

Um dos principais desafios em chatbots via WhatsApp e lidar com mensagens enviadas de forma fragmentada — quando o paciente divide uma ideia em varias mensagens consecutivas. Para resolver isso, o fluxo implementa um buffer inteligente com Redis via Upstash:

- **Buscar Historico** — verifica se ja existe alguma mensagem acumulada no buffer para aquele paciente
- **Atualizar Texto Acumulado** — adiciona a nova mensagem ao buffer do paciente
- **Ler Bloco Final** — le o conteudo completo acumulado no buffer
- **Wait** — aguarda 20 segundos para capturar todas as mensagens do bloco
- **Validar Ultima Mensagem** — garante que apenas a ultima mensagem do bloco dispara o processamento, evitando respostas duplicadas
- **Limpar Buffer** — limpa o buffer do Upstash apos consolidar todas as mensagens
- **Agrupa Mensagens** — une todas as mensagens acumuladas em uma unica string e envia para o Agente de IA

---

### Etapa 6 — Agente de IA

O coracao do fluxo. O node **AI Agent** recebe a mensagem consolidada e processa a resposta com base no system prompt configurado e nas ferramentas disponíveis:

**Modelo:** OpenRouter LLM 1 — GPT-4o mini (OpenAI via OpenRouter)

**Tools disponiveis para o Agente:**

| Tool | Funcao |
|---|---|
| `historico_atendimento_pacientes` | Memoria — recupera as ultimas 20 interacoes do paciente para contexto |
| `Atualiza nome confirmado` | Atualiza o campo `nome_confirmado` na tabela `pacientes` quando o paciente informa seu nome |
| `agendamentos` | Aciona o MCP do Google Calendar para gerenciar consultas |

**Structured Output Parser:** estrutura a saida do modelo em formato padronizado para o proximo node.

O system prompt completo do agente esta disponivel em: [System Prompt do Agente](docs/system_prompt_agente.md)

---

### Etapa 7 — Gerenciamento de Agendamentos (MCP)

Quando o Agente de IA identifica uma intencao de agendamento na mensagem do paciente, aciona automaticamente o **MCP Agendamento** — um servidor MCP integrado ao Google Calendar com 4 tools disponiveis:

| Tool | Operacao | Funcao | Prompt |
|---|---|---|---|
| `listar_eventos` | getAll | Lista consultas agendadas do paciente | [Ver prompt](docs/prompt_listar_eventos.md) |
| `criar_evento` | create | Agenda uma nova consulta | [Ver prompt](docs/prompt_criar_evento.md) |
| `reagendar_evento` | update | Altera data/hora de uma consulta existente | [Ver prompt](docs/prompt_reagendar_evento.md) |
| `deletar_evento` | delete | Cancela uma consulta agendada | [Ver prompt](docs/prompt_deletar_evento.md) |

Cada tool possui um prompt especifico que orienta o modelo sobre como executar a operacao corretamente no Google Calendar.

---

### Etapa 8 — Divisao de Mensagens

Para garantir uma experiencia de atendimento mais natural e agradavel, a resposta gerada pelo Agente de IA nao e enviada como um unico bloco de texto:

- **Divide Mensagens** — fragmenta a resposta em partes menores
- **Split Out** — separa cada parte em itens individuais
- **Loop Over Items** — itera sobre cada parte sequencialmente
- **Send message** — envia cada parte individualmente via WhatsApp API da Meta

O paciente recebe mensagens menores e mais naturais, simulando uma conversa humana fluida.

---

## 🧠 Cicatrizes de aprendizado

**1. Escolha do modelo de LLM**

Durante o desenvolvimento, testei tres modelos via OpenRouter: GPT-4o mini (OpenAI), Claude Sonnet 3.5 (Anthropic) e Tencent Hy3 (gratuito). O Tencent Hy3 foi descartado por apresentar alucinacoes nas respostas, comprometendo a confiabilidade do atendimento. O Claude Sonnet 3.5 performou bem, porem o GPT-4o mini foi escolhido como modelo definitivo pela melhor relacao entre custo, velocidade de resposta e qualidade nas interacoes de atendimento odontologico. O Groq foi utilizado exclusivamente para transcricao de audio via Whisper.

**2. Buffer Redis para mensagens picadas**

Pacientes frequentemente enviam mensagens fragmentadas no WhatsApp — uma ideia dividida em 3 ou 4 mensagens consecutivas. Sem o buffer, o Agente de IA respondia cada fragmento separadamente, gerando respostas sem contexto e experiencia ruim. A solucao com Redis via Upstash e um Wait de 20 segundos resolveu completamente este problema.

**3. Criacao automatica da tabela de historico**

Ao configurar o node de memoria Postgres Chat Memory no N8N, descobri que a tabela `n8n_chat_histories` e criada automaticamente no Supabase na primeira execucao — sem necessidade de configuracao manual. O numero de telefone do paciente e usado como Session ID, garantindo historico isolado por paciente.

**4. Context Window Length**

O Context Window Length do historico de conversas esta configurado com 20 interacoes — valor que garante contexto suficiente para conversas mais longas sem perder o fio da conversa. Este valor e configuravel conforme a necessidade do atendimento, equilibrando contexto e custo de tokens.

**5. Fallback para mensagens nao suportadas**

Inicialmente o fluxo nao tratava mensagens de tipos nao suportados (imagens, documentos, stickers), deixando o paciente sem resposta. A adicao do node **Mensagem invalida** no caminho Fallback do Switch resolveu a lacuna, orientando o paciente de forma educada sobre os tipos aceitos.

---

## 🔮 Proximas evolucoes

- Adicionar delay entre mensagens no Loop Over Items para evitar interpretacao como spam pelo WhatsApp
- Implementar tratamento do Fallback com encaminhamento para atendente humano quando necessario
- Adicionar confirmacao de agendamento por mensagem apos criacao do evento no Google Calendar
- Expandir para outros tipos de consultorios (medicos, psicologicos, fisioterapia)
- Implementar relatorio periodico de atendimentos via dashboard no Supabase

---

## 👨‍💻 Autor

Desenvolvido por **Marcelo Poliato de Oliveira** como projeto pratico de automacao inteligente com N8N e IA Generativa.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Marcelo%20Poliato-0077B5?logo=linkedin)](https://www.linkedin.com/in/marcelo-poliato)
[![GitHub](https://img.shields.io/badge/GitHub-poliato2015--max-181717?logo=github)](https://github.com/poliato2015-max)
