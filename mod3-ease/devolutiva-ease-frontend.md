# Devolutiva Técnica — Frontend EASE
**Sistema Financeiro para MEI · React 19 + Vite 7**  
**Programação Web · Módulo 3 · ETE Cícero Dias · Março 2026**

---

## Contexto

O EASE é o projeto mais completo e mais maduro da turma. O frontend está funcional, conectado ao backend Spring, com autenticação JWT, dashboard com abas, FAQ, termos de uso (com conformidade LGPD), formulário de contato, página institucional e fluxo de cadastro em dois passos. Isso é produto, não exercício.

Esta devolutiva não é sobre funcionalidade faltante — é sobre o que, agora que o projeto funciona, pode ser refatorado para elevar a qualidade técnica e deixar o código defensável em entrevista e em portfólio público.

> Antes de tudo: o grupo demonstra domínio real do domínio do problema. O cálculo de projeção de faturamento, o monitoramento do teto MEI, os alertas progressivos — tudo isso mostra que o grupo entendeu o produto, não só a tecnologia. Isso é raro.

---

## 1. Arquitetura e Estrutura

### ❌ Arquivo único de 2.500+ linhas — crítico

O arquivo `ease-v3-frontend.jsx` tem mais de 2.500 linhas com todos os componentes, estilos, constantes e lógica de API num único lugar.

**Por que isso é problema:**
- Qualquer bug exige buscar em 2.500 linhas
- Nenhum membro novo do time consegue se orientar
- Git blame e code review ficam inúteis
- É o principal indicador de dívida técnica que um entrevistador identifica ao abrir o repositório

**Como resolver:**

```
src/
├── components/
│   ├── Nav.jsx
│   ├── Logo.jsx
│   ├── Footer.jsx
│   ├── AnimatedCard.jsx
│   └── FAQAccordion.jsx
├── pages/
│   ├── HomePage.jsx
│   ├── AboutPage.jsx
│   ├── ContactPage.jsx
│   ├── LoginPage.jsx
│   ├── SignupPage.jsx
│   └── TermsPage.jsx
├── dashboard/
│   ├── DashboardPage.jsx
│   ├── DashOverview.jsx
│   ├── DashLancamentos.jsx
│   ├── DashDocumentos.jsx
│   └── DashNotificacoes.jsx
├── services/
│   └── api.js          ← o objeto api e parseApiResponse saem pra cá
├── styles/
│   └── theme.js        ← o objeto C de cores sai pra cá
└── hooks/
    └── useViewportFlags.js
```

Essa separação não muda nenhum comportamento — só reorganiza. O resultado é um repositório que parece profissional ao ser aberto.

---

### ⚠️ CSS 100% inline — atenção

Todos os estilos estão escritos como objetos JS direto no JSX: `style={{ color: '#fff', padding: 20 }}`.

**O que funciona:** as classes `.nf`, `.nc`, `.np` e `.btn-*` já estão no `index.css` e estão sendo reutilizadas — isso é positivo.

**O que pode melhorar:** os estilos específicos de cada componente ainda são 100% inline, o que:
- Torna o JSX verboso e difícil de ler
- Impede uso de pseudoclasses nativas (`:hover`, `:focus`) sem lógica JS extra
- Dificulta responsividade declarativa

**Próximo passo prático:** mover os estilos de componentes repetidos para classes no `index.css` ou criar arquivos `.module.css` por componente.

---

### ✅ Lógica de API centralizada — positivo

O objeto `api` com os métodos `get`, `post`, `put`, `delete` e `upload` é uma boa decisão de arquitetura. A função `parseApiResponse` que normaliza erros e a função `buildHeaders` com o controle de `skipAuth` para rotas públicas estão corretas e seguem o padrão do mercado.

Quando separar em módulos, esse objeto merece um arquivo próprio: `src/services/api.js`.

---

### ✅ Constante C para cores — base boa

Centralizar as cores num objeto é uma boa prática. O próximo passo natural é migrar para CSS custom properties:

```css
/* src/styles/variables.css */
:root {
  --color-primary: #EC8A3F;
  --color-navy: #2D3A49;
  --color-green: #3DB88A;
  --color-bg: #EDEAE4;
}
```

