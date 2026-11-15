# Kit de Comunicação de Emergência — pronto para uso

> **Verificado em novembro de 2026.** Versões e links envelhecem; reconfirme antes de treinar um grupo.
> Documento companheiro de [`contexto.md`](contexto.md), que traz a análise de viabilidade por trás destas escolhas.

---

## Leia isto antes de qualquer coisa

**1. Isto não substitui o canal oficial.** O **Defesa Civil Alerta**, sobre Cell Broadcast, entrega alerta direto ao celular na área de risco sem app e sem cadastro. É o canal de autoridade para população e funciona enquanto a rede da operadora estiver de pé. Este kit é para o caso oposto: comunicação **entre pessoas**, lateralmente, quando não há rede alguma.

**2. Você tem walkie-talkie, não telefone.** Há três modos, e a diferença importa:

- **Texto** e **nota de voz** (arquivo): confiáveis. Entregues em segundos, minutos ou horas — guardados e repassados até chegar.
- **PTT ao vivo** (aperta e fala): funciona, half-duplex, e **pode cortar**. É melhor-esforço: sem confirmação, sem reenvio. Ao soltar o botão, o app manda automaticamente a nota de voz confiável atrás.
- **Chamada telefônica** (falar junto, com garantia): **não existe.** Não tente.

Mensagem entregue em segundos é sorte; **entregue em minutos é sucesso**. Quem espera a qualidade de uma ligação vai concluir que está quebrado.

**3. Tudo tem de estar pronto ANTES.** Esta é a restrição que organiza o kit inteiro, e a razão é concreta: o Android passou a exigir verificação de identidade do desenvolvedor, e o fluxo para app não verificado impõe **24 horas de atraso** na instalação. Instalar durante o apagão pode simplesmente não funcionar. **Baixar e instalar em plena crise não é plano — é o que falha.**

**4. Bateria é o limite real — e governa o alcance.** Malha BLE exige anunciar e escanear continuamente. Nenhuma fonte pública mede esse consumo em aparelhos reais, então trate como custo alto e desconhecido. Pior: **o app reduz o alcance conforme a bateria cai**, e só usa o modo de maior alcance enquanto está carregando. Energia não é acessório do kit — é o que define até onde você fala. Ver "Como aumentar o alcance".

---

## A escolha: bitchat

**O que é:** malha Bluetooth LE ponto a ponto, sem servidor, sem conta, sem número de telefone, sem *pairing*. Cada aparelho é simultaneamente nó e repetidor; mensagens saltam de aparelho em aparelho até chegar. É a implementação mais madura e melhor documentada da arquitetura descrita na proposta do projeto.

**Por que esta e não outra:**

| Critério | Situação |
|---|---|
| Código aberto | **Domínio público (Unlicense)** — sem atrito de licença para auditar, adaptar ou distribuir |
| Hardware | Só o que o aparelho já tem. Nada de LoRa, nada de acessório |
| Infraestrutura | Nenhuma. Funciona com LTE e Wi-Fi inteiramente fora |
| Android mínimo | **API 26 (Android 8.0)** — cobre com folga a base de 2020+ |
| Multi-salto | Sim, *flood* controlado com TTL adaptado à densidade — inclusive para o PTT ao vivo |
| Offline real | *Store-and-forward*: guarda a mensagem e entrega quando o destinatário aparecer |
| Criptografia | Noise XX/ChaCha20-Poly1305, identidade por chave Curve25519 |
| APK distribuível | Sim, com manifesto SHA-256 e build reproduzível |

**O que ele faz:** texto, **nota de voz**, **PTT ao vivo** (half-duplex, melhor-esforço, com a nota de voz confiável enviada em seguida), canais com senha opcional, canais por *geohash* (área geográfica), favoritos para mensagem privada, comandos estilo IRC (`/join`, `/msg`, `/who`) e **compartilhamento do próprio APK por hotspot com QR code**.

Transportes: **BLE** e **Wi-Fi Aware**, mais um caminho Nostr/Tor que só se usa quando há internet.

**Ressalvas honestas:**
- **Sem auditoria de segurança pública conhecida.** Para uso onde a confidencialidade protege alguém de um adversário capaz, isso é uma lacuna, não um detalhe.
- O app tem caminho **Nostr + Tor** para alcance global quando há internet. Fora do escopo deste kit (e inútil no apagão), mas existe — saiba que está lá.
- O repositório avisa que **já foi alvo de pedidos de remoção** e que, quando releases desaparecem, aparecem espelhos não verificáveis. Daí a conferência de hash abaixo ser passo real, não cerimônia.

