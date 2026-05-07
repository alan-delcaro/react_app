# 2. De Automático até 100% Customizado

## Decision Framework: Quando Customizar?

A chave é **balancear captura automática com insights acionáveis**. Use este framework:

```
┌─────────────────────────────────────────────────────────────┐
│ DECISÃO: Devo Instrumentar Manualmente Esta Ação?          │
├─────────────────────────────────────────────────────────────┤
│ ✅ SIM se:                  │ ❌ NÃO se:                     │
│ • Afeta revenue/conversão   │ • É detalhe técnico raro       │
│ • Usuário espera resultado  │ • Automático já nomeia bem      │
│ • Preciso de contexto SLA   │ • Volume de ações > 100/sessão │
│ • Erro de negócio (não bug) │ • Overhead supera valor        │
└─────────────────────────────────────────────────────────────┘
```

### Exemplo: App de Pontos de Fidelidade

| Ação                   | Automático?          | Customizar? | Razão                                       |
| ---------------------- | -------------------- | ----------- | ------------------------------------------- |
| Toque em "Clique Aqui" | ✅ Sim               | ❌ Não      | Nome genérico mas raro                      |
| Abrir aba "Prêmios"    | ✅ Sim               | ✅ Sim      | Jornada crítica → reportar TTI              |
| Processar pagamento    | ✅ Sim               | ✅ Sim      | Revenue critical → capturar erro de negócio |
| Scroll infinito        | ⚠️ Ruidoso           | ✅ Sim      | Muitos "swipes" → consolidar em 1 métrica   |
| Crash de JS            | ⚠️ Stack trace bruto | ✅ Sim      | Erro sem contexto → adicionar userId        |

---

## Estágio 1: Automático (0% Customização)

**Configuração:**

```js
// dynatrace.config.js
module.exports = {
  input: {
    instrument: true, // ✅ Captura TODOS os cliques, gestos, HTTP
  },
  lifecycle: {
    includeUpdate: true, // ✅ Captura ciclo de vida do React Native
  },
};
```

**O que você ganha de graça:**

- ✅ Todas as sessões com `init_timestamp`, `duration`, `crash_reported`
- ✅ Todos os taps, swipes, HTTP (GET/POST, status, latência)
- ✅ Crashes de JS com stack trace nativo
- ✅ Localização, SDK version, device info

**O que falta:**

- ❌ Nome das ações: `"Touch on View"` (qual componente?)
- ❌ Contexto de negócio: qual promoção o usuário viu?
- ❌ Erro de negócio: "Pagamento negado" vs. "HTTP 500"?
- ❌ SLA por jornada: quanto tempo de Home → Checkout?

**Use quando:**

- MVP com < 1000 usuários/dia
- Buscando baseline antes de otimizar
- Compliance (apenas deve coletar, não investigar)

---

## Estágio 2: Automático + Customizações Estratégicas (20% Customização)

**Configuração:**

```js
// dynatrace.config.js
module.exports = {
  input: {
    instrument: true, // ✅ Captura automática...
    suppressedPatterns: [
      /HomeScreen/, // ...exceto HomeScreen (vamos instrumentar manualmente)
    ],
  },
  lifecycle: {
    includeUpdate: false, // ❌ Suprime ruído de "Component Update"
  },
};
```

**O que adicionar manualmente:**

### 1️⃣ **User Journeys Críticas** — Medida de Negócio

```js
// Envolver tudo em HomeScreen num withAction
const HomeScreen = () => {
  const screenStartTime = useRef(Date.now());

  useEffect(() => {
    dtRum.withAction("Home - Carregar Dashboard", async (parent) => {
      // Sub-actions para entender gargalos
      const usuario = await dtRum.subAcao(parent, "API - Buscar Usuário", () =>
        api.getUsuario(),
      );
      const premios = await dtRum.subAcao(parent, "API - Buscar Prêmios", () =>
        api.getPremios(),
      );

      // TTI: quanto tempo até a página ficar pronta
      dtRum.telaCarregada("Home", screenStartTime.current, {
        nivel_usuario: usuario.nivel,
        total_premios: premios.length,
      });
    });
  }, []);
};
```

**Resultado no Dynatrace:**

```
Home - Carregar Dashboard               850ms ← ação pai
├─ API - Buscar Usuário                120ms   [sub-action]
├─ API - Buscar Prêmios                630ms   [sub-action] ⚠️ Lento!
└─ Home - Tela Carregada               tti_ms: 850
```

### 2️⃣ **Conversão & Revenue** — Erro de Negócio

```js
const processarPagamento = async () => {
  try {
    const result = await dtRum.apiCall("Pagamento - Processar", () =>
      api.post("/pagamento", { valor: 50 }),
    );

    if (result.status === "negado") {
      // ❌ Erro de NEGÓCIO (não bug técnico)
      dtRum.erroBusiness(
        `Pagamento negado: ${result.motivo}`,
        result.codigo, // ex: "insufficient_balance"
      );
    }
  } catch (err) {
    // ❌ Erro TÉCNICO (HTTP, timeout, etc)
    dtRum.erroBusiness(`Erro técnico: ${err.message}`, "TECH_ERROR");
  }
};
```

### 3️⃣ **Session Properties (USQL Queryable)**

```js
// Após login bem-sucedido
dtRum.identificarUsuario("user_123", {
  nivel: "Gold",
  segmento: "high_value",
});

// Atualizar propriedades em tempo real
dtRum.sessao({
  nivel_usuario: userData.nivel,
  pontos_usuario: userData.pontos,
  segmento_usuario: userData.segmento,
});
```

