# comunica_cel — plano de implementação open source

> Companheiro de [`contexto.md`](contexto.md) (viabilidade) e [`kit-emergencia.md`](kit-emergencia.md) (o que usar hoje).
> **Decisão de engenharia:** não escrever protocolo próprio. **Fork do `bitchat-android`.** A justificativa está na §1.
>
> **Compilar não é requisito para usar.** Para ter rádio funcionando hoje, o caminho é o APK oficial verificado por hash — ver [`kit-emergencia.md`](kit-emergencia.md). O build descrito aqui serve ao fork, e só.

---

## 1. Por que fork, e não implementação nova

A pergunta "reuso vs. implementação" ficou aberta em `contexto.md` §9. A resposta é fork, por três motivos independentes — qualquer um deles já bastaria:

**1. A licença não oferece atrito.** O bitchat é **domínio público (Unlicense)**. Não há obrigação de atribuição, sem cláusula viral, sem negociação. Pode-se copiar, renomear, modificar e redistribuir à vontade. É o cenário mais permissivo que existe.

**2. A parte difícil já foi exercitada em campo.** O que um projeto novo levaria meses para descobrir, o bitchat já tem funcionando e com parâmetros calibrados: papel duplo GATT central+peripheral, fragmentação em ~469 B, TTL 7 limitado por densidade, *seen-set* LRU de 1000/5 min, *jitter* de relay de 10–220 ms, sessões Noise XX/X, outbox + *courier spray-and-wait* + histórico por gossip. Esses números não são escolha de projeto — são cicatriz de uso real. Reimplementar significa redescobri-los.

**3. O precedente diz onde o risco mora.** O Serval Project tinha arquitetura correta e morreu na camada física: dependia de Wi-Fi Ad-Hoc, que exige root no Android. **Um protocolo novo não reduz esse risco; só reposiciona você no começo da curva.** O código que sobrevive a fabricantes e versões de Android diferentes é o código que já rodou em fabricantes e versões de Android diferentes.

**Corolário:** um simulador do protocolo em host passaria nos próprios testes e **não produziria evidência alguma** sobre o que decide o resultado. Todos os modos de falha do levantamento vivem exatamente na camada que um simulador finge: o `192.168.49.1` compartilhado entre Group Owners, os 19–20% de perda do SWARNET em três grupos, o Ad-Hoc que o Android desabilitou. Validação, aqui, é aparelho na mão.

---

## 2. Escopo do fork

### Manter intacto (é o ativo que se está comprando)

- Transporte BLE e o papel duplo central/peripheral
- Fragmentação, remontagem, TTL adaptativo, deduplicação por *seen-set*, *jitter* de relay
- Sessões Noise XX/X, identidade Curve25519/Ed25519
- Camada de *store-and-forward* (outbox, *courier*, gossip de histórico público)
- Serviço em primeiro plano e o tratamento das restrições de execução em segundo plano do Android

Regra: **mexer aqui só com medição em aparelho real para justificar.**

### Mudar

| Área | Mudança | Por quê |
|---|---|---|
| **Idioma** | UI e textos em português do Brasil | Público-alvo é defesa civil / comunidade local |
| **Enquadramento** | Coordenação de emergência em vez de chat "IRC vibes" | Quem abre o app na crise precisa de "pedir ajuda / informar situação / ponto de encontro", não de `/join` |
| **Nostr + Tor** | Desligar por padrão com chave de build | Exige internet, violando R1. Reduz superfície de ataque e tamanho do APK |
| **Canal do grupo** | Convenção fixa configurável no build/primeira execução | Elimina o passo 5 do kit como ponto de falha humana |
| **Onboarding** | Fluxo que força as permissões e a isenção de bateria antes de liberar a tela principal | "Instalado sem permissão não está instalado" é a falha mais comum do kit |
| **Diagnóstico** | Tela de estado da malha: pares visíveis, saltos, fila de saída, pendentes | Sem isso não há como ensaiar nem depurar em campo. Verificar antes o que o pacote `features/` já expõe |

### Já existe — não construir de novo

A leitura do repositório (2026-11-14) encontrou pronto muito do que eu havia listado como trabalho a fazer. **Verificar antes de especificar qualquer feature:**

