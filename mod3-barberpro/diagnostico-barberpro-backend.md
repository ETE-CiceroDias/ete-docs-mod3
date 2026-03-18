# Diagnóstico de Backend — BarberPro
**Israel Paz · Wallace Cezar · Fagner Xavier · Valter Santos**
**DE_236 — Programação em Novas Tecnologias | Módulo 3**

🔗 Backend ao vivo: https://barberpro-api-1o2y.onrender.com  
🔗 GitHub: https://github.com/israel2640/Barberpro-java

---

## Visão Geral

O BarberPro tem o backend mais maduro da turma. JWT implementado corretamente, BCrypt nas senhas, PostgreSQL em produção no Neon, Docker configurado, deploy no Render funcionando. O diagnóstico aqui não é de problemas críticos — é de pontos que, corrigidos, levam o projeto do nível "funciona" para o nível "profissional de verdade".

---

## Stack

| Camada | Tecnologia |
|--------|-----------|
| Linguagem | Java 17 |
| Framework | Spring Boot 3.2.4 |
| Segurança | Spring Security + JWT (auth0 4.4.0) |
| Banco | PostgreSQL via Neon (cloud) |
| Deploy | Docker + Render |
| Extra | Lombok |

---

## Arquitetura

```
barberpro-api/src/main/java/br/com/barberpro/api/
├── config/
│   ├── CorsConfig.java              ← CORS configurado globalmente
│   ├── SecurityConfig.java          ← regras de acesso por role
│   ├── SecurityFilter.java          ← intercepta e valida o JWT em cada request
│   └── ApiProperties.java           ← lê variáveis do application.properties
├── controller/
│   ├── AutenticacaoController.java  ← POST /login (cliente)
│   ├── AutenticacaoBarbeiroController.java ← POST /login/barbeiro
│   ├── AdminController.java         ← CRUD de usuários e barbeiros (ADMIN only)
│   ├── AgendamentoController.java   ← criar, listar, deletar, concluir
│   ├── BarbeiroController.java      ← agenda do barbeiro logado (BARBER only)
│   ├── ClienteController.java       ← cadastro público de cliente
│   └── DadosController.java         ← endpoint público /api/dados
├── domain/                          ← entidades JPA
├── dto/                             ← DTOs separados do domínio ✅
├── repository/
└── service/
    ├── AuthenticationService.java
    └── TokenService.java            ← gera e valida JWT, secret via @Value
```

---

## O que Está Bem

**BCrypt nas senhas.** Todo cadastro e edição de usuário passa pelo `passwordEncoder.encode()` antes de salvar. Senha nunca fica em texto puro.

**JWT com controle de acesso por role.** O `SecurityConfig` define acesso granular:
```
POST /login, POST /login/barbeiro, POST /clientes  → público
/barbeiro/**                                        → só BARBER
/admin/**                                           → só ADMIN
qualquer outra rota                                 → autenticado
```

**Secret em variável de ambiente.** O `TokenService` lê o secret via `@Value("${api.security.token.secret}")` — não está hardcoded.

**Verificação de conflito de horário.** O `AgendamentoController` chama `existsByBarbeiroAndDataHora` antes de salvar. Dois clientes não conseguem agendar com o mesmo barbeiro no mesmo horário — retorna `409 Conflict`.

**DTOs separados do domínio.** `DadosDetalhamentoAgendamento`, `DadosCadastroCliente`, `DadosTokenJWT` — a entidade não é exposta diretamente pela API.

---

## Pontos de Atenção

### 1. CORS travado em URL hardcoded

**Severidade: Média** — Funciona em produção, quebra no desenvolvimento local.

```java
// CorsConfig.java — como está
.allowedOrigins("https://barberpro-frontend.onrender.com")
```

Qualquer membro do grupo que tentar testar o frontend localmente (`localhost:5500`) vai receber erro de CORS no browser. O desenvolvimento local fica inviável.

**O que fazer:**
```java
.allowedOrigins(
    "https://barberpro-frontend.onrender.com",
    "http://localhost:5500",
    "http://127.0.0.1:5500"
)
```

### 2. `System.out.println` de debug em produção

**Severidade: Baixa**

```java
// SecurityFilter.java
System.out.println("DEBUG: Usuário " + user.getUsername() + "...");
System.err.println("Erro ao validar o token JWT: " + e.getMessage());
```

Polui os logs do Render e pode expor informações desnecessárias. O padrão do mercado é usar um logger:

```java
private static final Logger log = LoggerFactory.getLogger(SecurityFilter.class);
log.debug("User {} authenticated: {}", user.getUsername(), user.getAuthorities());
log.error("Invalid JWT token: {}", e.getMessage());
```

### 3. Código em português — padronizar para inglês

O mercado usa inglês no código. Recrutadores que abrem o repositório esperam inglês. Misturar português com a nomenclatura em inglês do Spring cria inconsistência visual.

| Como está | Como deveria ser |
|-----------|-----------------|
| `Agendamento` | `Appointment` |
| `Barbeiro` | `Barber` |
| `gerarToken()` | `generateToken()` |
| `dataExpiracao()` | `getExpirationDate()` |
| `listarAgendamentos()` | `listAppointments()` |

A mudança pode ser feita gradualmente — não precisa acontecer de uma vez.

### 4. Sem migrações de banco (Flyway/Liquibase)

```properties
spring.jpa.hibernate.ddl-auto=update
```

`update` funciona, mas é frágil — se uma coluna for renomeada, pode causar comportamento inesperado silenciosamente. O padrão profissional é Flyway, que o próprio Intranet já usa (`V1__.sql`).

---

## Resumo de Prioridades

| Prioridade | O que fazer |
|---|---|
| 🟡 Importante | Adicionar `localhost` nas origens do CORS para desenvolvimento local |
| 🟡 Importante | Remover `System.out.println` — substituir por logger |
| 🟡 Importante | Migrar nomes para inglês gradualmente |
| 🟢 Melhoria | Considerar Flyway para controle de migrações |
| 🟢 Melhoria | Adicionar testes unitários nos services |

---

*Análise realizada com base no ZIP `Barberpro-java-main` entregue em 18/03/2026.*