**Query no Dynatrace USQL:**

```sql
SELECT userid, app.build, COUNT(*) as crashes
FROM usersession
WHERE stringProperties.nivel_usuario = "Gold"
GROUP BY userid
```

**Use este estágio quando:**

- App tem 1K-100K usuários/dia
- Modelo de negócio depende de conversão
- SRE precisa entender "por que o TTI aumentou?"

---

## Estágio 3: 100% Customizado (100% Customização)

**Configuração:**

```js
// dynatrace.config.js
module.exports = {
  input: {
    instrument: false, // ❌ Zero automático — controle total
  },
  lifecycle: {
    includeUpdate: false,
  },
};
```

**Quando implementar:**

- High-traffic apps (> 1M events/dia) — payload importa
- Compliance strict (LGPD, CCPA) — coletar apenas necessário
- Backend crítico — instrumentar só rotas importantes

**Wrapper dtRum (10 métodos core):**

```js
// src/monitoring/dtRum.js
export const dtRum = {
  init(),                          // Inicializar SDK
  identificarUsuario(userId, tags),  // User identification
  sessao(atributos),                 // Session enrichment
  telaCarregada(nome, startTime, props),  // TTI + page-level metrics
  clique(elemento, contexto),        // User interaction
  withAction(nome, fn),              // Parent action + error handling
  subAcao(parent, nome, fn),         // Child action
  apiCall(nome, fn),                 // HTTP monitoring
  evento(descricao),                 // Lightweight event
  erroBusiness(descricao, codigo),   // Business error
};
```

**Exemplo: Checkout Flow 100% Customizado**

```js
const checkout = async () => {
  await dtRum.withAction("Checkout - Processar", async (parent) => {
    // Validar
    await dtRum.subAcao(parent, "Validação - Endereço", () =>
      validateAddress(formData),
    );

    // Pagar
    const paymentResult = await dtRum.subAcao(
      parent,
      "Pagamento - Processar",
      () => api.processPagamento({ ...formData }),
    );

    if (paymentResult.approved) {
      dtRum.evento("Conversão - Pedido Confirmado");
      dtRum.sessao({ conversao_status: "sucesso" });
    } else {
      dtRum.erroBusiness(
        `Pagamento rejeitado: ${paymentResult.error}`,
        `PAG_${paymentResult.errorCode}`,
      );
    }
  });
};
```

---

## Boas Práticas: O Que Considerar Ao Criar Ações Customizadas

### 1. **Naming Convention**

```
[Categoria] - [Tela/Contexto] - [Ação Específica]

✅ Bom:     "Pagamento - Checkout - Processar Transação"
❌ Ruim:    "process"
❌ Ruim:    "clique em botao"
❌ Ruim:    "update user profile details"
```

### 2. **Granularidade**

```js
// ❌ Muito granular — ruído demais
Object.values(formData).forEach((field) => {
  dtRum.evento(`Campo alterado: ${field}`); // 30+ eventos por sessão
});

// ✅ Perfeito — 1 métrica por jornada
dtRum.withAction("Formulário - Preencher", async (parent) => {
  // Todas as alterações dentro
  await updateForm(formData);
});
```

### 3. **Context Matters**

```js
// ❌ Sem contexto — impossível debugar
dtRum.apiCall("GET Request", () => fetch(url));

// ✅ Com contexto — acionável imediatamente
dtRum.apiCall("API - Buscar Usuário", () => fetch(`/usuarios/${userId}`));
```

### 4. **Error Categorization**

```
Erro TÉCNICO               Erro DE NEGÓCIO
├─ HTTP 500               ├─ Saldo insuficiente
├─ Timeout                ├─ CPF inválido
├─ JSON parse error       ├─ Cupom expirado
└─ Connection lost        └─ Limite de transações
```

**Implementação:**

```js
try {
  const result = await api.processar();
  if (result.status === "ERRO_NEGOCIO") {
    dtRum.erroBusiness(`${result.mensagem}`, result.codigo);
  }
} catch (techError) {
  dtRum.erroBusiness(`Erro técnico: ${techError.message}`, "TECH_ERROR");
}
```

### 5. **TTI (Time to Interactive) é REI (Return on Engineering Investment)**

```
Se TTI de HomeScreen < 500ms:
  • Usuários completam mais ações
  • Conversão aumenta 5-10%
  • Retenção melhora

Portanto: SEMPRE monitore TTI de jornadas críticas
```

---

## Migration Path: Automático → 100% Customizado

```
Mês 1: Automático puro
  ↓ (Coletar dados, entender padrões)

Mês 2: Automático + Jornadas críticas
  ↓ (Adicionar HomeScreen, Checkout, Payment)

Mês 3: Automático + Error handling estratégico
  ↓ (Categorizar erros de negócio)

Mês 4-5: 100% Customizado (opcional)
  ↓ (Se volume/compliance exigir)

Contínuo: Otimizar baseado em alertas + dashboards USQL
```

---

## Próxima Etapa

→ [**3. Enriquecimento de Dados com Session Properties**](3-session-properties.md)

// Para instrumentar seletivamente por arquivo:
instrument: (filename) => filename.includes("checkout"),
},

````

## Monitoramento de Performance do SDK

Ativar logs apenas em desenvolvimento:

```js
// dynatrace.config.js
react: {
  debug: true,  // Ativa verbose logging no Metro
  // Verificar console por mensagens como:
  // "Dynatrace: instrumented function X"
  // "Dynatrace: action X finished"
},
````

---

## Próxima Etapa

→ [3. Enriquecimento de Dados](3-session-properties.md)
