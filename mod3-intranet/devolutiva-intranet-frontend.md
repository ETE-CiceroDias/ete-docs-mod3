# Devolutiva Técnica — Intranet (Frontend)
**Davi Valeriano Silva · Emanoel Bruno · Vinicius Adorno · Guilherme**
**DE_236 — Programação em Novas Tecnologias | Módulo 3**

🔗 GitHub: https://github.com/Dav1777/Intranet.git  
📁 Análise baseada no ZIP `intranet-feat-create-layout-sector`

---

## Visão Geral

O Intranet é o projeto com o contexto mais rico da turma — uma distribuidora real (Distribuidora União, 31 anos de mercado, 240+ funcionários) com setores definidos, fluxos de trabalho reais e um problema genuíno para resolver. Isso é uma vantagem enorme: o grupo não precisou inventar a persona, ela existe de verdade.

O frontend atual cobre uma parte considerável das telas. Existem arquivos para: login, home, chamados, almoxarifado, RH, informativos, comercial, financeiro, logística e cadastro. O problema não é ausência de conteúdo — é ausência de consistência, de JavaScript conectado ao backend, e de uma estrutura unificada que faça tudo funcionar junto como um sistema.

---

## O que Existe no Repositório

```
intranet-feat-create-layout-sector/
├── login/
│   └── index.html            ← login com formulário, SEM JS conectado
├── home/
│   ├── home.html             ← home com CSS path quebrado
│   ├── testing.html          ← versão funcional com CSS inline
│   └── logichome.js          ← lógica do carrossel
├── chamado/
│   └── secundex.html         ← formulário de abertura de chamado
├── almoxarifado/
│   ├── almoxarifado.html     ← formulário completo com Google Apps Script
│   └── almoxarife.html       ← versão alternativa
├── informativos/
│   └── informativos.html     ← com menu hamburger e submenus
├── RH/
│   └── rh.html               ← com carrossel e avisos estáticos
├── comercial/
│   └── coordenacao.html      ← estrutura incompleta, HTML inválido
├── financeiro/
│   └── fin.html              ← corpo praticamente vazio
├── Logistica/
│   └── log.html              ← corpo praticamente vazio
└── cadastro/
    └── cadastro.html         ← formulário de cadastro de funcionário
```

---

## 1. Sem Conexão com o Backend

**Severidade: Crítica** — O frontend existe, mas não conversa com o backend em nenhum ponto.

Todo o sistema de backend (Spring Boot, 5 controllers, endpoints de funcionários, chamados, informativos) é totalmente ignorado pelo frontend atual. Nenhum arquivo contém `fetch()` ou qualquer requisição HTTP.

Exemplos concretos:

**Login — `login/index.html`**
```html
<!-- Como está -->
<button onclick="submit">Entrar</button>
```
`onclick="submit"` não chama função nenhuma — `submit` é uma propriedade do formulário, não uma função. O botão não faz nada.

**O que precisa ser:**
```js
async function fazerLogin() {
  const matricula = document.getElementById('Matricula').value;
  const senha = document.getElementById('Senha').value;

  const response = await fetch('http://localhost:8080/funcionarios/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ matricula, senha })
  });

  if (response.ok) {
    const data = await response.json();
    localStorage.setItem('token', data.token);
    window.location.href = '../home/home.html';
  }
}
```

**Chamados — `chamado/secundex.html`**
O formulário tem campos de nome, resumo, descrição, setor e gravidade — todos corretos. Mas o botão "Registrar" tem um `<a href="#">` dentro que cancela qualquer submit. Nenhum `fetch POST /intranet/chamados` existe.

**Almoxarifado — `almoxarifado/almoxarife.html`**
Existe uma tentativa de fetch, mas aponta para um placeholder de Google Apps Script:
```js
const url = "COLE_AQUI_A_URL_DO_SEU_APP_SCRIPT"; // ← nunca foi preenchido
```
O almoxarifado precisa enviar para o endpoint real do backend — não para Google Sheets.

---

## 2. Dois HTMLs Concorrentes para a Home

**Severidade: Alta** — O grupo tem duas versões da home sem saber qual usar.

`home.html` importa um CSS com path quebrado (`/home/home.css` — barra inicial faz o browser procurar na raiz do servidor, não na pasta relativa). A página carrega sem estilo nenhum.

`testing.html` funciona mas tem todo o CSS embutido em `<style>` inline — mais de 100 linhas dentro do HTML.

