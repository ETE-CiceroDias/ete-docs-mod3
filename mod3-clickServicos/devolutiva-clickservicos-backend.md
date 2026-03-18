# Diagnóstico de Backend — ClickServiços
**Caio Amaral · Flávia Regina · Larissa Paiva · Magali França · Petyson Cauan · Thalison Felipe**
**DE_236 — Programação em Novas Tecnologias | Módulo 3**

🔗 GitHub: https://github.com/DS-Noite/marketplace-main-teste2

---

## Visão Geral

O ClickServiços tem o backend com mais entidades da turma — 8 controllers, 10 modelos com relacionamentos bem definidos entre `Professional`, `CustomerPF`/`CustomerPJ`, `Service`, `ServiceContract` e `Review`. A separação entre cliente pessoa física e jurídica mostra pensamento de produto real.

O backend está funcionando localmente. Três problemas precisam ser resolvidos antes de qualquer coisa no frontend: CORS, listagem geral e banco persistente.

**Decisão confirmada: o backend Java não será reescrito em Python.** O projeto tem 8 controllers implementados com lógica de relacionamento entre entidades. Reescrever jogaria fora meses de trabalho para chegar no mesmo lugar funcionalmente. O foco desta disciplina é o **frontend**.

---

## Stack

| Camada | Tecnologia |
|--------|-----------|
| Linguagem | Java |
| Framework | Spring Boot |
| Banco atual | H2 in-memory (temporário) |
| Banco de destino | PostgreSQL via Neon (gratuito) |
| Deploy previsto | Railway (backend) |

---

## Estrutura Atual

```
marketplace/src/main/java/br/com/marketplace/
├── controllers/
│   ├── AvaliaController.java      → /api/avalia
│   ├── ClienteController.java     → /api/cliente
│   ├── ClientePfController.java   → /api/clientePf
│   ├── ClientePjController.java   → /api/clientePj
│   ├── ContrataController.java    → /api/contrata
│   ├── PessoaController.java      → /api/pessoa
│   ├── ProfissionalController.java → /api/profissional
│   └── ServicoController.java     → /api/servico
├── models/
│   ├── Avalia.java        ← comentário, nota, data
│   ├── Cliente.java
│   ├── ClientePF.java     ← herda Cliente, tem CPF
│   ├── ClientePJ.java     ← herda Cliente, tem CNPJ
│   ├── Contrata.java      ← valor, data, forma de pagamento, status
│   ├── Endereco.java
│   ├── Especialidade.java
│   ├── Pessoa.java        ← base: id, nome, endereço, telefones
│   ├── Profissional.java  ← herda Pessoa
│   ├── Servico.java
│   └── Telefone.java
└── config/
    └── ModelMapperConfig.java
```

---

## Endpoints Disponíveis Hoje

Todos os 8 controllers seguem o mesmo padrão: `POST` para criar e `GET /{id}` para buscar por ID.

```
POST   /api/profissional       → Create professional
GET    /api/profissional/{id}  → Get professional by ID

POST   /api/servico            → Create service
GET    /api/servico/{id}       → Get service by ID

POST   /api/clientePf          → Create individual customer
GET    /api/clientePf/{cpf}    → Get individual customer by CPF

POST   /api/clientePj          → Create business customer
GET    /api/clientePj/{cnpj}   → Get business customer by CNPJ

POST   /api/contrata           → Create service contract
GET    /api/contrata/{id}      → Get contract by ID

POST   /api/avalia             → Create review
GET    /api/avalia/{id}        → Get review by ID
```

---

## Campos dos Modelos Principais

**Profissional / Professional:**
```json
{
  "nome": "string",
  "cnpj": 0,
  "classificacao": 0.0,
  "especialidade": "string",
  "endereco": {
    "rua": "", "numero": "", "bairro": "",
    "cidade": "", "estado": "", "cep": ""
  },
  "telefones": [{ "telefone": "" }]
}
```

**Contratação / ServiceContract:**
```json
{
  "valor": "string",
  "horarioAgendamento": "2026-01-01T10:00:00",
  "formaPagamento": "string",
  "idCliente": 1,
  "idProfissional": 1,
  "idServico": 1
}
```

**Avaliação / Review:**
```json
{
  "idCliente": 1,
  "idProfissional": 1,
  "comentario": "string",
  "nota": 4.5
}
```

---

## Problemas Críticos

### 1. Banco H2 com `create-drop`

**Severidade: Crítica** — Todos os dados somem ao reiniciar.

```properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.jpa.hibernate.ddl-auto=create-drop
```

`create-drop` é a configuração mais agressiva — cria o banco ao subir e apaga tudo ao desligar. Qualquer `Ctrl+C` apaga todos os dados cadastrados.

**O que fazer:** Migrar para PostgreSQL no Neon (gratuito, sem cartão de crédito). A mudança no `application.properties` é de 3 linhas:

```properties
spring.datasource.url=jdbc:postgresql://<host>.neon.tech/<db>?sslmode=require
spring.datasource.username=<user>
spring.datasource.password=<password>
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=update
```

### 2. Sem Listagem Geral — Marketplace Não Funciona Sem Isso

**Severidade: Crítica** — A funcionalidade central do produto não pode ser implementada no frontend.

Um marketplace precisa listar profissionais disponíveis. O endpoint `GET /api/profissional/{id}` busca por ID — o frontend precisaria saber o ID antecipadamente, o que não faz sentido para o usuário.

**O que adicionar em `ProfissionalController.java`:**
```java
@GetMapping
public ResponseEntity<List<Profissional>> listAll() {
    return ResponseEntity.ok(profissionalRepository.findAll());
}
```
O mesmo padrão se aplica ao `ServicoController`.

### 3. Sem CORS — Frontend Não Consegue Conectar

**Severidade: Crítica** — O browser bloqueia todas as requisições.

Nenhum controller tem `@CrossOrigin` e não há configuração global de CORS.

**Solução rápida — adicionar em cada controller:**
```java
@CrossOrigin(origins = "*")
@RestController
@RequestMapping("/api/profissional")
public class ProfissionalController { ... }
```

**Solução completa — arquivo de configuração global:**
```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
            .allowedOrigins("*")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH");
    }
}
```

### 4. Sem Autenticação

**Severidade: Alta** — Todos os endpoints abertos.

Qualquer pessoa com a URL pode cadastrar profissionais falsos, registrar contratações fraudulentas e criar avaliações. Será implementado ao longo do Bloco 2.

---

## Problemas de Modelagem

### `Long cnpj` no modelo Profissional

```java
private Long cnpj;  // ← errado
```

CNPJ tem 14 dígitos e pode ter zeros à esquerda. Como `Long`, o CNPJ `01234567000100` vira `1234567000100` — perde o zero. CPF e CNPJ sempre são `String`.

```java
private String cnpj;  // ← correto
```

---

## Padronização para Inglês

O mercado usa inglês no código. Recrutadores que abrem o repositório esperam inglês. Além disso, Axios, Spring e todos os frameworks estão em inglês — misturar cria inconsistência.

| Como está | Como deveria ser |
|-----------|-----------------|
| `Profissional` | `Professional` |
| `Contrata` | `ServiceContract` |
| `Avalia` | `Review` |
| `ClientePf` | `IndividualCustomer` |
| `ClientePj` | `BusinessCustomer` |
| `Servico` | `Service` |
| `listAll()` | já correto ✅ |

A renomeação pode ser feita gradualmente conforme o grupo mexe nos arquivos.

---

## Fetch vs Axios no Frontend

Quando o frontend for construído, a disciplina recomenda **Axios** no lugar de `fetch`.

**Por quê:**

`fetch` não trata erros HTTP automaticamente — um `404` não cai no `catch`, precisa verificar `response.ok` manualmente. Com Axios, qualquer `4xx` ou `5xx` lança uma exceção.

Axios também permite configurar interceptors — um único lugar para colocar o token JWT no header de todas as requisições sem repetir em cada arquivo.

**Importar via CDN (funciona em HTML puro — sem npm):**
```html
<script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>
```

---

## Resumo de Prioridades

| Prioridade | O que fazer |
|---|---|
| 🔴 Agora | Adicionar `@CrossOrigin` nos controllers |
| 🔴 Agora | Adicionar `GET /api/profissional` e `GET /api/servico` (listar todos) |
| 🔴 Agora | Criar estrutura de pastas do frontend |
| 🟡 Bloco 2 | Migrar H2 para PostgreSQL (Neon) |
| 🟡 Bloco 2 | Corrigir `Long cnpj` para `String cnpj` |
| 🟡 Bloco 2 | Implementar JWT |
| 🟡 Bloco 2 | Renomear modelos para inglês |
| 🟢 Bloco 3 | Deploy (Railway + GitHub Pages) |
| 🟢 Bloco 3 | README completo |

---

*Análise realizada com base no ZIP `marketplace-main-teste2-master` entregue em 18/03/2026.*