| Achado | Onde | Consequência |
|---|---|---|
| **PTT ao vivo**, half-duplex, melhor-esforço, multi-salto, interoperável com iOS/Wear | `docs/LIVE_VOICE_V1.md` | Derruba a premissa de que só havia nota de voz. Ver a correção em `contexto.md` §2 |
| **Compartilhamento do APK por hotspot Wi-Fi P2P com QR code** | `app/.../hotspot/` (`ApkWebServer.kt`, `QrCodeGenerator.kt`) | **R3 da proposta original já está resolvido nativamente.** Briar deixa de ser necessário para propagar o app |
| **Transporte Wi-Fi Aware** implementado | `app/.../wifi-aware/` | A "promoção para Wi-Fi sob demanda" que eu propus em `contexto.md` §5 já tem implementação |
| **Mesh Lab** — harness de validação em aparelhos físicos | `tools/release_gate/mesh_lab.py`, `docs/release-gate-runbook.md` | A Fase 3 não precisa de ferramenta nova. Usar esta |
| Matriz de teste por transporte e aparelho | `docs/device-transport-test-matrix.md` | Ponto de partida para o teste de heterogeneidade |
| Limites de payload, roteamento por origem, gargalos medidos | `docs/PACKET_PAYLOAD_LIMITS.md`, `SOURCE_ROUTING.md`, `PERFORMANCE_BOTTLENECKS.md` | Especificação já escrita; ler antes de medir |

**Lição:** o fork tem escopo menor do que este plano supunha. Boa parte da §2 "Mudar" vira idioma, enquadramento e padrões — não engenharia de rede.

### Não fazer

- **Chamada full-duplex com garantia de entrega.** Esse é o limite real, e é de banda: `contexto.md` §2 traz o orçamento. PTT ao vivo já existe e é o teto do que o enlace sustenta.
- **Protocolo incompatível.** Divergir do formato de pacote quebra interoperabilidade com a base instalada — que é justamente o que dá valor à malha. O `LIVE_VOICE_V1.md` é explícito: os valores de tipo são canônicos e há *golden vectors* testados contra o iOS. Quanto mais nós, melhor; incompatibilizar é jogar contra o próprio projeto.
- **Mexer no núcleo sem Mesh Lab.** O `AGENTS.md` do upstream exige validação em aparelho físico para qualquer mudança em descoberta, roteamento, transporte, Noise/cripto, identidade, energia do serviço em primeiro plano, mensageria, transferências, pacotes ou fragmentação. É a regra certa; herde-a.

---

## 3. Pré-requisitos de build — RESOLVIDO, upstream compilado

> **`BUILD SUCCESSFUL` em 3m54s**, 38 tasks, em 2026-11-14. Saída em `app/build/outputs/apk/debug/`:
> `app-arm64-v8a-debug.apk` (44 MB) · `app-armeabi-v7a-debug.apk` (41 MB) · `app-x86_64-debug.apk` (45 MB) · `app-x86-debug.apk` (45 MB) · `app-universal-debug.apk` (74 MB)
> Pacote `com.bitchat.droid`, versionName **2.2.0**, versionCode 42, compileSdk/targetSdk 37, minSdk 26.


Levantado e **destravado** em 2026-11-14 nesta máquina. O repositório pede **JDK 21** (`.java-version` = `21.0.11`, confirmado pelo `AGENTS.md`) — não 25, como este plano supunha antes do clone.

| Item | Exigido pelo repo | Estado |
|---|---|---|
| **JDK** | 21 | ✅ **Temurin 21.0.13** instalado em `~/.local/share/jdks/`, sem `sudo`. O pacote do sistema era só JRE |
| Gradle | **9.6.1** (via wrapper, com SHA-256 validado) | ✅ Baixado pelo wrapper. O 8.10.2 que havia em cache não serve |
| AGP / Kotlin | 9.3.1 / 2.4.10 | ✅ Resolvidos pelo Gradle |
| compileSdk / targetSdk | **37** | ✅ `android-37.0` presente no SDK |
| minSdk | **26** (Android 8.0) | ✅ Cobre a base de 2020+ |
| Android SDK | — | ✅ `local.properties` aponta para o SDK local (está no `.gitignore`) |
| `adb` | — | ✅ em `platform-tools/` |
| Aparelho físico | para Mesh Lab e testes instrumentados | ❌ **Pendência** — nenhum conectado |

Ambiente para reproduzir, sem `sudo`:

```bash
export JAVA_HOME="$HOME/.local/share/jdks/jdk-21.0.13+9"
export ANDROID_HOME="$HOME/Android/Sdk"
export PATH="$ANDROID_HOME/platform-tools:$PATH"
```

> O emulador do SDK serve para fumaça de UI e **não valida nada de BLE, Wi-Fi Aware ou malha.** Para isso, aparelho na mão — e é por isso que o item do aparelho físico é a única pendência que importa.

### Regras do upstream a herdar

O `AGENTS.md` do repositório impõe disciplina que vale adotar no fork, não contornar:

