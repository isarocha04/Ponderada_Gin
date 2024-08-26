# Ponderada Gin Isabela semana 3

O objetivo deste projeto é demonstrar a aplicação de testes unitários eficazes em um projeto Go utilizando o framework Gin, o ORM GORM, e o banco de dados PostgreSQL.

## Estrutura do Repositório

O projeto está organizado nas seguintes pastas:

- **`/handlers/`**: Contém os handlers de requisição HTTP para a aplicação.
  - `handler.go`: Define as funções que lidam com as requisições HTTP.
  - `handler_test.go`: Testes unitários para os handlers.

- **`/db/`**: Gerencia as conexões e consultas ao banco de dados.
  - `database.go`: Configura a conexão com o banco de dados PostgreSQL.
  - `database_test.go`: Testes unitários para as operações de banco de dados.

- **`/tests/`**: Contém testes mais gerais que não se enquadram diretamente nos handlers ou na camada de banco de dados.
  - `main_test.go`: Testes que verificam a integração e funcionamento geral da aplicação.

- **`main.go`**: Ponto de entrada da aplicação que inicializa o servidor Gin e conecta ao banco de dados.

- **`go.mod` e `go.sum`**: Gerenciam as dependências do projeto Go.

## Implementação das Funções com Comentários Explicativos

Nesta seção, você encontrará as implementações das funções principais do projeto, acompanhadas de comentários que explicam as técnicas e conceitos de Test-Driven Development (TDD) aplicados.

### `handler.go`

```go
package handlers

import (
    "net/http"
    "github.com/gin-gonic/gin"
    "gorm.io/gorm"
)

// Handler estrutura que contém as dependências necessárias para as operações
type Handler struct {
    DB *gorm.DB  // Injeção do banco de dados na estrutura Handler
}

// CreateUser lida com a requisição de criação de um novo usuário
func (h *Handler) CreateUser(c *gin.Context) {
    var user User  // Estrutura de dados que representa um usuário

    // BindJSON decodifica a requisição JSON para a estrutura de dados
    if err := c.ShouldBindJSON(&user); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Tentativa de criar um novo registro de usuário no banco de dados
    if err := h.DB.Create(&user).Error; err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    // Retorna um status de sucesso com os dados do usuário criado
    c.JSON(http.StatusCreated, user)
}
```

A função CreateUser demonstra a aplicação de TDD ao testar primeiramente o comportamento esperado da função antes de implementá-la. Os testes garantem que o handler lida corretamente com erros de validação e criação no banco de dados.

### handler_test.go

```go
package handlers

import (
    "net/http"
    "net/http/httptest"
    "testing"
    "github.com/gin-gonic/gin"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

// MockDB é uma estrutura que simula o comportamento do GORM para testes
type MockDB struct {
    mock.Mock
}

// Create simula a função de criação do GORM
func (m *MockDB) Create(value interface{}) error {
    args := m.Called(value)
    return args.Error(0)
}

// TestCreateUser verifica se o handler CreateUser funciona como esperado
func TestCreateUser(t *testing.T) {
    gin.SetMode(gin.TestMode)

    // Criação do mock do banco de dados
    mockDB := new(MockDB)
    handler := &Handler{DB: mockDB}

    // Simula uma resposta bem-sucedida na criação do usuário
    mockDB.On("Create", mock.AnythingOfType("*handlers.User")).Return(nil)

    // Criação de uma requisição HTTP de teste
    req, _ := http.NewRequest("POST", "/users", nil)
    resp := httptest.NewRecorder()

    // Execução do handler com a requisição de teste
    handler.CreateUser(resp, req)

    // Verificação do resultado esperado
    assert.Equal(t, http.StatusCreated, resp.Code)
    mockDB.AssertExpectations(t)
}
```


Este teste unitário utiliza o mock do banco de dados para isolar o código do handler e focar na lógica de processamento da requisição. Isso é um exemplo claro da aplicação de TDD, onde o teste é escrito para especificar o comportamento antes da implementação.

### database.go

```go
package db

import (
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

// InitDB inicializa a conexão com o banco de dados PostgreSQL
func InitDB(dsn string) (*gorm.DB, error) {
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        return nil, err
    }
    return db, nil
}
```

A função InitDB é responsável por inicializar a conexão com o banco de dados PostgreSQL. Em um contexto de TDD, você escreveria testes para garantir que a conexão é estabelecida corretamente ou que os erros são tratados de forma adequada.

### database_test.go

```go
package db

import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/suite"
    "gorm.io/gorm"
)

// DatabaseTestSuite define uma suíte de testes para o banco de dados
type DatabaseTestSuite struct {
    suite.Suite
    DB *gorm.DB
}

// SetupSuite configura a suíte de testes, criando a conexão com o banco de dados
func (suite *DatabaseTestSuite) SetupSuite() {
    dsn := "user=test password=test dbname=testdb"
    db, err := InitDB(dsn)
    suite.Require().NoError(err, "Erro ao conectar ao banco de dados")
    suite.DB = db
}

// TearDownSuite encerra a conexão com o banco de dados após os testes
func (suite *DatabaseTestSuite) TearDownSuite() {
    sqlDB, _ := suite.DB.DB()
    sqlDB.Close()
}

// TestSuite executa a suíte de testes
func TestSuite(t *testing.T) {
    suite.Run(t, new(DatabaseTestSuite))
}
```

Esta suíte de testes mostra como configurar e limpar a conexão com o banco de dados durante os testes. Isso é crucial em TDD para garantir que o ambiente de teste seja isolado e reproduzível.
