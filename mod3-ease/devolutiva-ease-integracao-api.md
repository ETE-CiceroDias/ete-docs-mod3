# Devolutiva Técnica — EASE | Integração Front-end ↔ API Spring

**Grupo:** EASE  
**Módulo:** 3 — Programação em Novas Tecnologias  
**Assunto:** Conexão do front-end React com a API Spring Boot hospedada na Oracle  

---

## O que foi analisado

Foram analisados os dois repositórios entregues:

- `ease-frontend-main` — projeto React + Vite
- `esasy-api-spring-main` — projeto Spring Boot + PostgreSQL + JWT

O código de ambos está bem estruturado. O problema **não é de lógica**, é de **configuração de ambiente**. São 3 ajustes pontuais que precisam ser feitos.

---

## Problema 1 — O front não sabe o endereço da API

### O que está acontecendo

O arquivo `src/services/api.js` busca o endereço da API assim:

```js
const API_BASE = (import.meta.env.VITE_API_BASE || "http://localhost:8080")
```

Isso significa: *"use a variável de ambiente VITE_API_BASE. Se ela não existir, usa localhost:8080".*

Quando você abre o front no navegador sem ter configurado essa variável, ele tenta chamar `http://localhost:8080` — que é a sua própria máquina, não o servidor da Oracle. Por isso a conexão falha.

### Como corrigir

Na **raiz do projeto front-end** (mesma pasta do `package.json`), crie um arquivo chamado `.env`:

```
VITE_API_BASE=http://SEU_IP_DA_ORACLE:8080
```

Substitua `SEU_IP_DA_ORACLE` pelo IP público da sua instância na Oracle Cloud.

> **Exemplo:** Se o IP da sua Oracle for `129.153.12.45`, o arquivo fica assim:
> ```
> VITE_API_BASE=http://129.153.12.45:8080
> ```

Depois de criar o `.env`, rode novamente:

```bash
npm run dev       # para testar local apontando pra Oracle
# ou
npm run build     # para gerar o build de produção
```

> ⚠️ O arquivo `.env` **não deve ser commitado** no GitHub (ele já está no `.gitignore` do projeto). Cada desenvolvedor cria o seu localmente.

---

## Problema 2 — O back-end está bloqueando o front (CORS)

### O que está acontecendo

O arquivo `CorsConfig.java` define quais endereços têm permissão de fazer requisições para a API:

```java
.allowedOriginPatterns(
    "http://127.0.0.1:5500",
    "http://localhost:5500",
    "http://127.0.0.1:5173",
    "http://localhost:5173"
)
```

Ou seja, só `localhost` está liberado. Se você abrir o front de qualquer outro endereço (outro computador, Vercel, GitHub Pages, ou até pelo IP da rede), o navegador vai bloquear a chamada com um erro de CORS — mesmo que a API esteja funcionando perfeitamente.

### Como corrigir

Abra o arquivo:
```
src/main/java/com/easy/easyapi/config/CorsConfig.java
```

E adicione os endereços onde o front vai rodar:

```java
@Override
public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/**")
        .allowedOriginPatterns(
            "http://127.0.0.1:5500",
            "http://localhost:5500",
            "http://127.0.0.1:5173",
            "http://localhost:5173",
            "http://SEU_IP_DA_ORACLE:5173",      // ← se o front rodar na Oracle também
            "https://SEU-PROJETO.vercel.app"     // ← se hospedar no Vercel/Netlify
        )
        .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
        .allowedHeaders("*")
        .allowCredentials(true);
}
```

Depois de alterar, **rebuild e reinicie a API** no servidor:

```bash
sudo docker compose down
sudo docker compose up -d --build
```

---

## Problema 3 — Porta 8080 precisa estar aberta na Oracle

### O que está acontecendo

A Oracle Cloud bloqueia portas por padrão. Mesmo que a API esteja rodando, ela pode não estar acessível externamente se a porta `8080` não foi liberada nas regras de firewall.

### Como verificar e corrigir

**Passo 1 — No painel da Oracle Cloud:**

1. Acesse sua instância em **Compute > Instances**
2. Clique na sua instância → **Subnet** → **Security List**
3. Em **Ingress Rules**, verifique se existe uma regra para a porta `8080`
4. Se não existir, adicione:
   - **Source CIDR:** `0.0.0.0/0`
   - **IP Protocol:** TCP
   - **Destination Port:** `8080`

**Passo 2 — No firewall do próprio servidor (iptables/firewalld):**

Conecte no servidor via SSH e rode:

```bash
sudo iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
```

Para deixar permanente:

```bash
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

**Passo 3 — Teste rápido:**

Abra no navegador ou no Insomnia:
```
http://SEU_IP_DA_ORACLE:8080/auth/login
```

Se aparecer uma resposta (mesmo que seja erro 405 ou similar), a API está acessível. Se travar sem resposta, a porta ainda está bloqueada.

---

## Checklist — Passo a passo para funcionar

Siga essa ordem:

- [ ] **1.** Verificar se a porta 8080 está aberta na Oracle (Security List + iptables)
- [ ] **2.** Confirmar que a API está rodando: `sudo docker ps` → deve aparecer o container `easy-api-spring`
- [ ] **3.** Testar a API diretamente no Insomnia com o IP da Oracle
- [ ] **4.** Editar `CorsConfig.java` adicionando o endereço do front-end
- [ ] **5.** Rebuild da API: `sudo docker compose down && sudo docker compose up -d --build`
- [ ] **6.** Criar o arquivo `.env` na raiz do front-end com `VITE_API_BASE=http://SEU_IP:8080`
- [ ] **7.** Rodar o front: `npm run dev`
- [ ] **8.** Testar o login no navegador e abrir o **DevTools > Network** para ver se as requisições estão saindo corretamente

---

## Como identificar o erro no navegador

Quando algo não funciona, abra o **DevTools** (F12) e vá na aba **Network** ou **Console**.

| O que aparece | O que significa |
|---|---|
| `net::ERR_CONNECTION_REFUSED` | A API não está acessível (porta bloqueada ou container parado) |
| `CORS policy: No 'Access-Control-Allow-Origin'` | CORS não configurado para o seu endereço |
| `net::ERR_NETWORK_CHANGED` | Está chamando `localhost` em vez do IP da Oracle |
| `401 Unauthorized` | Token JWT inválido ou ausente — a conexão funcionou! |
| `200 OK` | Tudo certo ✅ |

---

## Observação final

O código de vocês está correto e bem organizado. O `api.js` já tem suporte a variável de ambiente, o JWT já está implementado, o CORS já tem a estrutura certa — só falta **apontar os endereços certos** para o ambiente de produção.

Isso é algo que acontece em todo projeto real: o código funciona local, mas precisa de configuração específica para funcionar em servidor. Faz parte do processo! 💪

---

*Devolutiva gerada pela Prof. Samara — ETE Cícero Dias | Módulo 3*
