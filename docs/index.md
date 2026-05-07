# Observabilidade Mobile com Dynatrace

**Referência técnica para times de SRE e Engenharia Mobile**

> Capacite seu time a investigar e resolver incidentes mobile em produção com dados de observabilidade coletados desde o cliente, correlacionados com o backend e infraestrutura.

---

## 📚 Documentação

### Primeiros Passos

- [**Setup Inicial e Configuração**](guias/1-setup-config.md)  
  Plugin, metro.config.js, dynatrace.config.js, o que é capturado automaticamente

- [**Otimização da Captura Automática**](guias/2-otimizacao.md)  
  Input vs. Manual, Sub-actions, Time to Interactive (TTI)

### Para SREs e Arquitetos

- [**Enriquecimento de Dados**](guias/3-session-properties.md)  
  Session Properties, Action Properties, User Tags, Identificação de Usuários

- [**Padrões e Boas Práticas**](guias/4-padroes-sre.md)  
  Wrapper dtRum, Convenção de Nomenclatura, Instrumentação de Erros

- [**Correlação Full-Stack**](guias/5-full-stack.md)  
  Rastreamento Distribuído, Service Map, Davis AI, USQL para Dashboards

### Troubleshooting

- [**Guia Rápido de Troubleshooting**](guias/6-troubleshooting.md)  
  Checklists, Validação em Desenvolvimento, Comandos de Debug

---

## 🏗️ Arquitetura do Projeto

```
MOBILE (Android/iOS)
├─ @dynatrace/react-native-plugin v2.337.1
├─ Beacon → https://bf78240axh.bf.dynatrace.com/mbeacon
└─ W3C Trace Context headers em HTTP
    ↓
BACKEND (Node.js/Express — Kubernetes k3s)
├─ namespace: pontos-club
├─ OneAgent via dynatrace-operator
├─ ActiveGate: fov31014.live.dynatrace.com
└─ NetworkZone: ard-rancher-demo
    ↓
DATABASE (MySQL 8.0)
└─ Rastreamento distribuído automático
```

---

## 🎯 Casos de Uso

| Cenário                                             | Solução                                                                       |
| --------------------------------------------------- | ----------------------------------------------------------------------------- |
| **Sessão mobile carrega em 5s, deveria ser 2s**     | TTI por tela + sub-actions mostram qual tile degrada                          |
| **Usuário reclama de crash ao resgatar prêmio**     | Identificar by userId → histórico de sessão → logs correlacionados do backend |
| **Taxa de erro em login aumentou de 2% para 8%**    | USQL com segmentação por `nivel_usuario` e `segmento`                         |
| **Backend respondendo lento, mobile não sente?**    | Rastreamento distribuído mostra a falha ao nível de query SQL                 |
| **Problema apenas em produção iOS, não em Android** | Filtrar por `osVersion` + `appVersion` em USQL                                |

---

## 📋 Quick Reference

### Wrapper `dtRum` — API Centralizada

```javascript
// Inicialização
dtRum.init();

// Autenticação
dtRum.identificarUsuario(userId, { nivel, segmento });

// Enriquecimento
dtRum.sessao({ nivel_usuario: "Gold", pontos_usuario: 15430 });

// Performance
dtRum.telaCarregada("Home", startTime, { total_promocoes: 5 });

// Interações
dtRum.clique("Home - Acessar Prêmios", { premio_id: "prm_001" });

// Fluxos compostos
await dtRum.withAction("Home - Carregar Dashboard", async (parent) => {
  const [user, txns] = await Promise.all([
    dtRum.subAcao(parent, "Tile - Pontos", () => api.getUsuario()),
    dtRum.subAcao(parent, "Tile - Transações", () => api.getTransacoes()),
  ]);
});

// HTTP instrumentada
await dtRum.apiCall("Prêmios - Carregar", () => api.getPremios());

// Erros de negócio
dtRum.erroBusiness("Resgate negado: saldo insuficiente", 1001);

// Ciclo de vida
dtRum.encerrarSessao(); // flushEvents + endSession
```

### USQL Essenciais

**Performance por tela:**

```sql
SELECT name, AVG(longProperties.tti_ms) AS avg_tti, percentile(longProperties.tti_ms, 95) AS p95
FROM useraction
WHERE longProperties.tti_ms IS NOT NULL
GROUP BY name
ORDER BY avg_tti DESC
```

**Usuários por segmento:**

```sql
SELECT stringProperties.nivel_usuario, COUNT(DISTINCT userId) AS usuarios
FROM usersession
WHERE stringProperties.nivel_usuario IS NOT NULL
GROUP BY stringProperties.nivel_usuario
```

**Taxa de erro em APIs:**

```sql
SELECT name, COUNT(*) AS total,
       SUM(CASE WHEN stringProperties.status = 'erro' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS taxa_erro_pct
FROM useraction
WHERE name LIKE "API - %"
GROUP BY name
ORDER BY taxa_erro_pct DESC
```

---

## 🔗 Links Úteis

- [Dynatrace React Native Plugin (npm)](https://www.npmjs.com/package/@dynatrace/react-native-plugin)
- [Dynatrace Mobile SDK API Reference](https://dt-url.net/mobile-sdk-api)
- [USQL Reference](https://www.dynatrace.com/support/help/observe-and-explore/dashboards-and-charts/usql)
- [Dynatrace Operator (Kubernetes)](https://www.dynatrace.com/support/help/setup-and-configuration/setup-on-k8s)

---

## 📝 Sumário Completo

- [1. Setup Inicial e Configuração](guias/1-setup-config.md)
- [2. Otimização da Captura Automática](guias/2-otimizacao.md)
- [3. Enriquecimento de Dados](guias/3-session-properties.md)
- [4. Padrões e Boas Práticas SRE](guias/4-padroes-sre.md)
- [5. Correlação Full-Stack](guias/5-full-stack.md)
- [6. Troubleshooting](guias/6-troubleshooting.md)

---

_Documentação mantida pelo time de Engenharia Mobile. Última atualização: maio de 2026._
