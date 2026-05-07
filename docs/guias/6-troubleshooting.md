# 6. Guia Rápido de Troubleshooting

## Checklist 1: Dados Não Chegam no Dynatrace

```
[ ] 1. O app foi recompilado após mudanças no dynatrace.config.js?
       → Mudanças no config exigem 'npm run android' ou 'npm run ios'
       → Hot reload NÃO aplica mudanças de build-time

[ ] 2. O beacon URL está acessível pelo dispositivo/emulador?
       → Android Emulator: verificar se 'adb reverse tcp:443 tcp:443' está ativo
       → Testar: curl -I https://bf78240axh.bf.dynatrace.com/mbeacon

[ ] 3. O userOptIn está configurado e a privacidade foi aceita?
       → Verificar se dtRum.init() é chamado antes de qualquer enterAction
       → Verificar se applyUserPrivacyOptions recebeu DataCollectionLevel.UserBehavior

[ ] 4. A sessão foi encerrada antes de consultar no USQL?
       → Minimizar o app (dispara AppState 'background' → dtRum.encerrarSessao())
       → Aguardar 3-5 minutos para o backend processar o beacon

[ ] 5. As session properties estão registradas no tenant?
       → Caminho: Frontend → [App] → ... → Edit → Session and user action properties
       → A chave deve ser EXATAMENTE igual ao código (case-sensitive)
       → A sessão deve ter INICIADO após o registro da propriedade

[ ] 6. O metro.config.js tem o transformer do Dynatrace?
       → Verificar: babelTransformerPath apontando para dynatrace-transformer
       → Sem o transformer, input.instrument é ignorado

[ ] 7. Há erros no console relacionados ao SDK?
       → Ativar: dynatrace.config.js → debug: true (apenas em dev)
       → Procurar: "Dynatrace" nos logs do Metro
```

## Checklist 2: Session Properties Não Aparecem no USQL

```
[ ] 1. A propriedade está registrada no UI do tenant? (veja seção 3)

[ ] 2. O código usa action.reportStringValue("chave", valor)?
       → sendSessionPropertyEvent com chave sem prefixo NÃO funciona
       → A chave não deve ter o prefixo "session_properties."

[ ] 3. A sessão foi encerrada DEPOIS que você registrou a propriedade no UI?
       → Sessões abertas antes do registro não são retroativamente enriquecidas

[ ] 4. A propriedade está marcada como "Session" ou "Session, User action"?
       → "User action" only = aparece apenas na ação, não na sessão
       → Para USQL em usersession, precisa de scope "Session"

[ ] 5. Aguardou o tempo de processamento?
       → Beacons processados em ~1-3 min após envio
       → Sessões longas processadas somente após endSession()
```

## Checklist 3: Rastreamento Distribuído Não Conecta Mobile → Backend

```
[ ] 1. O OneAgent está injetado nos pods do backend?
       → kubectl get pods -n pontos-club -o yaml | grep dynatrace
       → Deve haver initContainer "dynatrace-operator"

[ ] 2. O namespace está com a annotation correta?
       → kubectl get namespace pontos-club --show-labels
       → Deve conter: dynatrace.com/inject=true

[ ] 3. O ActiveGate está Running?
       → kubectl get pods -n dynatrace | grep activegate
       → Status deve ser 1/1 Running

[ ] 4. O backend está retornando o header de resposta do rastreamento?
       → Inspecionar headers HTTP no app: procurar 'x-dynatrace-js-agent'
       → Se ausente, o OneAgent não está interceptando as requisições

[ ] 5. A URL do backend está na whitelist de domínios rastreados?
       → Por padrão, todos os domínios são rastreados
       → Verificar se há configuração de exclusão no dynatrace.config.js
```

## Validação Rápida em Desenvolvimento

```bash
# 1. Verificar se os pods do Dynatrace estão saudáveis
kubectl get pods -n dynatrace

# 2. Forçar envio do beacon (via ADB no emulador Android)
adb shell am force-stop com.pontosclublivelo    # fecha o app → dispara background
sleep 5
adb shell am start com.pontosclublivelo/.MainActivity

# 3. Consultar sessões recentes no tenant (USQL)
# Dynatrace → Observe and Explore → Multidimensional Analysis
SELECT userId, startTime, duration, osVersion, stringProperties.nivel_usuario
FROM usersession
WHERE startTime > now()-10m
ORDER BY startTime DESC
LIMIT 10

# 4. Verificar métricas de beacon no ActiveGate
kubectl logs -n dynatrace -l app.kubernetes.io/name=activegate --tail=50 | grep beacon

# 5. Validar disponibilidade do beacon
curl -I https://bf78240axh.bf.dynatrace.com/mbeacon
```

## Erros Comuns e Soluções

### Erro: "Property X não aparece em USQL"

**Causa provável:** Usando `sendSessionPropertyEvent` com chave sem namespace.

**Solução:**

```js
// ❌ ERRADO
const evento = new SessionPropertyEventData();
evento.addSessionProperty("nivel_usuario", "Gold");

// ✅ CORRETO
const action = Dynatrace.enterAction("Session - Atributos");
action.reportStringValue("nivel_usuario", "Gold");
action.leaveAction();
```

### Erro: "Cold Startup >> 5s"

**Causa provável:** Dynatrace não foi compilado no build — faltou transformer no metro.config.js.

**Solução:**

```js
// metro.config.js
module.exports = {
  transformer: {
    babelTransformerPath:
      require.resolve("@dynatrace/react-native-plugin/lib/dynatrace-transformer"),
  },
  reporter: require("@dynatrace/react-native-plugin/lib/dynatrace-reporter"),
};
```

### Erro: "Action 'Home - Tela Carregada' não aparece"

**Causa provável:** `dtRum.init()` não foi chamado, ou `userOptIn` não está ativo.

**Solução:**

1. Verificar se `dtRum.init()` é chamado no App.js/\_layout.tsx
2. Verificar se `dynatrace.config.js` tem `userOptIn: true` (Android) e `DTXUserOptIn: true` (iOS)
3. Recompilar com `npm run android`

### Erro: "BaseUrl localhost não acessível no emulador"

**Causa provável:** Falta de configuração `adb reverse`.

**Solução:**

```bash
# Android Emulator
adb reverse tcp:30080 tcp:30080
adb reverse tcp:8081 tcp:8081

# Testar
adb shell curl http://localhost:30080/health
```

---

## Suporte e Escalação

| Problema                          | Time            | Ação                                                  |
| --------------------------------- | --------------- | ----------------------------------------------------- |
| Beacon não chega                  | Mobile + SRE    | Verificar conectividade, ativar debug                 |
| Rastreamento distribuído quebrado | Backend + SRE   | Verificar OneAgent, namespace labels                  |
| Query USQL lenta                  | SRE + DBA       | Otimizar scope de tempo, adicionar índices            |
| Davis AI não detecta anomalias    | SRE + Dynatrace | Revisar baselines, aumentar tempo de dados históricos |

---

_Documentação mantida pelo time de Engenharia Mobile. Última atualização: maio de 2026._
