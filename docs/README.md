# GitHub Pages Setup

Esta documentação está configurada para ser servida automaticamente via GitHub Pages.

## Como Ativar

1. **No repositório `alan-delcaro/react_app`:**
   - Vá para `Settings` → `Pages`
   - Em "Source", selecione `Deploy from a branch`
   - Branch: `main`
   - Folder: `/docs`
   - Clique em `Save`

2. **Aguarde ~1-2 minutos**
   - GitHub Pages começará a construir o site automaticamente
   - Uma ação será executada em `Actions` → `pages build and deployment`
   - Após sucesso, o site estará em: `https://alan-delcaro.github.io/react_app/`

## Estrutura de Arquivos

```
docs/
├── _config.yml              # Configuração Jekyll
├── index.md                 # Homepage (sumário)
└── guias/
    ├── 1-setup-config.md
    ├── 2-otimizacao.md
    ├── 3-session-properties.md
    ├── 4-padroes-sre.md
    ├── 5-full-stack.md
    └── 6-troubleshooting.md
```

## Tema Usado

- **Jekyll Theme:** `jekyll-theme-minimal` (definido em `_config.yml`)
- **Markdown:** Kramdown com syntax highlighting via Rouge

## Como Editar

1. Faça mudanças nos arquivos `.md` dentro de `docs/`
2. Commit e push para `main`
3. GitHub Pages reconstrói automaticamente em ~1-2 minutos
4. Site atualiza em produção

```bash
git add docs/
git commit -m "docs: update troubleshooting section"
git push origin main
```

## URLs das Seções

- **Home:** `/react_app/`
- **Setup:** `/react_app/guias/1-setup-config`
- **Otimização:** `/react_app/guias/2-otimizacao`
- **Session Properties:** `/react_app/guias/3-session-properties`
- **Padrões SRE:** `/react_app/guias/4-padroes-sre`
- **Full-Stack:** `/react_app/guias/5-full-stack`
- **Troubleshooting:** `/react_app/guias/6-troubleshooting`

## Recursos Futuros

Considere adicionar:

- **Search:** Algolia para busca rápida
- **Dark mode:** Plugin `jekyll-theme-cayman` ou customização
- **Versioning:** Branch `gh-pages` para múltiplas versões
- **Analytics:** Google Analytics ou Plausible para tracking
- **Comments:** Disqus ou GitHub Issues para feedback

## Problemas Comuns

### Site não atualiza após push

- Verificar `Actions` → `pages build and deployment`
- Se houver erro, corrigir o `.md` (sintaxe Markdown)
- Fazer novo commit/push

### Links quebrados

- Verificar paths relativos (sempre usar `/guias/nome`)
- Links entre seções: `[Link](4-padroes-sre.md)` (relativo)

### Tema não aplica

- Limpar cache: `Settings` → `Pages` → `Change theme` → reselecionar `jekyll-theme-minimal`
- Reconstruir: Fazer novo commit em qualquer arquivo `.md`

---

**Status:** ✅ Ativo no GitHub Pages  
**URL:** https://alan-delcaro.github.io/react_app/  
**Atualizado:** maio de 2026