- **Privacidade por omissão:** nada de caminho absoluto, usuário, hostname, serial de aparelho, ID de par, log cru ou coordenada real em commit, teste, *fixture*, screenshot, issue ou PR. Dado de teste é sintético.
- **Geolocalização é dado sensível:** nunca ler, derivar ou registrar localização real de aparelho, emulador ou conta — geohash de teste é sintético.
- **Evidência de Mesh Lab e logcat ficam locais.**
- `app/` é a fonte de verdade do código de malha/protocolo compartilhado; `wear/` recebe por `syncSharedAppSources`. **Nunca** copiar Kotlin compartilhado para `wear/src/`.
- Mudança de protocolo ou segurança é *fail-closed*, mantém compatibilidade entre clientes e **atualiza especificação e golden vectors**.

## 4. Caminho de execução

### Fase 0 — Reproduzir o upstream antes de tocar em nada

```bash
git clone --depth 1 https://github.com/permissionlesstech/bitchat-android
cd bitchat-android
echo "sdk.dir=$ANDROID_HOME" > local.properties
./gradlew :app:assembleDebug
```

Status: **clone feito, toolchain resolvida, build executado** (ver §3). O que falta é o portão de verdade — instalar o APK em **dois aparelhos reais** e trocar uma mensagem:

```bash
adb devices                                   # confirmar aparelho autorizado
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

Vale rodar também a suíte que o CI roda, que não precisa de aparelho:

```bash
./gradlew testDebugUnitTest lintDebug
```

**Portão:** não se altera uma linha antes de ter o upstream compilando **e funcionando em aparelho**. Build que compila mas nunca trocou uma mensagem não é base — é chute.

### Fase 1 — Medir bateria (antes de qualquer feature)

A maior lacuna do levantamento, e provavelmente a métrica que decide a viabilidade real. Com o upstream limpo, em **3–4 aparelhos de fabricantes diferentes**, 24 h, app em primeiro plano de malha:

```bash
adb shell dumpsys batterystats --reset
# ... 24 h de operação ...
adb shell dumpsys batterystats > bateria_<fabricante>_<data>.txt
```

Decisão que sai daqui: se o consumo inviabilizar operação contínua, a arquitetura muda de "sempre ligado" para **janela de rádio** (ciclo de trabalho combinado), e isso é mudança de produto, não de código. Melhor descobrir na Fase 1 do que depois da UI pronta.

### Fase 2 — Fork mínimo viável

Nesta ordem, cada etapa com APK instalado e testado em aparelho:

1. Renomear pacote/aplicação, trocar identidade visual
2. Traduzir a UI para português
3. Desligar Nostr/Tor por chave de build
4. Tela de diagnóstico de malha
5. Onboarding que força permissões e isenção de bateria
6. Reenquadrar a UI para coordenação de emergência

### Fase 3 — Validar em campo, com heterogeneidade

**Use o Mesh Lab do upstream, não escreva harness novo:**

```bash
python3 tools/release_gate/mesh_lab.py setup \
  --serial-a <aparelho-a> --serial-b <aparelho-b> --apk <apk-debug>
python3 tools/release_gate/mesh_lab.py scenario all \
  --serial-a <aparelho-a> --serial-b <aparelho-b> --out <dir-local>