Com variáveis CSS, trocar tema claro/escuro vira uma linha de código. Com objeto JS, exige lógica de estado em cada componente.

---

## 2. Qualidade do Código React

### ⚠️ `useEffect` com `user.id` como dependência — atenção

No `DashboardPage`, `user` é criado com `JSON.parse` dentro do render:

```javascript
const user = JSON.parse(localStorage.getItem("user") || '{"id":1, "nome":"Usuário"}');
```

Isso recria o objeto a cada render. Quando `user.id` entra como dependência do `useEffect`, pode causar re-renders desnecessários.

**Correção:**

```javascript
const user = useMemo(
  () => JSON.parse(localStorage.getItem("user") || '{"id":1,"nome":"Usuário"}'),
  []
);
```

---

### ⚠️ Tratamento de erro com `alert()` — atenção

Os `try/catch` existem e estão corretos. Mas o tratamento de erro usa `alert()` em praticamente todos os casos.

`alert()` é aceitável em protótipos. Em produto: bloqueia a thread do browser, não é estilizável, e a UX é ruim.

**Sugestão:** criar um componente `Toast` simples com estado e substituir os `alert()`. Esse exercício também serve de prática de estado compartilhado entre componentes.

---

### ✅ ESLint configurado

O projeto tem `eslint.config.js` com `react-hooks` e `react-refresh`. A maioria dos alunos ignora lint completamente — o grupo não. O `eslint-disable-next-line` no componente `Nav` indica consciência dos avisos. Próximo passo: corrigir em vez de suprimir.

---

### ⚠️ Navegação por estado (sem React Router) — atenção

A navegação usa um estado React (`page`) para trocar componentes. Isso funciona, mas tem implicações:

- O botão Voltar do browser não funciona
- Não é possível compartilhar uma URL específica
- Leitores de tela não conseguem anunciar mudança de página

**Para o estágio atual:** aceitável. **Para o projeto final:** a instalação do React Router é necessária. Isso será coberto no Bloco 4 da disciplina.

---

### ⚠️ `localStorage` para token JWT — atenção consciente

É o padrão mais comum em projetos de nível técnico e está correto para este contexto. O ponto de atenção para o Bloco 2: `localStorage` é vulnerável a ataques XSS — o padrão mais seguro usa `httpOnly cookies`. Registrar como débito técnico consciente, não como erro.

---

## 3. Acessibilidade e Semântica

### ❌ `lang="en"` no index.html — crítico

```html
<!-- atual — errado -->
<html lang="en">

<!-- correto -->
<html lang="pt-BR">
```

O projeto é 100% em português. Com `lang="en"`, leitores de tela pronunciam o conteúdo com sotaque inglês e o Google indexa o idioma errado. **Essa é uma correção de uma linha que precisa acontecer hoje.**

---

### ⚠️ Botões sem `aria-label` no dashboard — atenção

O botão de fechar banner e o de alternar tema têm `aria-label` — correto. Mas outros botões icônicos no dashboard não têm.

**Regra:** todo botão que contém apenas ícone (sem texto visível) precisa de `aria-label`.

