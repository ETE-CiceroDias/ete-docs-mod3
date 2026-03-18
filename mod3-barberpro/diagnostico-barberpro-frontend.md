# Diagnóstico de Frontend — BarberPro
**Israel Paz · Wallace Cezar · Fagner Xavier · Valter Santos**
**DE_236 — Programação em Novas Tecnologias | Módulo 3**

🔗 Frontend ao vivo: https://barberpro-frontend.onrender.com

---

## Visão Geral

O BarberPro é o projeto mais completo da turma. É o único com frontend **e** backend em produção, JWT implementado e responsividade funcional. O foco desta disciplina para o grupo não é construir mais — é refinar o que existe. Os problemas abaixo aparecem em code reviews e entrevistas técnicas. Corrigi-los leva o projeto ao nível profissional.

---

## Arquivos do Projeto

```
barberpro-frontend/
├── index.html           ← página inicial
├── registrar.html       ← login + cadastro de cliente
├── agendamento.html     ← fluxo de agendamento
├── barbairo.html        ← ⚠️ typo: deveria ser barbeiro.html
├── barbeiro-agenda.html ← dashboard do barbeiro
├── administrador.html   ← painel admin
├── proficional.html     ← ⚠️ typo: deveria ser profissional.html
├── inicio.html
├── login-admin.html
├── login-barbeiro.html
├── localização.HTML     ← ⚠️ acento + maiúsculo — quebra em Linux
├── serviços.HTML        ← ⚠️ acento + maiúsculo
├── portifolio.HTML      ← ⚠️ typo + maiúsculo
├── produtos.HTML        ← ⚠️ maiúsculo
├── menu.js
├── style.css
├── responsive.css
└── index.css
```

---

## 1. Nomes de Arquivos — Typos e Maiúsculos

**Severidade: Alta** — Quebra em servidores Linux (Render).

Linux diferencia maiúsculas de minúsculas. `Produtos.HTML` e `produtos.html` são arquivos diferentes para o servidor. No Windows funciona, no Render quebra silenciosamente.

| Arquivo atual | Como deveria ser |
|---|---|
| `barbairo.html` | `barber.html` |
| `proficional.html` | `professional.html` |
| `portifolio.HTML` | `portfolio.html` |
| `produtos.HTML` | `products.html` |
| `localização.HTML` | `location.html` |
| `serviços.HTML` | `services.html` |

Renomear **e** atualizar todos os `href` que apontam para eles.

---

## 2. CSS Duplicado em 8 Arquivos

**Severidade: Alta** — Dívida técnica. O bloco inteiro da `nav` está copiado como `<style>` inline em cada HTML.

Se precisar mudar a cor da navbar, são 8 arquivos para editar. Um erro em qualquer um deixa o site inconsistente.

**O que fazer:**
1. Mover o CSS da nav para o `style.css` (já existe, mas nenhum HTML o importa)
2. Remover os blocos `<style>` inline de todos os arquivos
3. Importar o `style.css` no `<head>` de todos os HTMLs

---

## 3. `API_URL` Hardcoded em 8 Arquivos

**Severidade: Alta**

```js
// Repetido em administrador.html, agendamento.html, barbairo.html...
const API_URL = "https://barberpro-api-1o2y.onrender.com";
```

**O que fazer — criar `config.js`:**
```js
// services/config.js
const CONFIG = {
  API_URL: "https://barberpro-api-1o2y.onrender.com"
};
```
```html
<!-- No <head> de cada HTML, antes dos outros scripts -->
<script src="../../services/config.js"></script>
```

---

## 4. fetch → Axios

**Severidade: Média** — O projeto usa `fetch` em todos os arquivos. A disciplina recomenda migrar para **Axios**.

**Por que Axios é o padrão do mercado:**

fetch não trata erros HTTP como erros — um `404` ou `500` não cai no `catch`, você tem que verificar `response.ok` manualmente. Com Axios, qualquer status `4xx` ou `5xx` lança uma exceção automaticamente.

Axios também elimina a necessidade de chamar `response.json()` manualmente — o dado já vem em `response.data`.

O mais importante para vocês: Axios permite configurar interceptors — um único lugar para colocar o token JWT no header de **todas** as requisições, sem repetir em cada chamada.

**Importar via CDN (funciona em HTML puro):**
```html
<script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>
```

**Antes (fetch):**
```js
const response = await fetch(`${API_URL}/login`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer ' + localStorage.getItem('jwt_token')
  },
  body: JSON.stringify(data)
});
if (!response.ok) throw new Error('...');
const result = await response.json();
```

**Depois (Axios):**
```js
// Configurar uma vez em auth.service.js:
axios.defaults.baseURL = CONFIG.API_URL;
axios.interceptors.request.use(config => {
  const token = localStorage.getItem('jwt_token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Em cada arquivo — só isso:
const { data } = await axios.post('/login', { email, password });
```