```

Evidência bruta e logcat **ficam locais** — exigência do `AGENTS.md`. Comece pela `docs/device-transport-test-matrix.md`.

O que medir, e que a literatura **não** cobre — o SWARNET usou cinco Galaxy A10s **idênticos**, indoor, estáticos, medindo só vazão e perda:

- Pelo menos **3 fabricantes diferentes** (é onde o Serval morreu)
- **Multi-salto real:** dois nós fora de alcance direto, um terceiro no meio
- **Mobilidade:** nós andando, entrando e saindo de alcance
- **Alcance por salto** medido em ambiente real — aberto e com paredes
- **Latência** de entrega ponta a ponta, por número de saltos
- **Qualidade do PTT ao vivo por salto** — onde o áudio começa a cortar, dado o teto de 6.000 B/s por *burst* e o TTL limitado a 5 em malha densa

Cada número desses preenche uma linha da tabela de lacunas de `contexto.md` §8. É a contribuição original que o projeto pode dar, porque ninguém publicou.

### Fase 4 — Distribuição

- **Verificação de desenvolvedor do Google:** item de prazo externo, não negociável. Enforcement começou em **30/09/2026** em quatro países, expansão global com EUA em **2027**. App não verificado cai no fluxo de **24 h de atraso**. **Confirmar a posição do Brasil no cronograma.**
- **F-Droid:** o upstream já adicionou metadados de F-Droid — caminho natural, e sem a exigência de verificação da Play.
- **APK + `SHA256SUMS` assinados**, com build reproduzível (o upstream suporta rebuild byte a byte em container fixado). Preserve isso: é o que permite distribuir por pendrive sem pedir confiança cega.
- **Sideload sem internet:** replicar a abordagem do Briar — hotspot local servindo o próprio APK.

---

## 4-A. Alcance: o maior ganho técnico disponível no fork

### LE Coded PHY — não usado, e é a maior alavanca

Verificado no fonte (2026-11-14): o app usa **apenas advertising legado**. Não há `AdvertisingSet`, `setPrimaryPhy`, `setSecondaryPhy` nem `isLeCodedPhySupported` em lugar algum — portanto roda **só em 1M PHY**.

O **LE Coded PHY com S=8** do Bluetooth 5 entrega **≈ +12 dB de orçamento de enlace, equivalente a ~4× o alcance**, ao custo de **8× o tempo no ar** ([TI](https://software-dl.ti.com/lprf/simplelink_cc26x2_latest/docs/ble5stack/ble_user_guide/html/ble-stack-5.x/phy-coded.html), [Hubble](https://hubble.com/community/guides/how-ble-coded-phy-s-8-achieves-4x-range-at-the-cost-of-8x-airtime/)). A API existe desde **API 26** — o mesmo `minSdk` do projeto.

**Por que isso encaixa exatamente neste caso de uso:** o preço do Coded PHY é vazão, e o requisito aceito é **texto e voz gravada**. Mensagem e arquivo toleram 8× de tempo no ar; não há prazo. **É a troca certa — mas só para esse tráfego.**

### O que não cabe junto

- **PTT ao vivo é incompatível** com Coded PHY. O `LIVE_VOICE_V1` dimensiona cada pacote em 210 bytes para caber em 64 ms de áudio; multiplicar o tempo no ar por 8 destrói esse orçamento. O fork teria de **escolher o PHY por tipo de tráfego**, ou expor "modo longo alcance" que desliga o ao vivo.
- **Interoperabilidade:** quem anuncia só em Coded PHY fica **invisível** para scanner 1M. Exigiria anunciar nos dois, ou negociar — e romper com a base instalada custa justamente o que dá valor à malha.
- **Suporte desigual:** `isLeCodedPhySupported()` precisa ser consultado em tempo de execução. Nem todo aparelho de 2020 tem. Precisa de degradação graciosa.
- **Bateria:** 8× de tempo no ar é mais rádio ligado. Tensão direta com a Fase 1. **Medir, não supor.**

### Modo emergência: fixar `PERFORMANCE`

Mudança pequena e de ganho imediato. Hoje o `PowerManager` só escolhe `PERFORMANCE` (limiar −95 dBm, TX HIGH) **quando carregando**; sem carga cai para −85, −75 e −65 conforme a bateria. Um modo explícito que **fixe `PERFORMANCE` independente da banda de bateria** entrega ao usuário a escolha que hoje o app faz por ele: **alcance agora em troca de autonomia depois**. Numa emergência essa é decisão de quem está lá, não do algoritmo.

### Não mexer

- **Aumentar o TTL além de 7.** O custo de flood cresce rápido, e o valor é canônico no protocolo — divergir quebra interoperabilidade sem ganho proporcional.

## 5. Riscos

| Risco | Severidade | Mitigação |
|---|---|---|
| Consumo de bateria inviabiliza uso contínuo | **Alta** | Medir na Fase 1, antes de investir em features. Plano B: janela de rádio |
| Verificação de desenvolvedor bloqueia distribuição | **Alta** | Verificar cedo; F-Droid como rota paralela; APK verificável por hash |
| Divergência entre fabricantes na pilha BLE | **Média-alta** | Fase 3 com 3+ fabricantes. É a lição do Serval |
| Fork divergir do upstream e perder correções | **Média** | Manter o núcleo de protocolo intacto; isolar mudanças em camada de UI/configuração; acompanhar releases |
| Expectativa de "rádio ao vivo" no público | **Média** | Comunicação explícita, desde a tela de abertura. `kit-emergencia.md` já traz o texto |
| Ausência de auditoria de segurança | **Média** | Não prometer proteção contra adversário capaz até haver auditoria |

---

## 6. Próxima ação concreta

O bloqueador de toolchain caiu (§3). A próxima ação **não é código** — é hardware:

1. **Conectar dois aparelhos Android** (fabricantes diferentes, de preferência), com depuração USB ativa.
2. `adb install -r app/build/outputs/apk/debug/app-debug.apk` nos dois.
3. Modo avião + Bluetooth ligado. Trocar texto, nota de voz e PTT ao vivo.
4. Com isso de pé, **Fase 1 (bateria)** — e é ela, não a UI, que define se o projeto tem futuro na forma proposta.

Enquanto não houver aparelho, o que rende é leitura: `docs/PERFORMANCE_BOTTLENECKS.md`, `docs/PACKET_PAYLOAD_LIMITS.md` e `docs/release-gate-runbook.md`.
