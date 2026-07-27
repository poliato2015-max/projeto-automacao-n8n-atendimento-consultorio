# Configurando a Credencial PostgreSQL do Supabase no n8n

## Objetivo

Este projeto já possui o **Node PostgreSQL** configurado no workflow do n8n.

Após importar o arquivo JSON, **não é necessário criar um novo node PostgreSQL**. Basta abrir a credencial existente e substituir os parâmetros de conexão pelos dados do seu próprio projeto Supabase.

---

## Pré-requisitos

Antes de iniciar, certifique-se de que você possui:

- Um projeto criado no Supabase;
- O workflow deste projeto importado no n8n;
- Permissão para acessar o painel do Supabase.

---

# Passo 1 – Acesse as informações de conexão do projeto

No painel principal do seu projeto Supabase, clique em **Connect**.

Em seguida, clique em **Get Connected**.

![Passo 1 - Connect](https://raw.githubusercontent.com/poliato2015-max/imagens/main/projeto_automacao_n8n_atendimento_consultorio/projeto-automacao-n8n-atendimento-consultorio-projeto-supabase-connect.png)

---

# Passo 2 – Selecione o tipo de conexão

Na janela **Connect to your project**, configure as opções abaixo:

- **Direct connection string**
- **Transaction pooler**

Essas opções disponibilizam os parâmetros compatíveis com a credencial PostgreSQL utilizada neste projeto.

![Passo 2 - Tipo de conexão](https://raw.githubusercontent.com/poliato2015-max/imagens/main/projeto_automacao_n8n_atendimento_consultorio/projeto-automacao-n8n-atendimento-consultorio-projeto-supabase-transaction.png)

---

# Passo 3 – Localize os parâmetros da conexão

Role a página até localizar a seção **Connection parameters**.

Nesta seção estão disponíveis os dados que deverão ser copiados para a credencial PostgreSQL do n8n.

![Passo 3 - Connection Parameters](https://raw.githubusercontent.com/poliato2015-max/imagens/main/projeto_automacao_n8n_atendimento_consultorio/projeto-automacao-n8n-atendimento-consultorio-projeto-supabase-parameters.png)

Os números destacados correspondem aos campos abaixo:

| Nº | Campo no Supabase | Campo no n8n |
|:--:|-------------------|--------------|
| **1** | Host | Host |
| **2** | Database | Database |
| **3** | User | User |
| **4** | Password | Password |
| **8** | Port | Port |

> **Importante**
>
> O campo **Password** corresponde à senha do banco de dados do seu projeto.
>
> Caso você não conheça essa senha, utilize a opção **Reset database password** disponível no Supabase para definir uma nova senha.

---

# Passo 4 – Atualize a credencial PostgreSQL no n8n

Abra o workflow importado.

Em seguida:

1. Abra o **Node PostgreSQL**.
2. Clique na credencial **Postgres account**.
3. Substitua apenas os campos indicados abaixo.

![Passo 4 - Node PostgreSQL](https://raw.githubusercontent.com/poliato2015-max/imagens/main/projeto_automacao_n8n_atendimento_consultorio/projeto-automacao-n8n-atendimento-consultorio-projeto-supabase-node-postgres.png)

| Nº | Campo | Ação |
|:--:|--------|------|
| **1** | Host | Substituir pelo Host do Supabase |
| **2** | Database | Substituir pelo Database do Supabase |
| **3** | User | Substituir pelo User do Supabase |
| **4** | Password | Informar a senha do banco |
| **8** | Port | Informar a porta do Supabase |

---

# Configurações que NÃO devem ser alteradas

Os demais parâmetros da credencial já estão configurados corretamente para este projeto.

Mantenha as seguintes configurações exatamente como estão:

| Nº | Configuração | Valor esperado |
|:--:|--------------|----------------|
| **5** | Maximum Number of Connections | 100 |
| **6** | Ignore SSL Issues | Off |
| **7** | SSL | Disable |
| **9** | SSH Tunnel | Off |

---

# Testando a conexão

Após preencher os parâmetros:

1. Salve a credencial.
2. Clique em **Retry** ou execute qualquer node PostgreSQL do workflow.

Se tudo estiver configurado corretamente, será exibida a mensagem:

```text
Connection tested successfully
```

---

# Resumo da configuração

| Supabase | → | Node PostgreSQL |
|-----------|---|-----------------|
| Host | → | Host |
| Database | → | Database |
| User | → | User |
| Password | → | Password |
| Port | → | Port |

---

## Conclusão

Após concluir essa configuração, todos os nodes PostgreSQL presentes no workflow utilizarão automaticamente essa credencial para acessar o banco de dados do seu projeto Supabase.

Não será necessário realizar essa configuração novamente, a menos que a senha ou os parâmetros de conexão do seu projeto sejam alterados.

---

## Documentação oficial

Caso deseje obter mais informações sobre conexões PostgreSQL e Supabase, consulte:

- https://supabase.com/docs
- https://docs.n8n.io
