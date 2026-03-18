# Diagnóstico de Backend — Ease API
**Projeto:** Ease · Gestão Financeira para MEI  
**Repositório:** github.com/Lewis090/esasy-api-spring  
**Data da análise:** 18 de março de 2026  
**Disciplina:** Programação em Novas Tecnologias — DE_236

---

## 1. O que é o projeto

O Ease é uma aplicação de gestão financeira voltada especificamente para **MEI (Microempreendedor Individual)**. A escolha de nicho é um diferencial real: em vez de ser mais um app genérico de controle de gastos, ele resolve um problema concreto do empreendedor brasileiro — registrar receitas e despesas, acompanhar o teto anual permitido pela lei do MEI e projetar o fechamento do ano fiscal.

O sistema está na versão `v2.0` da API, o que indica que o grupo já passou por uma iteração de redesign — algo positivo.

---

## 2. Stack tecnológica

| Camada | Tecnologia |
|--------|-----------|
| Linguagem | Java |
| Framework | Spring Boot |
| Banco de dados | PostgreSQL |
| Containerização | Docker + Docker Compose |
| Testes de API | Insomnia |

A escolha de stack é adequada e coerente. Spring Boot + PostgreSQL é uma combinação sólida e amplamente usada no mercado.

---

## 3. Funcionalidades mapeadas (via coleção Insomnia)

A coleção `insomnia_v2_ease.json` revela os endpoints implementados e dá uma visão clara do que o sistema faz hoje:

### 3.1 Usuário / Perfil da empresa
| Método | Endpoint | O que faz |
|--------|----------|-----------|
| `POST` | `/api/v2/user/foto-perfil?userId=1` | Upload de foto de perfil |
| `PUT` | `/api/v2/user/perfil?userId=1` | Atualiza dados da empresa MEI (nome, se tem funcionário, salário do funcionário) |

### 3.2 Lançamentos financeiros
| Método | Endpoint | O que faz |
|--------|----------|-----------|
| `POST` | `/api/v2/lancamentos?userId=1` | Cria um lançamento (receita ou despesa) |
| `GET` | `/api/v2/lancamentos?userId=1` | Lista todos os lançamentos do usuário |

Os tipos de lançamento identificados são `RECEITA` e `DESPESA_FIXA`, com campo `formaRecebimento` (ex: `PIX`, `DINHEIRO`). O modelo de dados demonstra que o grupo pensou além do CRUD básico.

### 3.3 Projeções e controle fiscal
| Método | Endpoint | O que faz |
|--------|----------|-----------|
| `GET` | `/api/v2/projecoes?ano=2026&mes=6&userId=1` | Consulta projeção financeira mensal |
| `GET` | `/api/v2/anos-fiscais?userId=1` | Consulta status do teto anual MEI |

O endpoint de `anos-fiscais` é o mais sofisticado do sistema — ele rastreia o limite de faturamento anual do MEI (R$ 81.000/ano). Esse é um requisito de negócio muito específico e mostra que houve pesquisa sobre o domínio do problema. **Ponto alto do projeto.**

---

## 4. Infraestrutura

A aplicação usa Docker Compose para subir o PostgreSQL e a API juntos, o que é uma boa prática para desenvolvimento local. O README documenta os comandos de inicialização com clareza.

**O que está funcionando bem:**
- Health check configurado no PostgreSQL dentro do Docker
- URL JDBC com `sslmode=disable` para evitar erro de SSL em ambiente local
- `leia-me.txt` com troubleshooting de problemas comuns (conflito de container, etc.)

---

## 5. Pontos críticos — precisam ser resolvidos

### 5.1 Autenticação ausente na API

Todos os endpoints usam `?userId=1` como parâmetro na URL para identificar o usuário. Isso significa que **qualquer pessoa que souber o ID de outro usuário pode acessar os dados dele** — sem nenhuma validação de identidade.

O projeto precisa de autenticação via **JWT (JSON Web Token)**. Sem isso:
- O deploy não pode ser considerado seguro
- O critério de autenticação da avaliação não está atendido

