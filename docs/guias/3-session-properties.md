# 3. Enriquecimento de Dados — Session e Action Properties

## O Desafio: Reported Values vs. Propriedades

Reported values visíveis no detalhe de uma ação **não são consultáveis via USQL** — apenas properties registradas como "SDK reported value" no tenant são.

```
Ação no Dynatrace:
  ├─ Reported values (visível no detalhe)
  │  └─ ❌ Não consultável via USQL
  └─ Session/Action properties (registradas no UI)
     └─ ✅ Consultável: SELECT stringProperties.nivel_usuario FROM usersession
```

## Como Registrar uma Propriedade no Tenant

**Caminho no Dynatrace UI:**
`Frontend → [Nome do App] → ... → Edit → Session and user action properties → Add property`

| Campo           | Valor                                            |
| --------------- | ------------------------------------------------ |
| Expression type | SDK reported value                               |
| Key             | Nome exato usado no código (ex: `nivel_usuario`) |
| Display name    | Nome amigável                                    |
| Data type       | String ou Long                                   |
| Store as        | Session / User action / Session, User action     |

> ⚠️ A chave é **case-sensitive**. `nivel_usuario` ≠ `Nivel_Usuario`.  
> ⚠️ A sessão deve **iniciar após** o registro da propriedade no UI. Sessões abertas antes não são retroativamente enriquecidas.

## Properties Registradas Neste Projeto

| Chave            | Tipo   | Scope            | Código que reporta                                         |
| ---------------- | ------ | ---------------- | ---------------------------------------------------------- |
| `nivel_usuario`  | String | Session + Action | `dtRum.sessao()`, `dtRum.identificarUsuario()`             |
| `pontos_usuario` | Long   | Session          | `dtRum.sessao()`                                           |
| `segmento`       | String | Session + Action | `dtRum.sessao()`, `dtRum.identificarUsuario()`             |
| `tti_ms`         | Long   | Action           | `dtRum.telaCarregada()`                                    |
| `duracao_ms`     | Long   | Action           | `dtRum.withAction()`, `dtRum.apiCall()`, `dtRum.subAcao()` |
| `status`         | String | Action           | `dtRum.apiCall()`, `dtRum.subAcao()`                       |
| `startup_ms`     | Long   | Action           | `App.js — App - Cold Startup`                              |

## Mecanismo Correto: `reportStringValue` / `reportIntValue`

```js
// ✅ CORRETO — gera SDK reported value consultável no USQL
const action = Dynatrace.enterAction("Session - Atualizar Propriedades");
action.reportStringValue("nivel_usuario", "Gold");
action.reportIntValue("pontos_usuario", 15430);
action.leaveAction();

// ❌ INCORRETO — sendSessionPropertyEvent com chave sem namespace
// não é reconhecido pelo mecanismo SDK reported value do UI
evento.addSessionProperty("nivel_usuario", "Gold"); // chave sem prefixo válido
```

> **Por quê?** O `sendSessionPropertyEvent` exige chaves com namespace `session_properties.*` para ser aceito pelo validador interno do SDK. Porém o Dynatrace UI registra as propriedades pela chave "crua" (sem prefixo). Esse conflito torna o mecanismo inoperante. O `reportStringValue` em uma action é o único path garantido.

## Identificação de Usuário

```js
// dtRum.js
identificarUsuario: (userId, { nivel, segmento } = {}) => {
  Dynatrace.identifyUser(String(userId)); // Liga a sessão ao userId
  const action = Dynatrace.enterAction("Login - Usuário Identificado");
  action.reportStringValue("nivel_usuario", nivel);
  action.reportStringValue("segmento", segmento);
  action.leaveAction();
  dtRum.sessao({ nivel_usuario: nivel, segmento }); // Persiste na sessão
},
```

Com `identifyUser`, o SRE consegue:

- Buscar todas as sessões de um usuário específico em caso de reclamação
- Correlacionar sessão mobile com logs de backend que contêm o mesmo `userId`
- Ver o histórico completo de navegação antes de um crash

**USQL para buscar um usuário específico:**

```sql
SELECT userId, startTime, crashGroupId, stringProperties.nivel_usuario,
       longProperties.pontos_usuario, city, osVersion, appVersion
FROM usersession
WHERE userId = 'user_12345'
ORDER BY startTime DESC
```

## Garantir Envio do Beacon (Flush)

A sessão só é processada e fica disponível no USQL após ser **encerrada e enviada**. O projeto implementa flush automático ao ir para background:

```js
// App.js
const subscription = AppState.addEventListener("change", (nextState) => {
  if (appState.current === "active" && nextState === "background") {
    dtRum.encerrarSessao(); // flushEvents() + endSession()
  }
  appState.current = nextState;
});
```

Sem isso, sessões ficam abertas por até 30 minutos antes de serem processadas.

## Padrão de Uso Recomendado

```js
// 1. No login: identificar e criar propriedades iniciais
dtRum.identificarUsuario("user_12345", { nivel: "Gold", segmento: "PF" });

// 2. Após carregar dados críticos: atualizar propriedades da sessão
dtRum.sessao({
  nivel_usuario: usuario.nivel,
  pontos_usuario: usuario.pontos,
  segmento: usuario.nivel === "Gold" ? "premium" : "standard",
});

// 3. Ao sair: forçar flush do beacon
// (automático via AppState listener)
```

---

## Próxima Etapa

→ [4. Padrões e Boas Práticas SRE](4-padroes-sre.md)
