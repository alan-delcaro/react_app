# 4. Padrões e Boas Práticas SRE

## Wrapper Centralizado `dtRum`

Toda a instrumentação do projeto passa pelo arquivo `src/monitoring/dtRum.js`. Isso garante:

- **Consistência:** Um único ponto para mudar a convenção de nomes
- **Segurança:** `leaveAction` sempre chamado no `finally` — sem memory leak de ações abertas
- **Testabilidade:** O SDK é mockado nos testes unitários, o wrapper é testado isoladamente
- **Evolução:** Migrar para nova versão do SDK afeta apenas `dtRum.js`

### Interface Pública do Wrapper

```js
dtRum.init(); // Configurar privacidade — chamar no startup
dtRum.identificarUsuario(id, opts); // Após autenticação
dtRum.sessao(atributos); // Enriquecer sessão com dados de negócio
dtRum.telaCarregada(nome, startTime); // TTI por tela
dtRum.clique(elemento, contexto); // Ação de navegação ou interação
dtRum.withAction(nome, fn); // Ação pai com sub-actions
dtRum.subAcao(pai, nome, fn); // Sub-action dentro de withAction
dtRum.apiCall(nome, fn); // Chamada HTTP instrumentada
dtRum.evento(descricao); // Marco de negócio sem duração
dtRum.erroBusiness(descricao, codigo); // Erro de negócio (ex: saldo insuficiente)
dtRum.encerrarSessao(); // Flush + endSession — chamar no background
```

## Convenção de Nomenclatura

O padrão adotado é: `[Tela] - [O que aconteceu em linguagem de negócio]`

```
✅ "Home - Carregar Dashboard"
✅ "Prêmios - Iniciar Resgate: Voucher Amazon"
✅ "Login - Usuário Identificado"
✅ "API - Transações - Carregar Histórico"
✅ "Session - Atualizar Propriedades"

❌ "loadData"
❌ "Touch on Button"
❌ "handlePress"
❌ "getData()"
```

**Por que isso importa para SRE:**

- Dashboards podem filtrar por prefixo de tela (`action.name LIKE "Home - %"`)
- Alertas de anomalia no Davis AI têm contexto suficiente para auto-diagnóstico
- Em incidentes, a timeline de sessão é legível sem código-fonte

### Tabela de Referência de Prefixos

| Prefixo                        | Quando usar                           | Exemplo                               |
| ------------------------------ | ------------------------------------- | ------------------------------------- |
| `[Tela] - Tela Carregada`      | TTI após render completo              | `Home - Tela Carregada`               |
| `[Tela] - Carregar [Recurso]`  | Fetch de dados via `withAction`       | `Home - Carregar Dashboard`           |
| `Tile - [Nome do Tile]`        | Sub-action de um tile específico      | `Tile - Cartão de Pontos`             |
| `Clique - [Tela] - [Ação]`     | Interação do usuário                  | `Clique - Home - Ver Mais Transações` |
| `API - [Recurso] - [Operação]` | Chamada HTTP individual               | `API - Prêmios - Carregar Catálogo`   |
| `Login - [Evento]`             | Fluxo de autenticação                 | `Login - Usuário Identificado`        |
| `Session - [Evento]`           | Atualização de propriedades de sessão | `Session - Atualizar Propriedades`    |
| `App - [Evento]`               | Eventos de ciclo de vida do app       | `App - Cold Startup`                  |

## Instrumentação de Erros de Negócio

Diferencie **erros técnicos** (exceptions, HTTP 5xx) de **erros de negócio** (saldo insuficiente, produto indisponível):

```js
// Erro técnico — capturado automaticamente pelo SDK
// (HTTP 4xx/5xx são automaticamente marcados como falha no request)

// Erro de negócio — reportado explicitamente
if (usuario.pontos < premio.pontos) {
  dtRum.erroBusiness("Resgate negado: saldo insuficiente", 1001);
  // No Dynatrace: aparece em "Error/Exception" da sessão com errorCode 1001
}
```

## Padrão de Contexto em Actions

```js
// ✅ RECOMENDADO: enviar contexto relevante para triagem
dtRum.clique("Home - Resgatar Prêmio", {
  premio_id: "prm_001",
  pontos_necessarios: 2000,
  pontos_usuario: 15430,
  tentativas_falhas: 2,
});

// ❌ EVITAR: sem contexto, sem valor para SRE
dtRum.clique("Clique");
```

## Testes Unitários do Wrapper

O projeto inclui 27 testes cobrindo o `dtRum` (veja `src/__tests__/dtRum.test.js`):

```bash
npm test   # Jest runners — mock do SDK
```

Benefícios:

- Validar que `leaveAction` é sempre chamado (sem memory leaks)
- Verificar tipos corretos (`reportIntValue` vs. `reportStringValue`)
- Garantir que mudanças futuras não quebrem a interface pública

---

## Próxima Etapa

→ [5. Correlação Full-Stack](5-full-stack.md)
