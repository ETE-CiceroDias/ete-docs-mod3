# Devolutiva Técnica — Intranet
**Davi Valeriano Silva · Emanoel Bruno · Vinicius Adorno · Guilherme**
**DE_236 — Programação em Novas Tecnologias | Módulo 3**

🔗 GitHub: https://github.com/Dav1777/Intranet.git

---

## Visão Geral

O Intranet tem um backend bem estruturado — com DTOs, Services, Repositories e Controllers separados corretamente, enums de Status e Prioridade bem definidos e até upload de arquivo implementado no chamado. Isso mostra maturidade de arquitetura.

O problema central é que **o frontend não está no repositório**. Seja qual for o HTML que o grupo tem localmente, ele precisa ser versionado, organizado e conectado ao backend. Esse é o ponto de partida desta disciplina para o grupo.

---

## O que Existe no Repositório

```
intranet/
└── src/
    └── main/java/com/saf/intranet/
        ├── controller/
        │   ├── ChamadoController.java
        │   ├── ConteudoController.java
        │   ├── FuncionarioController.java
        │   ├── InformativoController.java
        │   └── SetorController.java
        ├── dtos/               ← DTOs de request e response para todas as entidades
        ├── models/
        │   ├── Chamado.java
        │   ├── Conteudo.java
        │   ├── Endereco.java
        │   ├── Funcionario.java
        │   ├── Informativo.java
        │   ├── Prioridade.java  ← enum: BAIXA, NORMAL, ALTA, EMERGENCIA
        │   ├── Status.java      ← enum: ABERTO, TRIAGEM, REGISTRADO, EM_ANDAMENTO, FINALIZADO
        │   └── Setor.java
        ├── repositories/
        └── services/
```

**Não existe nenhum arquivo `.html`, `.css` ou `.js` no repositório.**

---

## API Disponível — O que o Frontend Pode Usar

Esses são os endpoints reais já implementados no backend. O frontend precisa consumir eles.

### Funcionários
```
POST   /funcionarios          → Cadastrar funcionário
GET    /funcionarios          → Listar todos os funcionários
GET    /funcionarios/{id}     → Buscar funcionário por ID
DELETE /funcionarios/{id}     → Deletar funcionário
```

**Campos do funcionário:**
```json
{
  "nome": "string",
  "email": "string",
  "cpf": "string",
  "cargo": "string",
  "idSetor": 1,
  "senha": "string",
  "telefone": "string",
  "logradouro": "string",
  "complemento": "string",
  "bairro": "string",
  "cidade": "string",
  "estado": "string",
  "cep": "string"
}
```

### Chamados
```
POST   /intranet/chamados              → Abrir chamado
GET    /intranet/chamados              → Listar todos os chamados
GET    /intranet/chamados/{id}         → Buscar chamado por ID
PATCH  /intranet/chamados/{id}/conteudo → Adicionar conteúdo/arquivo ao chamado
```

**Campos do chamado:**
```json
{
  "titulo": "string",
  "descricao": "string",
  "idSetor": 1,
  "idFuncionario": 1,
  "mensagem": "string"
}
```

**Status disponíveis:** `ABERTO` · `TRIAGEM` · `REGISTRADO` · `EM_ANDAMENTO` · `FINALIZADO`
**Prioridades disponíveis:** `BAIXA` · `NORMAL` · `ALTA` · `EMERGENCIA`

### Informativos
```
POST   /informativos           → Criar informativo
GET    /informativos           → Listar todos
GET    /informativos/publico   → Listar apenas os visíveis
GET    /informativos/{id}      → Buscar por ID
GET    /informativos/buscar?titulo=... → Buscar por título
DELETE /informativos/{id}      → Deletar
```

### Setores
```
POST   /intranet/setores            → Cadastrar setor
GET    /intranet/setores            → Listar todos
GET    /intranet/setores/buscar     → Buscar por nome
DELETE /intranet/setores/{id}       → Deletar
```

---

## Problemas Críticos no Backend

### 1. Banco H2 em Memória
**Severidade: Crítica** — Todos os dados somem quando o servidor reinicia.

```properties
# application.properties — como está
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
spring.jpa.hibernate.ddl-auto=update
```

Cada vez que o backend reinicia, o banco é recriado do zero. Em produção isso é inviável — qualquer deploy, qualquer reinício do servidor apaga todos os funcionários, chamados e informativos cadastrados.

**O que fazer:** Migrar para PostgreSQL (Neon — gratuito, sem cartão de crédito).
A mudança no `application.properties` é pequena, mas o impacto é total.

### 2. Senha em Texto Puro
**Severidade: Crítica** — Vulnerabilidade de segurança real.

```java
// FuncionarioRequestDTO.java
public record FuncionarioRequestDTO(
    String nome,
    String email,
    String cpf,
    String cargo,
    Long idSetor,
    String senha,  // ← armazenada como texto puro
    ...
)
```

A senha do funcionário é salva diretamente no banco sem nenhum hash. Se o banco vazar, todas as senhas ficam expostas. Em Java/Spring Boot, o BCrypt resolve isso com duas linhas:

```java
// No FuncionarioService, antes de salvar:
String senhaCriptografada = new BCryptPasswordEncoder().encode(dto.senha());
```