Após o deploy, rodar o [wave.webaim.org](https://wave.webaim.org) na URL do projeto vai mostrar exatamente onde estão os problemas.

---

### ✅ Logo SVG inline — bem feito

O componente `Logo` usa SVG inline com `viewBox` correto, sem `width`/`height` fixos no elemento raiz — o que permite escalar com CSS. O parâmetro `inv` para alternar cores entre fundos claros e escuros é uma boa decisão de componente.

---

## 4. Documentação

### ✅ README — base muito boa

O README tem: descrição clara do produto, tecnologias, requisitos de ambiente, como executar, scripts disponíveis, estrutura de pastas, fluxo de navegação, integração com API e seção de solução de problemas.

Está acima da média da turma. É um README que um dev consegue seguir de verdade.

---

### ⚠️ README sem acentuação — corrigir

O README usa "Aplicacao", "Pagina", "Configuracao", "Visao Geral" — sem acentos. Provavelmente foi gerado ou editado com encoding incorreto. Precisa revisar e corrigir. Um README sem acentos num projeto em português parece descuidado.

---

### ⚠️ Screenshot/demo ausente — atenção

O README não tem screenshot da interface nem link para o projeto no ar. É a primeira coisa que um recrutador olha — projeto sem imagem é ignorado. Após o deploy, adicionar gif ou screenshot do dashboard e da página inicial.

---

### ❌ GitHub Wiki não criada — crítico

O repositório não tem Wiki configurada. A Wiki é entregável obrigatório da Sprint 1. Precisa criar e adicionar pelo menos a página de **Requisitos** com RF e RNF. O conteúdo já existe — o grupo conhece bem o domínio do produto.

**Passos:**
1. Acessar o repositório no GitHub
2. Settings → Features → marcar **Wikis**
3. Aba Wiki → **Create the first page**
4. Criar a página `Requisitos` com a tabela de RF e RNF

---

## 5. Deploy e Infraestrutura

### ✅ GitHub Actions configurada — impressionante

O projeto tem `.github/workflows/ci.yml`. Isso é raro na turma e demonstra maturidade técnica. O grupo entendeu o ciclo de integração contínua antes do tema aparecer formalmente na disciplina.

---

### ✅ `.env.example` presente

O arquivo `.env.example` com `VITE_API_BASE` está correto — é a forma certa de documentar variáveis de ambiente sem expor valores reais. O `.env` real está no `.gitignore` (confirmado).

---

### ⚠️ Deploy ainda não realizado — atenção

O projeto ainda não tem URL pública. Backend conectado localmente, mas deploy de frontend e backend ainda não foi feito.

**Sugestão:**
- Frontend → Vercel (suporte nativo para Vite, deploy em 2 minutos com o GitHub conectado)
- Backend Spring → Railway ou Render (suportam JVM, free tier disponível)

O grupo está adiantado o suficiente para fazer isso antes do Bloco 3.

---

## 6. Prioridades

### 🔴 Fazer agora (esta semana)

| O que | Por que é urgente |
|---|---|
| Corrigir `lang="en"` → `lang="pt-BR"` no `index.html` | Uma linha. Impacto real em acessibilidade e SEO. Não tem justificativa para estar errado. |
| Criar GitHub Wiki com página de Requisitos | Entregável obrigatório da Sprint 1 — está em atraso. |
| Corrigir acentuação do README | O README é a vitrine do projeto. Com erro de encoding, passa impressão de descuido. |

---

### 🟡 Próxima sprint

| O que | Impacto |
|---|---|
| Separar `ease-v3-frontend.jsx` em componentes individuais | Maior melhoria de qualidade técnica disponível no projeto. |
| Adicionar screenshot/demo ao README | Visibilidade imediata para recrutadores após o deploy. |
| Substituir `alert()` por componente de feedback visual | Qualidade de UX e prática de estado compartilhado. |
| Adicionar `aria-label` nos botões icônicos do dashboard | Acessibilidade. Requisito de mercado. |

---

### 🟢 Bloco 2 e 3 (planejado)

| O que | Quando |
|---|---|
| Deploy no Vercel (frontend) + Railway/Render (backend) | Bloco 3 — mas o grupo pode fazer antes |
| Migrar cores do objeto `C` para CSS custom properties | Bloco 2 — junto com a refatoração de estilos |
| Instalar React Router para URLs reais e botão Voltar funcional | Bloco 4 — junto com o conteúdo de React |
| Estudar `httpOnly cookies` como alternativa ao `localStorage` para JWT | Bloco 2 — conteúdo de segurança |

---

## Fechamento

O EASE é o projeto mais profissional da turma. Os três pontos críticos desta devolutiva — `lang`, Wiki e acentuação do README — são correções pequenas que estão abaixo do nível técnico do grupo. São simples de resolver e precisam ser resolvidas antes da próxima sprint.

A separação em componentes e o deploy são os dois próximos passos que transformam o projeto de "impressionante para nível técnico" para "pronto para portfólio público".

Qualquer dúvida sobre os pontos desta devolutiva: me procurem nas aulas de terça e quinta, ou via Classroom.

---

*Prof. Samara Silvia · Programação Web · Módulo 3 · ETE Cícero Dias · Março 2026*