---

## 5. Tratamento de Erro Genérico

**Severidade: Alta**

```js
} catch (error) {
  alert('Erro de comunicação com o servidor.');  // ← sempre igual
}
```

**O que fazer:**
```js
try {
  const { data } = await axios.post('/login', credentials);
  // sucesso
} catch (error) {
  if (error.response?.status === 401) {
    showToast('E-mail ou senha incorretos.', 'error');
  } else if (error.response?.status === 409) {
    showToast('Este e-mail já está cadastrado.', 'error');
  } else {
    showToast('Algo deu errado. Tente novamente.', 'error');
  }
}
```

---

## 6. Token Expirado Sem Redirecionamento

**Severidade: Média**

Quando o JWT expira, o backend retorna `401`. O frontend não trata — a sessão trava silenciosamente.

**Com Axios, resolve em um lugar só:**
```js
// auth.service.js — interceptor de resposta
axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.clear();
      window.location.href = '/pages/login/index.html';
    }
    return Promise.reject(error);
  }
);
```

---

## 7. Sem Feedback de Loading

**Severidade: Média**

Nenhum botão desabilita durante o fetch. O usuário pode clicar várias vezes.

```js
async function handleLogin(event) {
  event.preventDefault();
  const btn = document.getElementById('btn-login');
  btn.disabled = true;
  btn.textContent = 'Signing in...';

  try {
    // axios aqui
  } finally {
    btn.disabled = false;
    btn.textContent = 'Sign in';
  }
}
```

---

## 8. Semântica HTML

**Severidade: Média**

- Falta `<main>` em todos os arquivos
- Dois `<h1>` em `barbeiro-agenda.html` — o segundo deveria ser `<h2>`
- `<a>` envolvendo `<button>` em `index.html` — usar um ou outro
- `alt` genérico nas imagens do portfólio: `alt="Corte 1"` não descreve nada

---

## 9. Acessibilidade

- Botão hamburger não atualiza `aria-expanded` ao abrir/fechar
- Links de redes sociais sem `aria-label` — só `title` não é lido por todos os leitores de tela
- Contraste `#a0a0a0` sobre `#0c0c0c` ≈ 3.9:1 — abaixo do mínimo WCAG AA (4.5:1). Trocar por `#c0c0c0`

---

## 10. Inglês no Código

Nomes de variáveis, funções e arquivos em inglês — padrão da indústria.

| Como está | Como deveria ser |
|-----------|-----------------|
| `dadosLogin` | `loginCredentials` |
| `dadosRegistro` | `registerData` |
| `formRegistro` | `registerForm` |
| `showRegister()` | já está certo ✅ |
| `registrar.html` | `pages/login/index.html` |
| `barbairo.html` | `pages/barber/index.html` |

---

## 11. Estrutura de Pastas — Migrar para Feature-Based

**Estado atual:** todos os arquivos na raiz, CSS inline em cada HTML.

**Estrutura recomendada:**

```
barberpro-frontend/
├── pages/
│   ├── login/        ← registrar.html
│   ├── schedule/     ← agendamento.html
│   ├── admin/        ← administrador.html
│   └── barber/       ← barbeiro-agenda.html
├── components/
│   ├── navbar/       ← CSS e JS da nav extraídos dos HTMLs
│   └── toast/        ← notificações visuais
├── styles/
│   ├── reset.css
│   ├── variables.css ← --color-primary: #ff9900 etc
│   └── global.css
├── services/
│   ├── config.js     ← API_URL
│   ├── auth.service.js ← interceptors, token, logout
│   └── api/
│       └── appointments.api.js
└── shared/
    └── utils.js
```

Ver arquivo `refatoracao-barberpro-exemplo/` para código completo.

---

## 12. O que Está Bem

| O que | Por que vale |
|---|---|
| JWT com 3 tipos de usuário | Implementação correta e bem pensada |
| `aria-label` no hamburger | Mostra consciência de acessibilidade |
| `label` com `for` em todos os formulários | Boa prática corrida desde o início |
| `preconnect` para Google Fonts | Otimização de performance raramente feita |
| `defer` nos scripts | Correto — não bloqueia o carregamento |
| Backend em produção | Separa o projeto de 99% dos alunos técnicos |

---

## Prioridades desta Sprint

| # | O que fazer |
|---|---|
| 1 | Renomear todos os arquivos — inglês, sem acento, sem maiúsculo |
| 2 | Criar estrutura de pastas feature-based |
| 3 | Extrair CSS da nav para `components/navbar/navbar.css` |
| 4 | Criar `services/config.js` com API_URL |
| 5 | Instalar Axios via CDN e criar `services/auth.service.js` com interceptors |
| 6 | Substituir `alert()` por toast visual |
| 7 | Adicionar `<main>` em todos os arquivos |
| 8 | Tratar 401 → redirect automático (via interceptor) |
