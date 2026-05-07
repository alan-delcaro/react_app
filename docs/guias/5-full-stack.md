# 5. Correlação Full-Stack: Mobile → Backend → Infra

## Arquitetura de Observabilidade

```
┌──────────────────────────────────────────────────────────────────────┐
│  MOBILE (Android/iOS)                                                 │
│  @dynatrace/react-native-plugin v2.337.1                             │
│  beacon → https://bf78240axh.bf.dynatrace.com/mbeacon                │
└───────────────────────────┬──────────────────────────────────────────┘
                            │ HTTP com W3C Trace Context headers
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│  BACKEND (Node.js/Express — Kubernetes k3s)                          │
│  namespace: pontos-club                                               │
│  OneAgent injetado via dynatrace-operator (initContainer)            │
│  ActiveGate: fov31014.live.dynatrace.com                             │
│  NetworkZone: ard-rancher-demo                                       │
└───────────────────────────┬──────────────────────────────────────────┘
                            │ Rastreamento distribuído automático
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│  BANCO DE DADOS (MySQL 8.0 — Kubernetes k3s)                         │
│  OneAgent monitora queries via auto-instrumentação                   │
└──────────────────────────────────────────────────────────────────────┘
```

## Rastreamento Distribuído: Como Funciona

O SDK mobile injeta automaticamente os headers `traceparent` (W3C Trace Context) e `x-dynatrace` em todas as requisições HTTP quando `webRequests` está habilitado (padrão). O OneAgent no backend captura esses headers e liga a chamada à sessão mobile.

**Resultado no Dynatrace:** A sessão mobile mostra o waterfall completo:

```
[Mobile] API - Usuário - Carregar Perfil           180ms
  └─ [Backend] GET /usuarios/user_12345             165ms
       └─ [MySQL] SELECT * FROM usuarios WHERE id   45ms
```

## Investigar um Incidente com o Davis AI

**Fluxo de triagem recomendado:**

1. **Alerta dispara** (ex: aumento de `tti_ms` na Home)
2. Dynatrace → `Problems` → Abrir o problema
3. Davis AI mostra a **baseline** do `tti_ms` e quando começou a degradar
4. `Analyze` → `User sessions` → Filtrar por `action.name = "Home - Tela Carregada"` + janela de tempo do problema
5. No waterfall, identificar qual sub-action degradou:
   - `Tile - Cartão de Pontos` → problema no serviço de usuários
   - `Tile - Transações Recentes` → problema no serviço de transações
6. Clicar na sub-action → `Linked requests` → abrir o trace distribuído
7. Ver no backend qual query SQL ou serviço downstream causou o aumento de latência

## Service Map

`Dynatrace → Observe and Explore → Services → [pontos-club-backend]`

O Service Map mostra:

- Dependências upstream (app mobile como chamador)
- Dependências downstream (MySQL)
- Anomalias automáticas detectadas pelo Davis AI por serviço

## USQL para Dashboards SRE

### Performance por Tela

```sql
SELECT name,
       AVG(longProperties.tti_ms) AS avg_tti,
       percentile(longProperties.tti_ms, 95) AS p95_tti,
       COUNT(*) AS total
FROM useraction
WHERE longProperties.tti_ms IS NOT NULL
GROUP BY name
ORDER BY avg_tti DESC
```

### Distribuição de Usuários por Nível

```sql
SELECT stringProperties.nivel_usuario,
       COUNT(DISTINCT userId) AS usuarios,
       AVG(longProperties.pontos_usuario) AS media_pontos
FROM usersession
WHERE stringProperties.nivel_usuario IS NOT NULL
GROUP BY stringProperties.nivel_usuario
```

### Taxa de Erro em Chamadas de API

```sql
SELECT name,
       COUNT(*) AS total,
       SUM(CASE WHEN stringProperties.status = 'erro' THEN 1 ELSE 0 END) AS erros,
       (SUM(CASE WHEN stringProperties.status = 'erro' THEN 1 ELSE 0 END) * 100.0 / COUNT(*)) AS taxa_erro_pct
FROM useraction
WHERE name LIKE "API - %"
GROUP BY name
HAVING total > 10
ORDER BY taxa_erro_pct DESC
```

### Cold Startup por Versão do App

```sql
SELECT appVersion,
       AVG(longProperties.startup_ms) AS avg_startup,
       percentile(longProperties.startup_ms, 95) AS p95_startup,
       COUNT(*) AS amostras
FROM useraction
WHERE name = 'App - Cold Startup'
GROUP BY appVersion
ORDER BY avg_startup DESC
```

### Sessões com Crash por Segmento

```sql
SELECT stringProperties.segmento,
       osVersion,
       COUNT(*) AS sessoes_com_crash
FROM usersession
WHERE hasCrash = true
  AND stringProperties.segmento IS NOT NULL
GROUP BY stringProperties.segmento, osVersion
ORDER BY sessoes_com_crash DESC
```

## Alertas Recomendados para SRE

### 1. Degradação de Performance da Home

```
Métrica: builtin:apps.mobile.apdex.load
Filtro: action.name = "Home - Tela Carregada"
Baseline: Automática (Davis AI)
Threshold: 20% acima da baseline
Ação: Alerta + Slack notification
```

### 2. Taxa de Erro em APIs

```
Métrica: Custom Event: stringProperties.status = "erro"
Filtro: name LIKE "API - %"
Baseline: 2%
Threshold: > 5%
Ação: Page (Runbook) + Escalação
```

### 3. Crashes Críticos

```
Métrica: usersession.crashGroupId
Filtro: Crash detected
Baseline: Zero
Threshold: > 1 crash novo em 5 min
Ação: Alerta crítico + PagerDuty
```

---

## Próxima Etapa

→ [6. Troubleshooting](6-troubleshooting.md)
