# 2. Otimização da Captura Automática

## A Decisão: Automático vs. Manual

O SDK oferece dois modos de captura de interações:

```
AUTOMÁTICO → rápido de configurar, nomes genéricos, muito ruído
MANUAL     → trabalho extra, nomes semânticos, dados acionáveis para SRE
```

Este projeto optou por **100% manual**. A tabela abaixo mostra a diferença prática:

| Modo                              | Nome da ação no Dynatrace                     | Utilidade para SRE                  |
| --------------------------------- | --------------------------------------------- | ----------------------------------- |
| Automático                        | `Touch on Button`                             | ❌ Impossível distinguir qual botão |
| Automático + `accessibilityLabel` | `Touch on Ir para Prêmios`                    | ⚠️ Melhor, mas sem contexto         |
| Manual (`dtRum.clique`)           | `Clique - Home - Acessar Catálogo de Prêmios` | ✅ Acionável diretamente            |

## Quando Usar `input.instrument: true` (Seletivo)

Se preferir instrumentação automática em partes do código:

```js
// dynatrace.config.js
input: {
  // Instrumenta apenas a pasta de checkout — fluxo crítico
  instrument: (filename) => filename.includes("checkout"),
},
```

Combine isso com `accessibilityLabel` nos componentes para nomes legíveis:

```jsx
<TouchableOpacity
  accessibilityLabel="Confirmar Pagamento"
  onPress={handlePagamento}
>
```

## Sub-Actions para Fluxos Compostos

Quando uma ação do usuário dispara múltiplas operações paralelas, use `createSubAction` para criar um **waterfall** visível no Dynatrace:

```
Home - Carregar Dashboard        30ms   ← ação pai (withAction)
  ├─ Tile - Cartão de Pontos     12ms   ← subAcao → api.getUsuario()
  └─ Tile - Transações Recentes  18ms   ← subAcao → api.getTransacoes()
```

**Implementação no projeto:**

```js
// src/screens/HomeScreen.js
const carregarDados = useCallback(async () => {
  await dtRum.withAction("Home - Carregar Dashboard", async (parentAction) => {
    const [usuario, txns] = await Promise.all([
      dtRum.subAcao(parentAction, "Tile - Cartão de Pontos", () =>
        api.getUsuario(),
      ),
      dtRum.subAcao(parentAction, "Tile - Transações Recentes", () =>
        api.getTransacoes(),
      ),
    ]);
    setUserData(usuario);
    setTransacoes(txns.slice(0, 3));
  });
}, []);
```

**Valor para SRE:** Se o TTI da Home aumentar, o waterfall mostra imediatamente qual sub-action degradou — `getUsuario()` ou `getTransacoes()` — sem precisar acessar logs.

## Medição de Time to Interactive (TTI)

```js
// Topo do componente — antes de qualquer render condicional
const screenStartTime = useRef(Date.now());

// Após todos os dados carregarem
dtRum.telaCarregada("Home", screenStartTime.current, {
  total_promocoes: promocoes.length,
  total_transacoes: transacoes.length,
  nivel_usuario: userData.nivel,
});
```

O TTI é reportado como `tti_ms` na ação `"Home - Tela Carregada"`. Para criar um alerta de SLO no Dynatrace:

> `Frontend → [App] → Performance → Create SLO`  
> Métrica: `builtin:apps.mobile.apdex.load`  
> Filtro: `action.name = "Home - Tela Carregada"`

## Otimização: `lifecycleUpdate` vs. `instrument`

```js
// Para aplicações grandes, considere:
lifecycle: {
  // Se false: suprime "HomeScreen Update" e similares
  // Se true: captura lifecycle do React Native — pode gerar ruído
  includeUpdate: false,

  // Para instrumentar seletivamente por arquivo:
  instrument: (filename) => filename.includes("checkout"),
},
```

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
```

---

## Próxima Etapa

→ [3. Enriquecimento de Dados](3-session-properties.md)
