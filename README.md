# Person API — Quipux

Aplicação completa para **cadastro de pessoas** com **previsão de nacionalidade a partir do nome**, usando a API pública [nationalize.io](https://nationalize.io).

O projeto é composto por uma API REST em Java/Quarkus, um banco PostgreSQL e uma interface web que funciona como um console para testar todas as operações. Tudo sobe com um único comando via Docker Compose.

---

## Sumário

- [Funcionalidades](#funcionalidades)
- [Tecnologias](#tecnologias)
- [Arquitetura](#arquitetura)
- [Como executar](#como-executar)
- [Usando a interface web](#usando-a-interface-web)
- [Referência da API](#referência-da-api)
- [Validações e regras de negócio](#validações-e-regras-de-negócio)
- [Tratamento de erros](#tratamento-de-erros)
- [Configuração](#configuração)
- [Estrutura do projeto](#estrutura-do-projeto)
- [Solução de problemas](#solução-de-problemas)
- [Decisões de projeto](#decisões-de-projeto)

---

## Funcionalidades

| Funcionalidade | Descrição |
|---|---|
| **Cadastrar pessoa** | Registra CPF, nome, sobrenome e e-mail, com validação completa dos dados. |
| **Listar pessoas** | Retorna os cadastros de forma paginada, com total de registros e de páginas. |
| **Buscar por CPF** | Retorna os dados de uma pessoa específica. |
| **Excluir pessoa** | Remove o cadastro correspondente ao CPF. |
| **Prever nacionalidade** | Consulta a nationalize.io com o primeiro nome da pessoa e retorna o país mais provável, já traduzido para o português (ex.: `Brasil`). |

Além disso, o projeto conta com:

- **Autenticação por chave de API** nas operações de escrita (cadastro e exclusão).
- **Validação de CPF real**, incluindo os dígitos verificadores.
- **Bloqueio de duplicidade** de CPF e de e-mail.
- **Respostas de erro padronizadas** em todos os endpoints.
- **Documentação interativa** com Swagger UI.
- **Interface web** para testar as operações sem precisar de Postman ou cURL.

---

## Tecnologias

| Camada | Tecnologia |
|---|---|
| Backend | Java 21, Quarkus 3, Quarkus REST (Jackson) |
| Persistência | Hibernate ORM com Panache, PostgreSQL 16 |
| Validação | Hibernate Validator (Bean Validation) |
| Integração externa | Quarkus REST Client (nationalize.io) |
| Documentação | SmallRye OpenAPI + Swagger UI |
| Frontend | HTML, CSS e JavaScript puro, servido por nginx |
| Infraestrutura | Docker e Docker Compose |

---

## Arquitetura

```
┌──────────────────┐      HTTP/JSON      ┌──────────────────┐      JDBC      ┌──────────────────┐
│  Frontend        │ ──────────────────▶ │  Backend         │ ─────────────▶ │  PostgreSQL      │
│  nginx :8080     │                     │  Quarkus :8081   │                │  :5432           │
└──────────────────┘                     └────────┬─────────┘                └──────────────────┘
                                                  │ HTTPS
                                                  ▼
                                         ┌──────────────────┐
                                         │  nationalize.io  │
                                         └──────────────────┘
```

O backend é organizado em camadas, e cada requisição percorre este caminho:

1. **Filtro de autenticação** (`AuthFilter`): verifica a chave de API nas rotas protegidas.
2. **Resource** (`PersonResource`): recebe a requisição e valida os parâmetros.
3. **Service** (`PersonService`): aplica as regras de negócio.
4. **Repository** (`PersonRepository`): acessa o banco de dados.
5. **Client** (`NationalizeClient`): consulta a API externa, quando necessário.
6. **Exception mappers**: convertem qualquer falha no formato de erro padrão.

---

## Como executar

### Pré-requisito

Apenas o **Docker Desktop** ([download](https://www.docker.com/products/docker-desktop)). Java, Maven e PostgreSQL não precisam estar instalados na máquina, pois tudo é baixado dentro dos containers.

Para confirmar a instalação:

```bash
docker --version
docker compose version
```

> **macOS:** se aparecer `command not found: docker`, abra o Docker Desktop pelo menos uma vez e abra um terminal novo. Se o erro continuar, adicione o caminho manualmente:
>
> ```bash
> echo 'export PATH="$PATH:/Applications/Docker.app/Contents/Resources/bin"' >> ~/.zshrc
> source ~/.zshrc
> ```

### Subindo o projeto

Com o Docker Desktop aberto, na raiz do projeto:

```bash
cp .env.example .env        # apenas se o arquivo .env ainda não existir
docker compose up --build
```

Esse comando sobe os três serviços, na ordem: banco, backend (que aguarda o banco ficar pronto) e frontend.

Na primeira execução, o Maven baixa as dependências dentro do container, o que leva alguns minutos. As próximas execuções reaproveitam o cache. O backend está pronto quando aparecer nos logs:

```
quipux-backend  | ... Listening on: http://0.0.0.0:8081
```

### Endereços

| Serviço | Endereço |
|---|---|
| Interface web | http://localhost:8080 |
| API | http://localhost:8081 |
| Swagger UI | http://localhost:8081/q/swagger-ui |
| PostgreSQL | localhost:5432 |

### Comandos úteis

```bash
docker compose ps            # verifica se os serviços estão rodando
docker compose logs -f backend   # acompanha os logs do backend
docker compose down          # para os serviços
docker compose down -v       # para os serviços e apaga os dados do banco
```

### Modo de desenvolvimento (opcional)

Para editar o backend com recarregamento automático, suba apenas o banco no Docker e rode a API na máquina. Este modo exige o **JDK 21** instalado.

```bash
docker compose up -d postgres
cd backend
./mvnw quarkus:dev        # Linux e macOS
mvnw.cmd quarkus:dev      # Windows
```

Não é preciso instalar o Maven, pois o `mvnw` baixa a versão correta automaticamente. No macOS, se aparecer `permission denied`, rode `chmod +x mvnw` antes.

**No IntelliJ:** clique com o botão direito em `backend/pom.xml` e escolha **Add as Maven Project** para que as dependências sejam baixadas e o código seja reconhecido. Depois, rode o objetivo `quarkus:dev` pela aba **Maven**. Para abrir o frontend, basta abrir `frontend/index.html` no navegador.

---

## Usando a interface web

Acesse http://localhost:8080. A tela tem três áreas:

- **Barra superior:** campo da chave de API, já preenchido com o valor padrão (`quipux-secret-key`).
- **Menu lateral:** as cinco operações (Cadastrar, Listar, Buscar, Nacionalidade e Excluir), cada uma identificada pelo verbo HTTP.
- **Painel principal:** formulário da operação e a resposta da API, com o código de status e os dados retornados.

### Roteiro rápido de teste

1. Em **Cadastrar**, informe o CPF `11144477735`, nome `Maria`, sobrenome `Souza` e um e-mail.
2. Em **Listar**, veja o cadastro na tabela, com navegação entre páginas.
3. Em **Nacionalidade**, informe o mesmo CPF para ver o país mais provável para o nome.
4. Em **Excluir**, remova o cadastro.

Tente também cadastrar o mesmo CPF duas vezes, usar um CPF inválido ou apagar a chave de API para ver as mensagens de erro.

---

## Referência da API

| Método | Rota | Autenticação | Descrição |
|---|---|---|---|
| `POST` | `/registrarName` | Sim | Cadastra uma pessoa |
| `GET` | `/list` | Não | Lista as pessoas, com paginação |
| `GET` | `/list/{cpf}` | Não | Busca uma pessoa pelo CPF |
| `DELETE` | `/list/{cpf}` | Sim | Exclui uma pessoa |
| `GET` | `/findNacionalityByPerson/{cpf}` | Não | Retorna o país mais provável |

### Autenticação

As operações de escrita exigem o cabeçalho:

```
X-API-Key: quipux-secret-key
```

O valor é definido pela variável `API_KEY` no arquivo `.env`. No Swagger UI, use o botão **Authorize** no topo da página.

### Cadastrar pessoa

```bash
curl -X POST http://localhost:8081/registrarName \
  -H "Content-Type: application/json" \
  -H "X-API-Key: quipux-secret-key" \
  -d '{
    "cpf": "11144477735",
    "firstName": "Maria",
    "lastName": "Souza",
    "email": "maria.souza@exemplo.com"
  }'
```

Resposta `201 Created`:

```json
{
  "id": 1,
  "cpf": "11144477735",
  "firstName": "Maria",
  "lastName": "Souza",
  "email": "maria.souza@exemplo.com"
}
```

### Listar pessoas

Parâmetros opcionais: `page` (começa em 1, padrão `1`) e `size` (entre 1 e 100, padrão `50`).

```bash
curl "http://localhost:8081/list?page=1&size=20"
```

Resposta `200 OK`:

```json
{
  "data": [
    { "id": 1, "cpf": "11144477735", "firstName": "Maria", "lastName": "Souza", "email": "maria.souza@exemplo.com" }
  ],
  "page": 1,
  "size": 20,
  "totalElements": 1,
  "totalPages": 1,
  "hasNext": false
}
```

### Buscar por CPF

```bash
curl http://localhost:8081/list/11144477735
```

Retorna `200 OK` com os dados da pessoa, ou `404 Not Found` se o CPF não estiver cadastrado.

### Excluir pessoa

```bash
curl -X DELETE http://localhost:8081/list/11144477735 \
  -H "X-API-Key: quipux-secret-key"
```

Retorna `204 No Content` em caso de sucesso.

### Prever nacionalidade

```bash
curl http://localhost:8081/findNacionalityByPerson/11144477735
```

Resposta `200 OK`:

```json
{ "nationality": "Brasil" }
```

O backend busca a pessoa pelo CPF, envia o **primeiro nome** para a nationalize.io, escolhe o país com maior probabilidade e converte o código (ex.: `BR`) para o nome em português.

---

## Validações e regras de negócio

| Campo | Regras |
|---|---|
| `cpf` | Obrigatório, exatamente 11 dígitos numéricos (sem pontos ou traços) e dígitos verificadores válidos. Sequências como `12345678901` são rejeitadas. |
| `firstName` | Obrigatório, entre 2 e 100 caracteres. |
| `lastName` | Obrigatório, entre 2 e 100 caracteres. |
| `email` | Obrigatório, formato válido, até 150 caracteres. |

Regras adicionais:

- **CPF e e-mail são únicos.** Tentar cadastrar um valor já existente retorna `409 Conflict`.
- **E-mails são normalizados** para minúsculas antes de gravar, então `Maria@Exemplo.com` e `maria@exemplo.com` são considerados o mesmo endereço.
- **Espaços extras** no início e no fim dos campos são removidos.

Para testes, use um CPF válido como `11144477735`. Geradores de CPF online também podem ser usados para criar outros valores válidos.

---

## Tratamento de erros

Todas as respostas de erro seguem a mesma estrutura. O campo `errors` detalha falhas por campo e vem vazio quando o erro não é de validação.

```json
{
  "message": "Erro de validação nos dados enviados.",
  "errors": [
    { "field": "email", "message": "O e-mail está em um formato inválido." }
  ]
}
```

| Status | Situação |
|---|---|
| `400` | Dados inválidos, parâmetros incorretos ou JSON malformado |
| `401` | Chave de API ausente ou inválida |
| `404` | Pessoa não encontrada, ou nacionalidade sem previsão para o nome |
| `409` | CPF ou e-mail já cadastrado |
| `503` | API externa de nacionalidade indisponível |
| `500` | Erro interno inesperado (registrado nos logs do backend) |

---

## Configuração

As variáveis ficam no arquivo `.env`, na raiz do projeto:

| Variável | Padrão | Descrição |
|---|---|---|
| `DB_USER` | `postgres` | Usuário do banco |
| `DB_PASSWORD` | `postgres` | Senha do banco |
| `DB_NAME` | `personsdb` | Nome do banco |
| `DB_PORT` | `5432` | Porta do PostgreSQL na máquina |
| `BACKEND_PORT` | `8081` | Porta da API na máquina |
| `FRONTEND_PORT` | `8080` | Porta da interface web na máquina |
| `API_KEY` | `quipux-secret-key` | Chave exigida nas operações de escrita |

> Se alterar `BACKEND_PORT`, atualize também o `baseUrl` em `frontend/js/config.js`, que é o endereço usado pela interface para chamar a API.

---

## Estrutura do projeto

```
.
├── docker-compose.yml          orquestração dos três serviços
├── .env.example                modelo de variáveis de ambiente
├── backend/
│   ├── Dockerfile              build em duas etapas (Maven → JRE)
│   ├── pom.xml                 dependências do projeto
│   └── src/main/
│       ├── resources/
│       │   └── application.properties   configuração do Quarkus
│       └── java/org/quipux/
│           ├── clients/        consumo da API de nacionalidade
│           ├── config/         configuração do OpenAPI/Swagger
│           ├── dtos/           objetos de entrada e saída da API
│           ├── entities/       mapeamento JPA da tabela persons
│           ├── exceptions/     tratamento padronizado de erros
│           ├── filters/        autenticação por chave de API
│           ├── repositories/   acesso a dados
│           ├── resources/      endpoints REST
│           └── services/       regras de negócio
└── frontend/
    ├── Dockerfile              imagem nginx
    ├── nginx.conf              configuração do servidor
    ├── index.html              estrutura da página
    ├── css/styles.css          estilos
    └── js/
        ├── config.js           endereço da API e parâmetros
        ├── operations.js       definição das operações e formulários
        ├── api.js              chamadas HTTP para o backend
        ├── render.js           renderização das respostas
        └── app.js              controle da interface
```

---

## Solução de problemas

| Problema | Solução |
|---|---|
| `command not found: docker` | Abra o Docker Desktop e um terminal novo. No macOS, veja a dica em [Pré-requisito](#pré-requisito). |
| `Cannot connect to the Docker daemon` | O Docker Desktop não está aberto. Abra-o e aguarde o ícone da baleia ficar estável. |
| `port is already allocated` | Outra aplicação usa a porta. Altere `BACKEND_PORT`, `FRONTEND_PORT` ou `DB_PORT` no `.env`. |
| Interface mostra "sem conexão" | O backend ainda está iniciando ou parou. Verifique com `docker compose logs backend`. |
| Nacionalidade retorna `503` | A nationalize.io está fora do ar ou o limite gratuito de consultas foi atingido. Tente novamente mais tarde. |
| Quero começar com o banco limpo | Rode `docker compose down -v` e suba novamente. |

---

## Decisões de projeto

**Camadas separadas.** Os endpoints recebem `PersonRequest` e devolvem `PersonResponse`, nunca a entidade JPA. Isso evita expor o identificador interno como campo de entrada e desacopla o contrato da API do modelo de persistência.

**Erros centralizados.** O tratamento fica nos `ExceptionMapper`, não espalhado pelos endpoints, garantindo que qualquer falha saia no mesmo formato.

**Autenticação por anotação.** A anotação `@Secured` marca as rotas protegidas, e o `AuthFilter` só atua sobre elas. Assim, proteger um novo endpoint exige apenas adicionar a anotação.

**Falha controlada na integração externa.** Se a nationalize.io estiver indisponível, a API responde `503` com uma mensagem clara, em vez de um erro genérico.

**Schema gerado pelo Hibernate.** Em um cenário de produção, o schema seria versionado com Flyway. Aqui, a geração automática foi mantida pela simplicidade.

**Rotas conforme o enunciado.** Os caminhos `/registrarName` e `/findNacionalityByPerson` seguem exatamente a especificação recebida, incluindo a grafia original.