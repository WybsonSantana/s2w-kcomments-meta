# s2w-kcomments-meta

# Design Doc: Sistema de Moderação de Comentários com Microsserviços

## 1. Visão Geral

Este documento de design detalha a arquitetura da aplicação KComments, um sistema de recebimento, moderação e consulta de comentários composto por dois microsserviços:

- CommentService
- ModerationService

O sistema recebe comentários enviados por usuários, solicita a moderação do conteúdo e armazena somente os comentários aprovados. A comunicação entre os serviços ocorre de forma síncrona, por HTTP/REST, utilizando `RestClient` no `CommentService`.

A arquitetura separa a responsabilidade de gerenciamento dos comentários da responsabilidade de validar o conteúdo, mantendo cada serviço com um escopo bem definido.

## 2. Objetivos

- **Separação de Responsabilidades**: Manter a criação e consulta de comentários separadas da validação de conteúdo.
- **Integridade dos Dados**: Armazenar somente comentários aprovados pela moderação.
- **Comunicação Síncrona**: Validar cada novo comentário por meio de uma chamada HTTP/REST ao `ModerationService`.
- **Tratamento de Falhas**: Tratar erros gerais e timeouts na integração, retornando os códigos HTTP adequados.
- **Consulta Eficiente**: Permitir a consulta individual e a listagem paginada de comentários aprovados.
- **Manutenibilidade**: Facilitar a manutenção e evolução dos serviços por meio de responsabilidades específicas.

## 3. Descrição dos Serviços

### 3.1 Capacidades de Negócio Mapeadas

Foram identificadas duas capacidades principais para a aplicação KComments:

1. **Gerenciamento de Comentários**
   - Recebe novos comentários, solicita sua moderação e armazena somente os aprovados.
   - Permite consultar um comentário aprovado por identificador e listar comentários aprovados com paginação.
   - Serviço relacionado: `CommentService`.

2. **Moderação de Conteúdo**
   - Verifica se o texto de um comentário contém palavras proibidas.
   - Retorna o resultado da moderação e, quando aplicável, o motivo da rejeição.
   - Serviço relacionado: `ModerationService`.

A separação por capacidade de negócio mantém o fluxo de gerenciamento dos comentários independente das regras de moderação. Neste desafio, a comunicação é síncrona: o `CommentService` aguarda a resposta do `ModerationService` antes de decidir se deve persistir o comentário e qual resposta HTTP deve retornar ao cliente.

### 3.2 Serviço de CommentService

**Descrição**: Responsável pela criação, consulta e persistência dos comentários aprovados. Ao receber um novo comentário, o serviço gera um identificador UUID e chama o `ModerationService` por meio de uma requisição HTTP `POST`. O comentário só é armazenado se a moderação o aprovar.

**Funcionalidades Principais**:
- Receber e validar solicitações de criação de comentários.
- Enviar comentários para moderação de forma síncrona usando `RestClient`.
- Persistir apenas comentários aprovados.
- Consultar um comentário aprovado por ID.
- Listar comentários aprovados com paginação.
- Tratar falhas e timeouts na comunicação com o `ModerationService`.

**Endpoints**:
- `POST /api/comments`: Cria um comentário e solicita sua moderação.
- `GET /api/comments/{id}`: Consulta um comentário aprovado pelo ID.
- `GET /api/comments`: Lista comentários aprovados de forma paginada, recebendo os parâmetros `page` e `size`.

**Controller**: `CommentController`

**Comportamento dos endpoints**:

`POST /api/comments`
- Recebe um objeto `CommentInput`.
- Se o comentário for aprovado, persiste o comentário e retorna `201 Created` com um objeto `CommentOutput`.
- Se o comentário for rejeitado, não o armazena e retorna `422 Unprocessable Entity` com o motivo da rejeição.
- Se ocorrer um erro geral na integração com o `ModerationService`, retorna `502 Bad Gateway`.
- Se a chamada ao `ModerationService` exceder o timeout configurado, retorna `504 Gateway Timeout`.

`GET /api/comments/{id}`
- Retorna `200 OK` com `CommentOutput` se o comentário existir.
- Retorna `404 Not Found` se o comentário não existir.
- O identificador do comentário deve ser um UUID.

`GET /api/comments`
- Retorna uma lista paginada de comentários aprovados.
- Recebe os parâmetros `page` e `size`.
- A resposta deve conter os campos `page`, `size`, `totalElements`, `totalPages` e `content`.

**DTOs**:

`CommentInput` — modelo para criação de comentário:
- `text`: String com o conteúdo do comentário.
- `author`: String com o autor do comentário.

Exemplo de representação JSON:

```json
{
  "text": "string",
  "author": "string"
}
```

`CommentOutput` — modelo de exibição de comentário aprovado:
- `id`: UUID do comentário.
- `text`: String com o conteúdo do comentário.
- `author`: String com o autor do comentário.
- `createdAt`: data e hora de criação, representada em formato ISO 8601.

Exemplo de representação JSON:

```json
{
  "id": "string",
  "text": "string",
  "author": "string",
  "createdAt": "2023-11-15T10:00:00Z"
}
```

**Formato da resposta paginada**:

```json
{
  "page": 0,
  "size": 10,
  "totalElements": 45,
  "totalPages": 5,
  "content": [
    {
      "id": "string",
      "text": "string",
      "author": "string",
      "createdAt": "2023-11-15T10:00:00Z"
    }
  ]
}
```

**Persistência**:
- Banco de dados: H2.
- Somente comentários aprovados devem ser persistidos.
- Comentários rejeitados não devem ser armazenados.

**Tecnologias Utilizadas**:
- Linguagem de programação: Kotlin.
- Framework: Spring.
- Comunicação HTTP síncrona: `RestClient`.
- Banco de dados: H2.

### 3.3 Serviço de ModerationService

**Descrição**: Responsável por avaliar o conteúdo dos comentários e determinar se podem ser aprovados. O serviço mantém em memória uma lista fixa de palavras proibidas e verifica se o texto recebido contém algum dos termos dessa lista.

**Funcionalidades Principais**:
- Receber solicitações de moderação.
- Verificar o texto em busca de palavras proibidas.
- Retornar o resultado da avaliação e o motivo quando aplicável.

**Endpoint**:
- `POST /api/moderate`: Avalia o conteúdo de um comentário.

**Controller**: `ModerationController`

**Comportamento do endpoint**:

`POST /api/moderate`
- Recebe um objeto `ModerationInput`.
- Verifica se o texto contém palavras proibidas.
- Retorna `200 OK` com `ModerationOutput`, tanto para comentários aprovados quanto para comentários reprovados.

**DTOs**:

`ModerationInput` — modelo para solicitação de moderação:
- `commentId`: UUID do comentário que está sendo moderado.
- `text`: String com o conteúdo a ser avaliado.

Exemplo de representação JSON:

```json
{
  "commentId": "string",
  "text": "string"
}
```

`ModerationOutput` — modelo de resultado da moderação:
- `commentId`: UUID do comentário moderado.
- `approved`: Boolean que indica se o comentário foi aprovado.
- `reason`: String com o motivo da decisão.

Exemplo de representação JSON:

```json
{
  "commentId": "string",
  "approved": true,
  "reason": "string"
}
```

**Regras de Validação**:
- Lista fixa de palavras proibidas mantida em memória: `["ódio", "xingamento"]`.
- Se o texto contiver uma das palavras proibidas, o comentário deve ser reprovado.
- O serviço retorna o resultado da moderação em `ModerationOutput`.

**Tecnologias Utilizadas**:
- Linguagem de programação: Kotlin.
- Framework: Spring.
- Lista de palavras proibidas mantida em memória.

## 4. Comunicação entre Microsserviços

A comunicação entre os microsserviços ocorre de forma síncrona, utilizando HTTP/REST.

- **Síncrona**: O `CommentService` envia uma requisição `POST` para o endpoint `/api/moderate` do `ModerationService`, utilizando `RestClient`.
  - O corpo da requisição contém o texto do comentário e seu `commentId`, representado por um UUID.
  - O `CommentService` aguarda a resposta da moderação antes de decidir se persiste o comentário.
  - A resposta contém `approved` e `reason`, indicando a decisão e seu motivo.

**Configuração de timeout**:
- Configurar read timeout de 5 segundos para as chamadas ao `ModerationService`.
- Se a chamada exceder o tempo limite, o `CommentService` deve retornar `504 Gateway Timeout`.
- Se ocorrer outro erro de integração, o `CommentService` deve retornar `502 Bad Gateway`.

**Fluxo de criação de comentário**:

1. O cliente envia `POST /api/comments` ao `CommentService`.
2. O `CommentService` recebe `CommentInput` e gera um UUID para o comentário.
3. O `CommentService` envia `POST /api/moderate` ao `ModerationService`, com `text` e `commentId`.
4. O `ModerationService` verifica o texto contra a lista fixa de palavras proibidas.
5. O `ModerationService` retorna `ModerationOutput`.
6. Se aprovado, o `CommentService` persiste o comentário no H2 e retorna `201 Created` com `CommentOutput`.
7. Se reprovado, o `CommentService` não persiste o comentário e retorna `422 Unprocessable Entity` com o motivo da rejeição.