**Links oficiais**
- Android — [releases no GitHub](https://github.com/permissionlesstech/bitchat-android/releases) · [Google Play](https://play.google.com/store/apps/details?id=com.bitchat.droid)
- iOS/macOS — [App Store](https://apps.apple.com/us/app/bitchat-mesh/id6748219622)
- Código — [bitchat (iOS/macOS)](https://github.com/permissionlesstech/bitchat) · [bitchat-android](https://github.com/permissionlesstech/bitchat-android) · [whitepaper do protocolo](https://github.com/permissionlesstech/bitchat/blob/main/WHITEPAPER.md)

---

## Preparação — faça agora, com rede funcionando

### Passo 1 — Baixar e guardar o APK

Versão de referência: **v2.2.0** (a mais recente em novembro de 2026). Baixe da página de releases:

| Arquivo | Para quem |
|---|---|
| `bitchat-android-arm64.apk` | Maioria dos celulares modernos |
| `bitchat-android-universal.apk` | Fallback de compatibilidade, se o arm64 não instalar |
| `bitchat-android-x86_64.apk` | Emuladores e aparelhos x86 |
| `bitchat-android-wear.apk` | Wear OS |

Baixe **também** o `BITCHAT_SHA256SUMS`.

### Passo 2 — Conferir o hash

Com o `gh` CLI:

```bash
gh release download v2.2.0 --repo permissionlesstech/bitchat-android --dir release-files
cd release-files && shasum -a 256 -c BITCHAT_SHA256SUMS
```

Sem o `gh` — baixe os arquivos manualmente na mesma pasta e rode:

```bash
sha256sum -c BITCHAT_SHA256SUMS --ignore-missing
```

Só prossiga se a saída disser `OK` para o APK que você vai usar. **Hash que não confere = descarte o arquivo.** O projeto também suporta rebuild byte a byte em container Linux fixado, se você quiser verificação independente de ponta a ponta.

### Passo 3 — Replicar fora da nuvem

Copie o APK verificado **e** o `BITCHAT_SHA256SUMS` para:
- um pendrive ou cartão SD do grupo;
- o armazenamento interno de cada celular participante (não só o instalado — o arquivo, para repassar depois).

Esse é o estoque que permite instalar em quem não se preparou, sem internet.

### Passo 4 — Instalar e **abrir uma vez**

Instalar não basta. Abra o app em cada aparelho e conceda, de fato:

- **Bluetooth / Dispositivos por perto**
- **Localização** — o Android exige para escanear BLE. Não é rastreamento pelo app; é exigência da plataforma.
- **Notificações**
- **Isenção de otimização de bateria** — sem isso o Android mata o serviço de malha em segundo plano

> Um app instalado sem permissão concedida **não está instalado**. Essa é a falha mais comum e a mais silenciosa: só aparece quando você precisa.

### Passo 5 — Combinar o protocolo social, por escrito

Decida e anote **enquanto ainda há LTE**:

- **Nome do canal** do grupo e, se usar, **a senha** (`/join #nome`). Canal combinado na hora da crise é canal que ninguém acha.
- **Apelidos** de cada pessoa, e marcar os contatos essenciais como **favoritos** (favorito tem retenção indefinida de mensagem; par comum, ~12 h).
- **Ponto de encontro físico** de recado — qual lugar alguém passa para repassar mensagem fisicamente se a malha não fechar.
- **Janela de rádio**: todos abrem o app em horário combinado (ex.: a cada hora cheia, por 10 min). Isso poupa bateria e aumenta muito a chance de dois nós se encontrarem.

### Passo 6 — Ensaiar

Com o grupo reunido, **coloque os celulares em modo avião e ligue só o Bluetooth.** Troque texto e uma nota de voz. Depois teste o salto: afaste dois aparelhos até perderem contato direto e ponha um terceiro no meio. Se o recado passar, a malha funciona.

Ensaio não é zelo excessivo: é como você descobre que um aparelho do grupo não suporta algo, **com tempo de resolver**.

---

## Segunda camada: Briar

Opcional, como rede de segurança. **O bitchat já propaga o próprio APK** por hotspot com QR code, então a razão original para o Briar caiu. Resta uma:

**Canal de último recurso por mídia física** — o Briar sincroniza mensagem por **pendrive ou cartão SD** (`Connections → Connect via Removable Drive`). Quando nem Bluetooth fecha, o recado vai a pé, na mão de alguém, dentro de um cartão. O bitchat não tem equivalente.

Também sincroniza por Bluetooth e Wi-Fi, sem servidor central e sem número de telefone. É texto — não tem nota de voz nem malha multi-salto como o bitchat. Daí ser camada complementar.

- [briarproject.org](https://briarproject.org/) · [Google Play](https://play.google.com/store/apps/details?id=org.briarproject.briar.android)

---

## Resumo operacional

| Situação | Ferramenta |
|---|---|
| Alerta oficial de desastre | **Defesa Civil Alerta** (Cell Broadcast) — automático, nada a instalar |
| Recado entre pessoas, sem rede, até poucas centenas de metros por salto | **bitchat** — texto, nota de voz e PTT ao vivo |
| Repassar o app para quem não tem | **bitchat**: hotspot + QR code, nativo. Ou o APK no pendrive |
| Nada fecha por rádio | **Briar** por cartão SD / pendrive, levado fisicamente |
| Alcance de quilômetros, infraestrutura zero | Fora deste kit — exige **hardware** (Meshtastic/LoRa). Ver `contexto.md` §3.2 |

---

## Como aumentar o alcance — sem trocar de aplicativo

Tudo aqui é operacional: vale hoje, com o app como está. A ordem é por ganho.

**1. Mantenha os aparelhos carregando.** É o maior ganho gratuito que existe. O app só usa o modo de maior alcance (`PERFORMANCE`, limiar −95 dBm) **quando o aparelho está carregando**. Sem carregar, ele opera a −85 dBm; abaixo de 20% de bateria cai para −75; abaixo de 10%, para −65 — onde só enxerga quem está quase encostado. **O alcance encolhe conforme a emergência se arrasta.** Power bank não é conforto: é o que mantém o alcance.

**1-A. Energia renovável muda o jogo — compre antes.** O power bank tem fundo; **painel solar e carregador manual (dínamo de manivela) não**. E o efeito não é só manter o aparelho vivo: enquanto houver carga entrando, o app **permanece no modo de maior alcance**. Num apagão prolongado, essa é a diferença entre uma malha que encolhe dia após dia e uma que se mantém.

| Fonte | Serve para | Limites |
|---|---|---|
| **Painel solar dobrável (10–20 W)** | Sustentar um aparelho em operação contínua durante o dia e ainda sobrar para recarregar | Depende de sol; rende pouco em dia nublado; inútil à noite |
| **Carregador manual / dínamo** | Piso garantido: funciona à noite, dentro de casa, com qualquer tempo | Potência baixa — minutos de manivela por minutos de uso. Resgate, não sustentação |
| **Power bank** | Pico e transporte | Acaba, e não volta |

Dois cuidados práticos:

- **Painel no sol, aparelho na sombra.** Celular exposto ao sol direto superaquece, reduz desempenho e desliga. Deixe só o painel no sol, com o cabo chegando ao aparelho abrigado.
- **Carga fraca e intermitente** (nuvem passando) pode fazer o aparelho entrar e sair do estado "carregando", oscilando o modo de alcance. Painel com folga de potência é melhor que painel no limite.

**Combinação mais forte deste kit:** aparelho velho + painel solar + janela = **repetidor permanente e autossustentável** (ver item 4). É infraestrutura de verdade, montada com o que estava na gaveta.

**2. App aberto, tela acesa.** Em segundo plano o app é rebaixado para `POWER_SAVER` mesmo com bateria cheia. Na janela de rádio combinada, deixe o app à frente.

**3. Altura e linha de visada.** 2,4 GHz atravessa mal concreto, metal e corpo humano — telefone no bolso perde muito sinal. Um aparelho **na janela, no parapeito, no andar mais alto** alcança incomparavelmente mais que o mesmo aparelho no bolso no térreo. Janela para janela, atravessando a rua, bate parede contra parede.

**4. Transforme aparelhos velhos em repetidores fixos.** Esta é a melhor ideia do kit. Qualquer Android 8.0+ parado na gaveta, com o app instalado, **ligado na tomada ou num power bank e deixado numa janela**, vira um nó permanente da malha. Ele não precisa de chip, nem de conta, nem de alguém operando. Três aparelhos velhos bem posicionados podem ligar dois grupos que não se enxergavam. É a forma mais barata de construir infraestrutura onde ela não existe.

**5. Planeje a corrente, não a potência.** O alcance total é (alcance por salto) × (até 7 saltos) — **mas só se houver alguém em cada trecho**. Cobertura aqui é propriedade da multidão, não do aplicativo. Na prática: é preciso um aparelho ativo a cada 20–30 m no caminho. Mapeie onde estão as pessoas e posicione os repetidores nos vãos.

**6. Use o mensageiro humano de propósito.** Com *store-and-forward*, quem caminha carrega as mensagens no bolso. Combine que alguém percorra uma rota entre dois grupos de tempos em tempos. Isso converte distância em espera — e geograficamente deixa de haver limite.

## O que este kit não resolve

- **Chamada telefônica.** PTT ao vivo existe e funciona, mas é half-duplex e melhor-esforço: corta, e não avisa que cortou. Não há como falar junto com garantia de entrega. Ver `contexto.md` §2 para o orçamento de banda.
- **Alcance longo.** Cada salto é alcance de Bluetooth. Cobrir um bairro exige densidade de participantes, não potência.
- **Multidão.** Nenhuma fonte pública mede essas malhas além de poucas dezenas de nós.
- **Garantia de entrega.** É melhor-esforço. Nenhuma mensagem é confirmada até aparecer na tela do outro.
- **Adversário capaz.** Sem auditoria de segurança pública, não trate como ferramenta de proteção contra um ator estatal.
