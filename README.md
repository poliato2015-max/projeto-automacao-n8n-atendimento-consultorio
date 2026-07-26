# Automação N8N — Atendimento Consultório Odontológico

> Fluxo de atendimento automático via WhatsApp para consultórios odontológicos — com IA Generativa, transcrição de áudio, buffer de mensagens picadas, histórico de conversas e gerenciamento de agendamentos via Google Calendar.

[![N8N](https://img.shields.io/badge/Automação-N8N-orange)](https://n8n.io)
[![WhatsApp](https://img.shields.io/badge/WhatsApp-API%20Oficial%20Meta-25D366?logo=whatsapp)](https://developers.facebook.com/docs/whatsapp)
[![OpenAI](https://img.shields.io/badge/LLM-GPT--5%20mini-412991?logo=openai)](https://openai.com)
[![Supabase](https://img.shields.io/badge/Backend-Supabase-3ECF8E?logo=supabase)](https://supabase.com)
[![Redis](https://img.shields.io/badge/Buffer-Redis%20via%20Upstash-DC382D?logo=redis)](https://upstash.com)
[![Google Calendar](https://img.shields.io/badge/Agenda-Google%20Calendar-4285F4?logo=googlecalendar)](https://calendar.google.com)
[![Status](https://img.shields.io/badge/Status-Em%20Desenvolvimento-yellow)]()

---

## 📋 Sobre o projeto

Este projeto foi desenvolvido como parte da minha imersão prática em **Inteligência Artificial aplicada à automação de processos**, utilizando o N8N como orquestrador de um fluxo completo de atendimento automático para consultórios odontológicos via WhatsApp.

O fluxo substitui o atendimento manual de recepção para tarefas rotineiras como agendamento, reagendamento e cancelamento de consultas, além de responder dúvidas frequentes dos pacientes — tudo de forma automática, humanizada e disponível 24 horas.

> *"O maior aprendizado foi entender que automação com IA não é sobre substituir pessoas — é sobre liberar a equipe para o que realmente importa: o cuidado com o paciente."*

---

## 🎯 Problema que resolve

Consultórios odontológicos enfrentam diariamente:

- Alto volume de mensagens no WhatsApp fora do horário comercial
- Tempo da recepção consumido em agendamentos manuais repetitivos
- Pacientes que enviam mensagens fragmentadas e esperam respostas rápidas
- Dificuldade em manter histórico organizado de conversas por paciente
- Ausência de integração entre atendimento e agenda do consultório

---

## 🏗️ Arquitetura do fluxo

![Fluxo Completo](https://raw.githubusercontent.com/poliato2015-max/imagens/main/projeto-automacao-n8n-atendimento-consultorio-completo.png)

O fluxo é composto por 8 grupos de nodes que trabalham em sequência:

```
1. Receber Mensagens        — WhatsApp Trigger via API Meta
2. Seleciona Campos         — Normalização dos dados de entrada
3. Banco de Dados           — Verificação e cadastro do paciente
4. Tratativa da Mensagem    — Classificação e transcrição (texto ou áudio)
5. Buffer (Banco Redis)     — Agrupamento de mensagens picadas
6. Agente de IA             — Processamento e decisão com OpenAI GPT-5 mini
7. Gerenciamento de Agendamentos — Google Calendar via MCP
8. Divisão de Mensagens     — Envio humanizado da resposta
```

---

## 🛠️ Tecnologias utilizadas

| Tecnologia | Uso no projeto |
|---|---|
| **N8N** | Orquestrador do fluxo de automação |
| **WhatsApp Cloud API (Meta)** | Recepção e envio de mensagens |
| **OpenAI GPT-5 mini** | Modelo de IA conversacional em produção |
| **Groq Whisper** | Transcrição de mensagens de áudio para texto |
| **Supabase (PostgreSQL)** | Banco de dados de pacientes e histórico |
| **Redis via Upstash** | Buffer de mensagens picadas |
| **Google Calendar** | Gerenciamento de agendamentos via MCP |

---

## 📂 Estrutura do repositório

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

## ⚙️ Pré-requisitos e Configuração

### Serviços necessários
- Conta no N8N (self-hosted ou cloud)
- Conta de desenvolvedor Meta com WhatsApp Business API configurada
- Conta na OpenAI com acesso ao GPT-5 mini
- Conta no Groq (gratuita)
- Projeto no Supabase
- Conta no Upstash (Redis gratuito)
- Google Calendar com credenciais OAuth configuradas

### Criação da tabela de pacientes no Supabase

O fluxo inclui um node utilitário isolado chamado **"Criar Tabela Dados Paciente"** — localizado acima do fluxo principal no canvas do N8N. Basta configurar suas credenciais do Supabase neste node e executá-lo manualmente uma única vez antes de ativar o fluxo principal.

```sql
create table pacientes (
    id bigserial primary key,
    created_at TIMESTAMPTZ,
    telefone text,
    nome_paciente text,
    nome_confirmado text
);
```

| Campo | Tipo | Função |
|---|---|---|
| `id` | bigserial (PK) | Identificador único do paciente |
| `created_at` | TIMESTAMPTZ | Data e hora do primeiro contato |
| `telefone` | text | Número do WhatsApp do paciente |
| `nome_paciente` | text | Nome capturado da API Meta WhatsApp |
| `nome_confirmado` | text | Nome confirmado pelo paciente durante a conversa |

### Tabela de histórico de conversas

A tabela `n8n_chat_histories` é criada **automaticamente** pelo N8N na primeira execução do fluxo — nenhuma configuração manual é necessária no Supabase.

| Campo | Tipo | Função |
|---|---|---|
| `id` | int4 | Identificador do registro |
| `session_id` | varchar | Número de telefone do paciente (Session ID) |
| `message` | jsonb | Conteúdo da mensagem com tipo (human/ai) |

---

## 🔄 Descrição detalhada do fluxo

### Etapa 1 — Receber Mensagens

O fluxo é iniciado pelo node **WhatsApp Trigger**, um webhook que monitora continuamente a chegada de mensagens dos pacientes via API oficial da Meta. A cada nova mensagem recebida, o fluxo é disparado automaticamente.

---

### Etapa 2 — Seleciona Campos

O node **Agrega campos** filtra e normaliza os dados do payload recebido da API Meta, extraindo apenas as informações relevantes para os próximos nodes do fluxo — número de telefone, tipo de mensagem e conteúdo.

---

### Etapa 3 — Banco de Dados

Após normalizar os dados de entrada, o fluxo consulta o Supabase pelo número de telefone do paciente:

- **Buscar Cliente** — executa uma query para verificar se o contato já existe na base
- **Telefone Cadastrado?** — node condicional que avalia o resultado:
  - `true` — paciente já cadastrado, segue diretamente para a próxima etapa
  - `false` — paciente novo, o node **Gravar Lead na Base** registra automaticamente o novo paciente no banco antes de continuar

---

### Etapa 4 — Tratativa da Mensagem

O node **Classifica mensagens** identifica o tipo de mensagem recebida e direciona para o caminho correto:

- **Áudio** — o node **Download audio** faz o download do arquivo via API da Meta e o envia para o node **Transcrição audio**, que converte a fala em texto via Groq Whisper
- **Texto** — o node **Normaliza Mensagem** padroniza o conteúdo diretamente
- **Outros tipos** (imagem, documento, sticker) — o node **Mensagem inválida** responde automaticamente ao paciente: *"Oi! Tudo bem? Não conseguimos abrir arquivos anexados, só aceitamos texto ou áudio"*

Ao final, o node **Mensagem Final** unifica ambos os caminhos (áudio transcrito e texto), garantindo que o próximo node sempre receba uma mensagem em formato texto padronizado.

---

### Etapa 5 — Buffer (Banco Redis)

Um dos principais desafios em chatbots via WhatsApp é lidar com mensagens enviadas de forma fragmentada — quando o paciente divide uma ideia em várias mensagens consecutivas. Para resolver isso, o fluxo implementa um buffer inteligente com Redis via Upstash:

- **Buscar Histórico** — verifica se já existe alguma mensagem acumulada no buffer para aquele paciente
- **Atualizar Texto Acumulado** — adiciona a nova mensagem ao buffer do paciente
- **Ler Bloco Final** — lê o conteúdo completo acumulado no buffer
- **Wait** — aguarda um intervalo de tempo configurável para capturar todas as mensagens do bloco
- **Validar Última Mensagem** — garante que apenas a última mensagem do bloco dispara o processamento, evitando respostas duplicadas
- **Limpar Buffer** — limpa o buffer do Upstash após consolidar todas as mensagens
- **Agrupa Mensagens** — une todas as mensagens acumuladas em uma única string e envia para o Agente de IA

> 💡 O tempo de espera do node **Wait** é configurável conforme a necessidade do atendimento — ajuste o valor para equilibrar entre agilidade na resposta e captura completa das mensagens fragmentadas.

---

### Etapa 6 — Agente de IA

O coração do fluxo. O node **AI Agent** recebe a mensagem consolidada e processa a resposta com base no system prompt configurado e nas ferramentas disponíveis:

**Modelo:** OpenAI / GPT-5 mini

> O fluxo utiliza dois nodes de modelo — identificados como **OpenAI** e **OpenAI2** — ambos configurados com o mesmo modelo GPT-5 mini. A separação existe por necessidade arquitetural do N8N, onde o Agente de IA e o Structured Output Parser requerem conexões de modelo independentes.

**Tools disponíveis para o Agente:**

| Tool | Função |
|---|---|
| `historico_atendimento_pacientes` | Memória — recupera as interações anteriores do paciente para contexto |
| `Atualiza nome confirmado` | Atualiza o campo `nome_confirmado` na tabela `pacientes` quando o paciente informa seu nome |
| `agendamentos` | Aciona o MCP do Google Calendar para gerenciar consultas |

**Structured Output Parser:** estrutura a saída do modelo em formato padronizado para o próximo node.

O system prompt completo do agente está disponível em: [System Prompt do Agente](docs/system_prompt_agente.md)

---

### Etapa 7 — Gerenciamento de Agendamentos (MCP)

Quando o Agente de IA identifica uma intenção de agendamento na mensagem do paciente, aciona automaticamente o **MCP Agendamento** — um servidor MCP integrado ao Google Calendar com 4 tools disponíveis:

| Tool | Operação | Função | Prompt |
|---|---|---|---|
| `listar_eventos` | getAll | Lista consultas agendadas do paciente | [Ver prompt](docs/prompt_listar_eventos.md) |
| `criar_evento` | create | Agenda uma nova consulta | [Ver prompt](docs/prompt_criar_evento.md) |
| `reagendar_evento` | update | Altera data/hora de uma consulta existente | [Ver prompt](docs/prompt_reagendar_evento.md) |
| `deletar_evento` | delete | Cancela uma consulta agendada | [Ver prompt](docs/prompt_deletar_evento.md) |

Cada tool possui um prompt específico que orienta o modelo sobre como executar a operação corretamente no Google Calendar.

---

### Etapa 8 — Divisão de Mensagens

Para garantir uma experiência de atendimento mais natural e agradável, a resposta gerada pelo Agente de IA não é enviada como um único bloco de texto:

- **Divide Mensagens** — fragmenta a resposta em partes menores
- **Split Out** — separa cada parte em itens individuais
- **Loop Over Items** — itera sobre cada parte sequencialmente
- **Send message** — envia cada parte individualmente via WhatsApp API da Meta

O paciente recebe mensagens menores e mais naturais, simulando uma conversa humana fluida.

---

## 🧠 Cicatrizes de aprendizado

**1. Escolha do modelo de LLM**

Durante o desenvolvimento, testei três modelos: GPT-5 mini (OpenAI), Claude Sonnet 3.5 (Anthropic) e Tencent Hy3 (gratuito via OpenRouter). O Tencent Hy3 foi descartado por apresentar alucinações nas respostas, comprometendo a confiabilidade do atendimento. O Claude Sonnet 3.5 performou bem, porém o GPT-5 mini foi escolhido como modelo definitivo pela melhor relação entre custo, velocidade de resposta e qualidade nas interações de atendimento odontológico. O Groq foi utilizado exclusivamente para transcrição de áudio via Whisper.

**2. Buffer Redis para mensagens picadas**

Pacientes frequentemente enviam mensagens fragmentadas no WhatsApp — uma ideia dividida em 3 ou 4 mensagens consecutivas. Sem o buffer, o Agente de IA respondia cada fragmento separadamente, gerando respostas sem contexto e experiência ruim para o paciente. A solução com Redis via Upstash e o node Wait resolveu completamente este problema. O tempo de espera do node Wait e a quantidade de interações do histórico são configuráveis conforme a preferência e o perfil do atendimento.

**3. Criação automática da tabela de histórico**

Ao configurar o node de memória Postgres Chat Memory no N8N, descobri que a tabela `n8n_chat_histories` é criada automaticamente no Supabase na primeira execução — sem necessidade de configuração manual. O número de telefone do paciente é usado como Session ID, garantindo histórico isolado por paciente.

**4. Context Window Length configurável**

A quantidade de interações anteriores que o Agente de IA considera como contexto é definida pelo parâmetro Context Window Length — configurável conforme a necessidade do atendimento. Um valor maior garante mais contexto para conversas longas, mas aumenta o consumo de tokens. Ajuste este valor para equilibrar qualidade de resposta e custo operacional.

**5. Fallback para mensagens não suportadas**

Inicialmente o fluxo não tratava mensagens de tipos não suportados (imagens, documentos, stickers), deixando o paciente sem resposta. A adição do node **Mensagem inválida** no caminho Fallback do Switch resolveu a lacuna, orientando o paciente de forma educada sobre os tipos de mensagem aceitos.

---

## 🔮 Próximas evoluções

- Adicionar delay entre mensagens no Loop Over Items para evitar interpretação como spam pelo WhatsApp
- Implementar tratamento do Fallback com encaminhamento para atendente humano quando necessário
- Adicionar confirmação de agendamento por mensagem após criação do evento no Google Calendar
- Implementar relatório periódico de atendimentos via dashboard no Supabase

---

## 👨‍💻 Autor

Desenvolvido por **Marcelo Poliato de Oliveira** como projeto prático de automação inteligente com N8N e IA Generativa.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Marcelo%20Poliato-0077B5?logo=linkedin)](https://www.linkedin.com/in/marcelo-poliato)
[![GitHub](https://img.shields.io/badge/GitHub-poliato2015--max-181717?logo=github)](https://github.com/poliato2015-max)