### 3. Sem Autenticação — Endpoints Abertos
**Severidade: Crítica** — Qualquer pessoa com a URL consegue acessar tudo.

Não existe nenhum `SecurityConfig`, nenhum filtro JWT, nenhuma verificação de token. Qualquer pessoa que descobrir a URL do backend consegue:
- Listar todos os funcionários com CPF e cargo
- Deletar funcionários
- Criar informativos falsos
- Ver todos os chamados e suas descrições

Isso será implementado ao longo da disciplina — mas é importante o grupo entender o risco.

### 4. Sem CORS Configurado
**Severidade: Alta** — O frontend não vai conseguir fazer fetch no backend.

Não há nenhuma configuração de CORS no projeto. Quando o frontend (rodando em um domínio diferente, como GitHub Pages) tentar fazer uma requisição para o backend, o navegador vai bloquear com erro `CORS policy`.

**O que fazer:** Adicionar `@CrossOrigin` nos controllers ou criar uma configuração global:

```java
// Solução rápida — adicionar em cada controller:
@CrossOrigin(origins = "*")
@RestController
@RequestMapping("/funcionarios")
public class FuncionarioController { ... }
```

---

## O que Precisa Ser Construído — Frontend

O frontend não existe. Essas são as telas mínimas necessárias para o projeto funcionar de ponta a ponta:

### Telas Obrigatórias (MVP)

**1. Login**
- Formulário com email e senha
- Fetch `POST /funcionarios/login` (quando JWT for implementado)
- Redirecionar para o dashboard após login

**2. Dashboard**
- Lista de chamados em aberto
- Lista de informativos recentes
- Botões de ação rápida: "Novo Chamado", "Ver Funcionários"

**3. Chamados**
- Listagem de chamados com status e prioridade visual (badge colorido por status)
- Formulário de abertura de chamado
- Fetch `GET /intranet/chamados` e `POST /intranet/chamados`

**4. Funcionários**
- Listagem de funcionários com nome, cargo e setor
- Formulário de cadastro
- Fetch `GET /funcionarios` e `POST /funcionarios`

**5. Informativos**
- Listagem de informativos
- Fetch `GET /informativos/publico`

### Estrutura de Pastas Recomendada
```
intranet-frontend/
├── index.html          ← login
├── dashboard.html      ← dashboard principal
├── chamados.html       ← listagem e abertura de chamados
├── funcionarios.html   ← listagem e cadastro
├── informativos.html   ← listagem
├── css/
│   ├── global.css      ← reset, variáveis, tipografia
│   ├── components.css  ← botões, cards, badges, formulários
│   └── layout.css      ← navbar, sidebar, grid
└── js/
    ├── config.js       ← API_URL centralizada
    ├── auth.js         ← verificação de token, logout
    ├── chamados.js     ← fetch de chamados
    ├── funcionarios.js ← fetch de funcionários
    └── informativos.js ← fetch de informativos
```

---

## Requisitos de Qualidade para o Frontend

Esses pontos serão cobrados na avaliação final:

### Semântica HTML
- Usar `<header>`, `<nav>`, `<main>`, `<section>`, `<footer>` corretamente
- Um único `<h1>` por página
- `<label>` com atributo `for` associado ao `id` de cada `<input>`
- Botões que submetem formulários devem ser `<button type="submit">`, não `<div>`

### Acessibilidade
- Todos os `<input>` devem ter `<label>` associado (nunca só `placeholder`)
- Imagens com `alt` descritivo
- Contraste mínimo 4.5:1 entre texto e fundo (WCAG AA)
- Navegação funcionando com Tab no teclado

### Usabilidade
- Feedback visual em todos os estados:
  - **Loading:** botão desabilitado e texto "Salvando..." enquanto o fetch processa
  - **Sucesso:** mensagem clara de confirmação (não `alert()`)
  - **Erro:** mensagem específica — o que deu errado e o que o usuário deve fazer
- Empty state — o que mostrar quando a lista de chamados está vazia
- Badge colorido por status do chamado:
  - 🟡 ABERTO · 🔵 TRIAGEM · 🟠 EM_ANDAMENTO · 🟢 FINALIZADO

### Responsividade
- O sistema precisa funcionar em telas a partir de 360px de largura
- Sidebar ou hamburger menu para mobile

---

## Resumo de Prioridades

| Prioridade | O que fazer |
|---|---|
| 🔴 Agora | Adicionar `@CrossOrigin` nos controllers (senão o frontend não conecta) |
| 🔴 Agora | Criar a estrutura de pastas do frontend e o primeiro HTML |
| 🔴 Agora | Construir tela de Login e Dashboard com fetch real |
| 🟡 Bloco 2 | Migrar H2 para PostgreSQL (Neon) |
| 🟡 Bloco 2 | Implementar hash de senha com BCrypt |
| 🟡 Bloco 2 | Implementar JWT para autenticação |
| 🟢 Bloco 3 | Deploy do backend (Railway) e frontend (GitHub Pages) |
| 🟢 Bloco 3 | README completo |

---

## README

O projeto não tem `README.md`. Para a avaliação final, o README é obrigatório e deve conter:
- Descrição do sistema (o que é a intranet, para quem)
- Tecnologias usadas
- Como rodar o backend localmente
- Como rodar o frontend localmente
- Endpoints disponíveis (resumo)
- Integrantes do grupo