**Ação necessária:** Implementar endpoint de login que retorne um token JWT, e proteger os endpoints existentes com validação do token.

### 5.2 Arquivos que não devem estar no repositório

Os seguintes itens estão sendo versionados no GitHub e não deveriam estar:

- **`target/`** — pasta de build do Maven. É gerada automaticamente ao compilar. Deve estar no `.gitignore`.
- **`.idea/`** — configurações da IDE (IntelliJ). São locais de cada desenvolvedor e não pertencem ao repositório.
- **`data/postgres`** — dados do volume do Docker. Contém o estado local do banco de dados. Não deve ser versionado.

Esses arquivos inflam o repositório e podem causar conflitos quando outros membros do time clonarem o projeto.

**Ação necessária:** Adicionar ao `.gitignore` e remover esses arquivos do histórico do repositório.

### 5.3 Arquivos duplicados com "Copia" no nome

Existem três versões do docker-compose e duas do pom.xml no repositório:

```
docker-compose.yml
docker-compose - Copia.yml
docker-compose - Copia (2).yml

pom.xml
pom - Copia.xml
```

Isso indica que o grupo estava usando "salvar cópia" como estratégia de versionamento — exatamente o problema que o Git resolve. Os arquivos duplicados devem ser removidos; o Git já mantém o histórico de todas as versões anteriores.

---

## 6. Pontos de atenção — importantes, mas não bloqueantes

### 6.1 README em inglês

O README está escrito em inglês, mas o projeto é inteiramente em português (nomes de variáveis, campos, coleção Insomnia). Há também uma frase que indica geração automática: *"If you want I can add a simple retry-on-startup... say the word and I will implement it"*.

O README precisa ser reescrito pelo grupo — em português, com as seções obrigatórias da disciplina, e com a voz do time, não de uma ferramenta.

### 6.2 Histórico de commits insuficiente

O repositório tem **apenas 4 commits** no branch `main`. Para um projeto que já está na versão v2.0 da API com múltiplos endpoints implementados, isso indica que o código foi desenvolvido fora do Git e empurrado de uma vez — ou em poucos lotes grandes.

O histórico de commits é um dos critérios de avaliação. A partir de agora, cada mudança deve virar um commit com mensagem no padrão Conventional Commits.

### 6.3 Sem branches, sem Issues, sem Pull Requests

O repositório não tem uso de branches, nenhuma Issue aberta e nenhum Pull Request. Isso mostra que o fluxo de trabalho colaborativo ainda não foi adotado.

Para o restante do projeto, o grupo deve trabalhar com a seguinte estrutura mínima: branch por feature, merge via Pull Request, e Issues para rastrear tarefas pendentes.

---

## 7. O que está bem — reconhecimentos

- **Nicho definido e diferenciado.** MEI é um problema real, com regras fiscais específicas que o grupo entendeu e modelou na API.
- **Endpoint de teto anual.** Demonstra compreensão do domínio — vai além do CRUD e resolve uma dor real do público-alvo.
- **Docker configurado.** A infraestrutura local funciona e está documentada.
- **Coleção Insomnia no repositório.** Facilita muito os testes e o onboarding de novos membros.
- **Versão v2.0.** Indica que houve iteração e evolução do design da API.

---

## 8. Resumo de prioridades

| Prioridade | Item |
|------------|------|
| 🔴 Crítico | Implementar autenticação JWT |
| 🔴 Crítico | Remover `target/`, `.idea/` e `data/postgres` do repositório |
| 🟡 Importante | Excluir arquivos "Copia" duplicados |
| 🟡 Importante | Reescrever o README em português com as seções obrigatórias |
| 🟡 Importante | Adotar commits frequentes com mensagens no padrão Conventional Commits |
| 🟢 Melhoria | Criar branches por feature e Pull Requests |
| 🟢 Melhoria | Abrir Issues no GitHub para rastrear as tarefas pendentes |

---

*Análise realizada com base no repositório público `Lewis090/esasy-api-spring` em 18/03/2026.*