## 5. Problema e Solução

### 5.1 Problema Identificado

Um sistema que recebe comentários precisa impedir que conteúdos contendo palavras proibidas sejam armazenados como comentários aprovados. Se a validação estiver misturada às operações de persistência e consulta, as responsabilidades ficam acopladas e as regras de moderação podem ser mais difíceis de manter.

Também é necessário tratar falhas de comunicação entre os serviços de forma explícita, para que o cliente receba uma resposta HTTP adequada quando a moderação rejeitar um comentário ou quando a integração não puder ser concluída.

### 5.2 Solução Proposta

A solução separa o gerenciamento dos comentários da validação de conteúdo, delegando cada responsabilidade a um microsserviço.

#### 5.2.1 Separação entre Gerenciamento e Moderação

- **Solução**: O `CommentService` gerencia criação, persistência e consulta. O `ModerationService` avalia o texto e retorna a decisão de moderação.
- **Benefício**: As responsabilidades ficam delimitadas, facilitando a manutenção e a evolução das regras de moderação.

#### 5.2.2 Persistência Apenas de Comentários Aprovados

- **Solução**: O `CommentService` persiste o comentário somente depois de receber uma resposta de aprovação do `ModerationService`.
- **Benefício**: Comentários rejeitados não são armazenados e não aparecem nas consultas de comentários aprovados.

#### 5.2.3 Tratamento de Falhas na Integração

- **Solução**: O `RestClient` deve ser configurado com read timeout de 5 segundos. Erros gerais de integração são tratados como `502 Bad Gateway`, enquanto timeouts são tratados como `504 Gateway Timeout`.
- **Benefício**: O cliente recebe códigos HTTP que diferenciam falhas gerais de comunicação de situações em que a resposta demorou além do limite configurado.

#### 5.2.4 Consulta e Paginação

- **Solução**: O `CommentService` disponibiliza consulta individual por UUID e listagem paginada com os parâmetros `page` e `size`.
- **Benefício**: Os clientes podem consultar comentários aprovados individualmente ou percorrer conjuntos maiores de resultados por páginas.

### 5.3 Benefícios Gerais da Solução

- **Separação de Responsabilidades**: Cada microsserviço possui um objetivo específico.
- **Consistência das Regras de Moderação**: A avaliação do conteúdo fica centralizada no `ModerationService`.
- **Integridade da Persistência**: Apenas comentários aprovados são armazenados.
- **Tratamento Explícito de Falhas**: Os códigos `422`, `502` e `504` distinguem rejeição de conteúdo e problemas na integração.
- **Consulta Organizada**: A paginação permite listar comentários aprovados sem exigir o retorno de todos os registros em uma única resposta.
- **Testabilidade**: Os cenários de aprovação, rejeição, timeout e comentário inexistente podem ser testados separadamente.

### 5.4 Desenvolvimento dos Microsserviços

1. **Implementação do ModerationService**
   - Criar o endpoint `POST /api/moderate`.
   - Definir a lista fixa de palavras proibidas em memória.
   - Implementar a avaliação do texto e a resposta `ModerationOutput`.

2. **Implementação do CommentService**
   - Criar os endpoints de criação e consulta de comentários.
   - Definir os DTOs `CommentInput` e `CommentOutput`.
   - Configurar a persistência H2 para comentários aprovados.
   - Implementar a consulta individual e a listagem paginada.

3. **Configuração da integração**
   - Configurar o `RestClient` para realizar a chamada síncrona ao `ModerationService`.
   - Definir read timeout de 5 segundos.
   - Tratar erros gerais de integração e timeouts com os códigos HTTP esperados.

4. **Testes dos cenários principais**
   - Comentário válido e aprovado.
   - Comentário contendo palavras proibidas.
   - Timeout durante a chamada de moderação.
   - Consulta de comentário inexistente.
   - Listagem paginada de comentários aprovados.

5. **Logs**
   - Registrar logs nas operações importantes para facilitar o diagnóstico e a investigação de falhas.

## 6. Conclusão

A arquitetura proposta para o KComments separa o gerenciamento de comentários da moderação de conteúdo em dois microsserviços que se comunicam de forma síncrona por HTTP/REST. O `CommentService` coordena a criação, a consulta e a persistência dos comentários aprovados, enquanto o `ModerationService` aplica a lista fixa de palavras proibidas.

Com persistência em H2, paginação, timeout de 5 segundos e tratamento específico para rejeições, erros de integração e timeouts, o sistema atende aos requisitos definidos para o desafio e oferece uma base organizada para a evolução das funcionalidades.