**O que fazer:** Usar `testing.html` como base, extrair o CSS inline para um arquivo `home.css` separado, e deletar `home.html`. Ter dois arquivos para a mesma tela gera confusão no time e no Git.

---

## 3. HTML Inválido em Múltiplos Arquivos

**Severidade: Alta** — Erros estruturais que podem causar comportamento inesperado no browser.

### 3.1 `<body>` fora do lugar — `chamado/secundex.html`
```html
<!-- Como está -->
<html>
<head>...</head>
   <header>...</header>
<body>          ← body abre DEPOIS do header
```
O `<body>` precisa envolver o `<header>`, não vir depois dele. O navegador tenta corrigir automaticamente, mas o resultado é imprevisível.

### 3.2 Dois `<header>` aninhados — `fornecedorpasta.html`
```html
<header>
  <header class="header" id="cabeca">  ← header dentro de header
```
`<header>` não pode ser filho de outro `<header>`.

### 3.3 `</main>` sem abertura — `comercial/coordenacao.html`
```html
  <span class="material-symbols-outlined">arrow_back</span>
</header>

<div class="topcomercial">...</div>
</main>   ← nunca foi aberto
</main>   ← aparece duas vezes
```

### 3.4 `type` duplicado e inválido — `login/index.html`
```html
<input type="Matricula" id="Matricula" type="number" required>
```
O atributo `type` aparece duas vezes, e `type="Matricula"` não é um tipo HTML válido. Só `type="number"` deveria existir.

---

## 4. IDs Duplicados no Formulário de Chamados

**Severidade: Alta** — Quebra a lógica de qualquer JavaScript que use `getElementById`.

```html
<!-- chamado/secundex.html — todos com id="Nfunc" -->
<input id="Nfunc" ... placeholder="Digite seu nome">
<input id="Nfunc" ... placeholder="breve descrição chamado">
<input id="Nfunc" ... placeholder="descrição do chamado">
```

`id` deve ser único por página. Com três elementos tendo o mesmo `id`, `document.getElementById('Nfunc')` sempre retorna o primeiro — os campos de resumo e descrição ficam invisíveis para o JavaScript.

**Correto:**
```html
<input id="nome" ...>
<input id="resumo" ...>
<input id="descricao" ...>
```

---

## 5. Ausência de CSS Compartilhado

**Severidade: Alta** — Cada página tem seu próprio CSS com nomes inconsistentes. Nenhum código é reaproveitado.

| Arquivo | CSS |
|---------|-----|
| `home/home.html` | `home.css` (path quebrado) |
| `home/testing.html` | CSS inline |
| `chamado/secundex.html` | `styless.css` |
| `almoxarifado/almoxarifado.html` | `amox.css` |
| `Logistica/log.html` | `logist.css` |
| `RH/rh.html` | `rh.css` |

Cada página redefine os mesmos estilos de navbar, botões e inputs do zero — com valores diferentes. A navbar do `informativos.html` é azul escura. A do `RH/rh.html` segue outra paleta. São páginas do mesmo sistema que parecem aplicações diferentes.

A estrutura de pastas precisa ter um `css/global.css` com reset, variáveis e estilos compartilhados, importado em todos os arquivos.

---

## 6. Páginas Incompletas com Corpo Vazio

**Severidade: Média**

`financeiro/fin.html` e `Logistica/log.html` têm estrutura de header e navegação, mas o corpo está vazio. São telas que existem no menu de informativos mas não têm conteúdo — o usuário clica e vê uma página em branco.

Essas páginas precisam ter pelo menos um estado placeholder que comunique que o conteúdo está em construção, evitando a sensação de erro para o usuário.

---

## 7. Semântica e Acessibilidade

**Severidade: Média**

### 7.1 `<button>` com `<a>` dentro
Padrão repetido em múltiplos arquivos:
```html
<button><a href="#">Registrar</a></button>
<button><a href="../almoxarifado/almoxarifado.html">Solicitar Material</a></button>
```
`<a>` dentro de `<button>` é semanticamente inválido. Use um ou outro. Para navegação, use `<a class="btn">`. Para ação JavaScript, use `<button onclick="funcao()">`.

### 7.2 `alt` nas imagens do carrossel
```html
<!-- home.html -->
<img src="./img/INICIO.png" alt="Imagem 1">
<img src="./img/MEIOpFIM.png" alt="Imagem 2">
```
`alt="Imagem 1"` não descreve nada. Descreva o conteúdo real: `alt="Fachada do galpão da Distribuidora União"`.

