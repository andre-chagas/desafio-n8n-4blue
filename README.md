# Desafio Técnico – Automação de Matrículas (n8n)

Este repositório contém a solução do desafio técnico de automação de matrículas usando o n8n.

O objetivo é:

- Receber dados de novos alunos via Webhook
- Tratar nome, telefone e data de nascimento
- Decidir a tabela de destino no Postgres conforme o curso
- Enviar um e-mail de boas-vindas para o aluno
- Tratar o “desafio oculto” descrito no enunciado

## Visão geral do workflow

Fluxo principal:

1. Webhook Matrícula
2. Tratar Dados Aluno (Code em JavaScript)
3. Switch por curso (Tech, Business, Default)
4. Postgres – matriculas_tech e Postgres – matriculas_biz
5. Email – Boas-vindas Tech e Email – Boas-vindas Business
6. Rota Default para cursos inválidos (tratamento do desafio oculto)

### 1. Webhook Matrícula

- Método: POST
- Path: `/matricula-aluno`
- Campos esperados no JSON de entrada:

  ```json
  {
    "nome_completo": "john smith",
    "email": "john@4blue.com.br",
    "data_nascimento": "28/02/2020",
    "telefone": "(11) 99999-9999",
    "curso": "Tech"
  }
  ```

O Webhook está configurado para responder quando o último nó do fluxo terminar.

### 2. Tratar Dados Aluno (Code)

Node de código em JavaScript que:

- Normaliza o nome, deixando a primeira letra de cada palavra em maiúscula
- Remove caracteres não numéricos do telefone
- Converte a data de `DD/MM/YYYY` para `YYYY-MM-DD`
- Normaliza o valor do curso para `Tech` ou `Business`

Saída do node:

- `nome`
- `email`
- `data_nascimento`
- `telefone`
- `curso`

### 3. Switch por curso

Node Switch que avalia o campo `curso`:

- Regra 1: `curso === "Tech"` → segue para Postgres `matriculas_tech`
- Regra 2: `curso === "Business"` → segue para Postgres `matriculas_biz`
- Default: qualquer outro valor de curso → segue para rota de tratamento de erro

### 4. Postgres

Para cada curso existe um node Postgres configurado como inserção:

- `Postgres – matriculas_tech` insere os campos tratados na tabela `matriculas_tech`
- `Postgres – matriculas_biz` insere na tabela `matriculas_biz`

No ambiente atual, não existe um servidor Postgres acessível a partir do n8n, por isso os nodes retornam erro de conexão (ECONNREFUSED). Em um ambiente real, basta apontar a credencial para um Postgres válido (local, Supabase, RDS, etc.) e as inserções passam a funcionar sem alterações na lógica do workflow.

### 5. Emails de boas-vindas

Após o Postgres, o fluxo passa por nodes de e-mail (Gmail) configurados para enviar uma mensagem transacional ao aluno:

Corpo do email:

```text
Olá {{ $json["nome"] }}, sua matrícula no curso {{ $json["curso"] }} foi recebida com sucesso!
```

As credenciais Gmail utilizam OAuth2. No ambiente de teste, não há Client ID/Secret válidos configurados, por isso a autenticação OAuth não é concluída. A configuração do node e o uso de variáveis dinâmicas estão prontos para serem usados assim que uma credencial Gmail válida estiver disponível.

### 6. Tratamento do “desafio oculto”

O enunciado assume que sempre serão recebidos apenas os cursos `Tech` e `Business`. Isso é um risco, pois:

- Um valor inesperado de curso poderia ser inserido na tabela errada
- O aluno poderia receber e-mail de confirmação mesmo com dados inconsistentes

Para tratar esse ponto:

- O valor de `curso` é normalizado no node de código
- O Switch possui uma rota Default para valores que não sejam `Tech` ou `Business`
- A rota Default direciona para um node de tratamento de erro, que marca o item como inválido e impede que ele siga para Postgres ou e-mail


## Vídeo de explicação

- Link do vídeo: `https://www.youtube.com/watch?v=_AsrROVqLbA`

