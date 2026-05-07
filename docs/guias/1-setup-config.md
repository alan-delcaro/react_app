# 1. Setup Inicial e Configuração

## Instalação do Plugin

```bash
npm install @dynatrace/react-native-plugin
npx @dynatrace/react-native-plugin configure
```

O comando `configure` injeta automaticamente as dependências nativas no `android/build.gradle` e no `ios/Podfile`. Após isso:

```bash
cd ios && pod install
npm run android   # ou npm run ios
```

## `metro.config.js` — Transformer e Reporter

O Metro precisa de dois hooks para que o Dynatrace funcione em build time:

```js
// metro.config.js
module.exports = {
  transformer: {
    // Instrumenta automaticamente o código JS durante o bundle
    babelTransformerPath:
      require.resolve("@dynatrace/react-native-plugin/lib/dynatrace-transformer"),
  },
  // Reporter para logs de build do SDK durante desenvolvimento
  reporter: require("@dynatrace/react-native-plugin/lib/dynatrace-reporter"),
};
```

> ⚠️ **SRE:** Sem o `babelTransformerPath`, as diretivas do `dynatrace.config.js` (como `input.instrument`) são ignoradas silenciosamente. O app compila, mas nenhuma instrumentação automática é aplicada.

## `dynatrace.config.js` — Configuração Central

Este arquivo é processado em **build time** pelo transformer. Mudanças aqui exigem rebuild nativo (`npm run android`).

```js
module.exports = {
  react: {
    debug: false, // true apenas em desenvolvimento local

    lifecycle: {
      // false: suprime ações como "HomeScreen Update" — sem valor para SRE
      includeUpdate: false,
      instrument: (filename) => false,
    },

    input: {
      // false: suprime "Touch on Button" genéricos
      // Todas as ações são instrumentadas manualmente via dtRum.clique()
      instrument: (filename) => false,
    },
  },

  android: {
    config: `
      dynatrace {
        configurations {
          defaultConfig {
            autoStart {
              applicationId 'baebdda9-0c35-4e0f-a32e-f501e0e62225'
              beaconUrl 'https://bf78240axh.bf.dynatrace.com/mbeacon'
            }
            userOptIn true
            agentBehavior.startupLoadBalancing true
          }
        }
      }
    `,
  },

  ios: {
    config: `
      <key>DTXApplicationID</key>
      <string>baebdda9-0c35-4e0f-a32e-f501e0e62225</string>
      <key>DTXBeaconURL</key>
      <string>https://bf78240axh.bf.dynatrace.com/mbeacon</string>
      <key>DTXLogLevel</key>
      <string>WARNING</string>
      <key>DTXUserOptIn</key>
      <true/>
      <key>DTXStartupLoadBalancing</key>
      <true/>
      <key>DTXInstrumentViewControllerLoading</key>
      <false/>
    `,
  },
};
```

## O Que o SDK Captura Automaticamente

| Categoria                    | Capturado       | Observação                                        |
| ---------------------------- | --------------- | ------------------------------------------------- |
| **Sessões**                  | ✅ Sim          | Início/fim, duração, dispositivo, OS, geo         |
| **Crashes nativos**          | ✅ Sim          | Stack trace nativo Android/iOS                    |
| **JS Exceptions**            | ✅ Sim          | Requer `userOptIn: true` configurado              |
| **Requisições HTTP**         | ✅ Sim          | Via interceptor nativo (não via Axios)            |
| **Cold/Warm Startup**        | ✅ Sim          | Medido desde o carregamento nativo                |
| **Taps/Gestures**            | ❌ Desabilitado | `input.instrument: false` — instrumentação manual |
| **Lifecycle de componentes** | ❌ Desabilitado | `lifecycle.instrument: false` — reduz ruído       |
| **Localização**              | ✅ Sim          | Baseada no IP, não GPS                            |

> **Por que desabilitamos `input.instrument`?**  
> Com a instrumentação automática ativa, cada `TouchableOpacity` sem `accessibilityLabel` gera uma ação chamada "Touch on Button" — completamente inútil para triagem de incidentes. A estratégia deste projeto é instrumentação **explícita e semântica** via wrapper `dtRum`.

## Inicialização no App

```jsx
// App.js ou _layout.tsx
import { useEffect } from "react";
import dtRum from "./src/monitoring/dtRum";

export default function App() {
  useEffect(() => {
    dtRum.init(); // Configurar privacidade e crash reporting

    console.log("✅ App Initialized — Dynatrace RUM ativo");
  }, []);

  return (
    // ... rest of app
  );
}
```

## Próxima Etapa

→ [2. Otimização da Captura Automática](2-otimizacao.md)