### 7.3 `<label>` sem `for` em vários formulários
```html
<!-- cadastro.html -->
<label for="nome">Nome: </label>
<input id="nome" ...>   ← esse está correto

<!-- almoxarifado.html -->
<label>Tipo de material:</label>  ← sem for, radio buttons sem associação
```

### 7.4 Nenhum `<main>` na maioria das páginas
`rh.html` usa `<main>` corretamente. `home/testing.html` também. Os demais não — o conteúdo fica flutuando sem container semântico principal.

---

## 8. Nomenclatura de Arquivos e Pastas

**Severidade: Média**

| Problema | Arquivo |
|----------|---------|
| Nome genérico | `styless.css` — o "ss" é typo? Qual tela estiliza? |
| Nome abreviado | `amox.css` — almoxarifado |
| Nome abreviado | `logist.css` — logística |
| Nome abreviado | `rifado.js` — almoxarifado |
| Pasta com inicial maiúscula | `Logistica/` — inconsistente com as outras |
| Arquivo de marketing vazio | `marketing/markt.html` — arquivo existe no menu, body vazio |

Padrão a adotar: nomes descritivos, letras minúsculas, sem abreviações. `almoxarifado.css`, `logistica.css`, `chamados.js`.

---

## 9. Estrutura de Pastas — O que Precisa Mudar

**Estado atual:** cada setor é uma pasta separada com seus próprios CSS e JS.

**O problema:** não é estrutura de frontend — é estrutura de arquivos físicos da empresa. O sistema inteiro não tem um ponto de entrada único, nenhum CSS compartilhado, e cada "módulo" é uma ilha.

**Estrutura recomendada:**

```
intranet-frontend/
├── index.html              ← login (ponto de entrada único)
├── home.html               ← dashboard principal
├── chamados.html           ← listagem e abertura de chamados
├── funcionarios.html       ← listagem e cadastro
├── informativos.html       ← informativos públicos
├── css/
│   ├── reset.css           ← zera estilos do browser
│   ├── variables.css       ← paleta, tipografia (--primaria, --secundaria...)
│   ├── global.css          ← navbar, botões, formulários compartilhados
│   └── pages/
│       ├── home.css
│       ├── chamados.css
│       └── almoxarifado.css
├── js/
│   ├── config.js           ← API_URL centralizada
│   ├── auth.js             ← token, login, redirect
│   ├── chamados.js         ← fetch de chamados
│   ├── funcionarios.js     ← fetch de funcionários
│   └── informativos.js     ← fetch de informativos
└── assets/
    └── img/
```

---

## 10. O que Está Bem — Reconhecimentos

| O que | Por que vale destacar |
|---|---|
| `testing.html` | Layout de home bem executado — estrutura, carrossel, responsividade |
| `informativos.html` | Menu hamburger com submenus funcionais — boa UX |
| `almoxarifado.html` | Formulário mais completo do projeto — campos bem pensados para o contexto real |
| `rh.html` | Usa `<main>`, `<aside>` e `<section>` corretamente — melhor semântica do projeto |
| `cadastro.html` | `label` com `for` associado ao `id` em todos os campos — boas práticas de formulário |
| Variáveis CSS em `testing.html` | `--primaria`, `--secundaria`, `--claro` definidas — caminho certo |

---

## Prioridades para Esta Sprint

| Prioridade | O que fazer |
|---|---|
| 🔴 Agora | Conectar o login ao backend — `POST /funcionarios/login` com salvamento do token |
| 🔴 Agora | Unificar as duas homes — usar `testing.html` como base, extrair CSS para arquivo externo |
| 🔴 Agora | Corrigir HTML inválido: `body` fora do lugar em `chamados`, IDs duplicados, `type` duplicado |
| 🔴 Agora | Criar `css/global.css` e `css/variables.css` — importar em todos os HTMLs |
| 🟡 Bloco 2 | Conectar formulário de chamados ao backend — `POST /intranet/chamados` |
| 🟡 Bloco 2 | Conectar listagem de funcionários — `GET /funcionarios` |
| 🟡 Bloco 2 | Implementar JWT: token salvo, verificação de sessão, redirect se não logado |
| 🟡 Bloco 2 | Preencher páginas vazias (financeiro, logística) com conteúdo ou placeholder |
| 🟢 Bloco 3 | Migrar banco H2 para PostgreSQL e fazer deploy |
| 🟢 Bloco 3 | README completo |

---

*Análise realizada com base no ZIP `intranet-feat-create-layout-sector` entregue em 18/03/2026.*
