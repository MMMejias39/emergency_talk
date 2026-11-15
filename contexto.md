# comunica_cel — Documento de Contexto

> **Status:** levantamento concluído (prior art + viabilidade de rádio). Encaminhamento em [`kit-emergencia.md`](kit-emergencia.md) e [`plano-fork.md`](plano-fork.md).
> **Base de pesquisa:** buscas e leitura de fontes primárias em **novembro de 2026**. Os prazos regulatórios do Android e o suporte a Wi-Fi Aware no iOS têm validade curta — reconfirmar antes de decisões de arquitetura.

---

## 1. A proposta

> Em caso de pane na rede de telefonia via LTE, que haja um aplicativo que possa ser instalado de smartphone para smartphone e que consiga funcionar como **rádio entre aparelhos**, sem necessidade de LTE nem de Wi-Fi central — usando **somente os recursos já existentes na maioria dos smartphones de 2020 em diante**.

Três restrições derivam dessa frase, e é por elas que todo o resto do documento se organiza:

| # | Restrição | Consequência técnica |
|---|---|---|
| R1 | Sem infraestrutura (LTE, AP Wi-Fi, servidor) | Descoberta e roteamento precisam ser *zero-config*, feitos entre pares |
| R2 | Só hardware já presente em aparelhos 2020+ | Elimina LoRa, RNode e qualquer acessório externo |
| R3 | Instalável de aparelho para aparelho | O próprio APK precisa trafegar sem internet |

**Critério de aceite (confirmado pelo proponente em 2026-11-14):** simular um rádio **entre smartphones**, sem depender de LTE nem de rede Wi-Fi funcional. Não se exige qualidade de ligação telefônica.

> **Desambiguação necessária — "Wi-Fi" aparece nos dois lados da restrição.**
> R1 proíbe **rede** Wi-Fi: roteador, ponto de acesso, infraestrutura. Não proíbe o **chip** Wi-Fi.
> **Wi-Fi Direct** e **Wi-Fi Aware** usam o rádio Wi-Fi para falar **direto com o aparelho ao lado**, sem AP, sem DHCP, sem rede. São rádio ponto a ponto, tanto quanto o Bluetooth — e portanto **permitidos** por R1. Confundir os dois descartaria, sem motivo, justamente o transporte de maior banda disponível.

---

## 2. Conclusão principal (lida antes do resto)

> **Correção de 2026-11-14.** A primeira versão desta seção afirmava que voz ao vivo não era viável nessas restrições. **Isso estava errado.** A leitura do código-fonte do bitchat-android mostrou um protocolo de **PTT ao vivo em produção**, interoperável entre Android, Wear OS e iOS ([`docs/LIVE_VOICE_V1.md`](https://github.com/permissionlesstech/bitchat-android/blob/main/docs/LIVE_VOICE_V1.md)). O texto abaixo reflete a evidência primária.

**O que é construível hoje, em hardware de prateleira:**

1. Mensagens de **texto** e **nota de voz** (arquivo), entregues em malha com *store-and-forward*, tolerando atraso de segundos a horas.
2. **PTT ao vivo** — half-duplex, melhor-esforço, com perda tolerada — atravessando **múltiplos saltos**.

**O que não é construível:** **chamada full-duplex com garantia de entrega**, no sentido telefônico. Não há confirmação, não há retransmissão, e o áudio ao vivo é descartável por projeto.

### Por que os dois convivem

O número que eu havia usado — **≈137 kbps**, de 66,67 kbps de quadro de áudio mais *overhead* de L2CAP/ATT ([ST, UM2382](https://www.st.com/resource/en/user_manual/um2382-getting-started-with-the-stm32cube-function-pack-for-fullduplex-voice-streaming-over-bluetooth-low-energy-using-opus-compression-stmicroelectronics.pdf)) — descreve **streaming full-duplex contínuo, grau telefonia**. É o orçamento de um problema diferente. O PTT ao vivo do bitchat resolve outro, e cada decisão de projeto dele é uma renúncia que compra banda:

| Decisão | O que ela compra |
|---|---|
| **Half-duplex** (aperta e fala) | Um falante por vez, não dois fluxos simultâneos |
| **AAC-LC 16 kHz mono**, 1024 amostras/quadro = **64 ms** de áudio | Taxa baixa, quadro curto |
| Pacote inteiro limitado a **210 bytes** — quadro que não cabe é **descartado, não fragmentado** | Evita por construção o custo de fragmentação e remontagem que mata a latência |
| **Sem ACK, sem retransmissão**, fora do gossip sync | Nada de fila de reenvio competindo por banda |
| Teto de recepção: **6.000 bytes/s** por *burst* (≈48 kbps), 384 KiB por *burst* | Limite explícito, uma ordem de magnitude abaixo dos 137 kbps |
| Receptor **pula lacuna** após 550 ms e finaliza parcial em 3 s | Perda é tratada como normal, não como erro |
| Ao soltar o botão, envia a **nota de voz M4A confiável** pelo caminho normal | O ao vivo é o rascunho; o arquivo é o registro |

Relay ao vivo usa o caminho normal da malha, com *jitter* de 8–25 ms e TTL 7 — limitado a 5 em malha com mais de seis pares. Em malha pública exige assinatura Ed25519 de par já conhecido; em privado, sessão Noise estabelecida.

**A leitura correta, portanto:** o enlace não sustenta uma *ligação*. Sustenta um **walkie-talkie**: fala curta, half-duplex, que pode cortar — com o arquivo chegando atrás como garantia. Isso é mais perto da proposta original ("funcionar como rádio entre aparelhos") do que eu havia concluído, e **R2 não precisa ceder**.

O que permanece válido da análise original: em ponto a ponto otimizado o Android entrega 700–1400 kbps via GATT ([Argenox](https://argenox.com/blog/bluetooth-le-throughput-max-performance), [Novel Bits](https://novelbits.io/bluetooth-5-speed-maximum-throughput/)), e essa folga encolhe em malha por causa de fragmentação, *jitter* e flood. A conclusão a tirar disso não é "voz é impossível" — é que **voz só passa se o projeto couber dentro do orçamento, e o bitchat mostra qual é o tamanho da caixa.**

---

## 3. Estado da arte — o que já existe

### 3.1 Compatíveis com as restrições (só hardware de smartphone)

**bitchat** — malha BLE ad-hoc, sem contas, sem *pairing*, sem infraestrutura. É a referência arquitetural mais próxima e mais bem documentada ([repo](https://github.com/permissionlesstech/bitchat), [whitepaper](https://github.com/permissionlesstech/bitchat/blob/main/WHITEPAPER.md)):

- Todo aparelho é **simultaneamente GATT central e peripheral**, retransmitindo pacotes em *flood* controlado.
- Cabeçalho binário compacto (versão, tipo, TTL, timestamp, flags) + IDs de remetente/destinatário + payload + assinatura opcional.
- Fragmentação em pedaços de ~469 bytes, ID de fragmento de 8 bytes, remontagem em cada nó (128 concorrentes, timeout de 30 s, teto de 1 MiB).
- **TTL inicial 7**, limitado pela densidade: grafo denso (≥6 enlaces) corta em 5; cadeia esparsa (≤2 enlaces) retransmite na profundidade cheia.
- Deduplicação por *seen-set* LRU (1000 entradas, expiração em 5 min); *jitter* de 10–220 ms no relay para evitar colisão.
- Criptografia: Noise XX para sessão viva (Curve25519/ChaCha20-Poly1305), Noise X para envelopes de *courier* offline; identidade por chave estática Curve25519 + Ed25519 de assinatura. Peer ID = primeiros 8 bytes do SHA-256 da chave estática.
- *Store-and-forward* em quatro mecanismos: outbox do remetente (100 msgs/par, TTL 24 h), envelopes de *courier* com orçamento *spray-and-wait*, histórico público sincronizado por gossip (janela de 6 h, 1000 pacotes), e caixas postais Nostr (retenção 24 h) — essa última **só quando há internet**, portanto fora das nossas restrições.

**Briar** — mensageria sem servidor central, sincroniza por Bluetooth, Wi-Fi ou cartão de memória ([projeto](https://briarproject.org/news/2021-briar-1.4-released/)). Importa aqui por dois recursos:
- *Offline sharing*: abre um hotspot local e serve uma página de onde o vizinho baixa o próprio app — resposta direta à restrição R3.
- Sincronização via pendrive/cartão SD (`Connections → Connect via Removable Drive`), o canal de último recurso.

**Bridgefy** — mensageria offline sobre BLE, comercial, com histórico de uso em protestos e apagões; já criticada por fragilidade do modelo de segurança em versões anteriores ([Wikipédia](https://en.wikipedia.org/wiki/Bridgefy)). Licenciamento por usuário ativo (MAU) é um fator a considerar caso se avalie SDK de terceiros.

### 3.2 Incompatíveis — e por quê

**Meshtastic** — malha LoRa madura e de longo alcance, mas **exige rádio externo**. Viola R2.

**Reticulum / Sideband / LXMF** — pilha de malha com roteamento zero-config, criptografia ponta a ponta e sigilo futuro; o Sideband já oferece **PTT, nota de voz e chamada ao vivo** via LXST ([Sideband](https://github.com/markqvist/Sideband), [manual](https://reticulum.network/manual/interfaces.html)). Arquiteturalmente é o que a proposta descreve — mas o transporte não fecha: a `AutoInterface` precisa de **enlace Ethernet/Wi-Fi local já existente** (switch, hub ou AP) com IPv6 link-local, e **não há interface Bluetooth/BLE nativa para Android**. A única interface de rádio documentada é RNode LoRa, que é hardware. Ou seja: ótima camada de rede, sem camada física que caiba em R1+R2. *Reusar o modelo de endereçamento/roteamento do Reticulum sobre um transporte BLE próprio é uma linha de investigação legítima.*

**Serval Project / batphone** — o precedente histórico mais próximo: voz, texto e arquivos em malha, com o Rhizome fazendo *store-and-forward* de arquivos ([wiki](http://developer.servalproject.org/dokuwiki/doku.php?id=content%3Atech%3Aserval_mesh), [repo](https://github.com/servalproject/batphone)). Morreu pelo transporte: dependia de **Wi-Fi Ad-Hoc**, que no Android só sobe com **root**, ficou instável e foi desabilitado nas versões recentes, com interoperabilidade ruim entre fabricantes. É a lição central do levantamento — **a arquitetura não salva uma escolha de camada física que o sistema operacional não sustenta.**

---

## 4. Camadas de rádio disponíveis em aparelhos 2020+

| Camada | Vazão prática | Multi-hop | Cobertura em aparelhos 2020+ | Dependências / travas |
|---|---|---|---|---|
| **BLE (GATT)** | 700–1400 kbps ponto a ponto otimizado; muito menos em malha | **Sim** — cada nó é central+peripheral ao mesmo tempo (modelo bitchat) | Universal | Nenhuma externa. Android 14+ passou a solicitar MTU ATT de 517 B por padrão na primeira conexão GATT |
| **Wi-Fi Direct (P2P)** | 16–20 Mbit/s medidos em testbed de 5 aparelhos | **Não nativamente** — cliente não vira Group Owner de outro grupo e clientes do mesmo grupo não se falam direto | Ampla | Todo GO usa o mesmo `192.168.49.1`, o que impede encaminhamento GO↔GO trivial; multi-grupo só com *relay* resolvido na aplicação |
| **Wi-Fi Aware (NAN)** | Alta (NDP sobre Wi-Fi) | **Sim** — topologia mais adequada à proposta | **Fraca** — Android 8.0+ por API, mas poucos modelos implementam de fato | Pode ficar indisponível se Wi-Fi Direct/SoftAP/tethering estiverem em uso; mensagens de serviço limitadas a ~255 B, sem garantia de entrega ou ordem |
| **Nearby Connections** | `P2P_STAR` maior; `P2P_CLUSTER` forma malha com banda menor | Malha via `P2P_CLUSTER` | Ampla | **Exige Google Play Services** — dependência proprietária ruim para um app de resiliência (aparelhos sem GMS, builds alternativas) |

Notas de suporte: Wi-Fi Aware desde Android 8.0 / API 26 ([Android Developers](https://developer.android.com/develop/connectivity/wifi/wifi-aware)); emuladores não suportam NAN, o que dificulta teste automatizado. No lado Apple, o **iOS 26 passou a expor Wi-Fi Aware** por exigência regulatória da Comissão Europeia, abrindo pela primeira vez a possibilidade de par iPhone↔Android com o mesmo app instalado — porém limitado a modelos recentes ([heise](https://www.heise.de/en/news/Peer-to-peer-WLAN-by-order-of-the-EU-Apple-integrates-Wi-Fi-Aware-10446649.html)). Isso é uma porta que abriu há pouco e **não** cobre a base instalada de 2020.

### O que a literatura mostra sobre Wi-Fi Direct multi-grupo

O estudo mais recente e concreto é o **SWARNET** ([arXiv 2604.07889](https://arxiv.org/pdf/2604.07889)): arquitetura em camadas (Flutter + Kotlin nativo, sockets ligados a interface específica, gestão de estado de relay, tabelas de encaminhamento por assinatura/tópico) rodando em **Android 11 sem root**. Resultados em cinco Samsung Galaxy A10s:

| Cenário | Pico de vazão no receptor |
|---|---|
| 2 aparelhos, 1 grupo | ~19,7 Mbit/s |
| 3 aparelhos, 1 grupo | ~17,9 Mbit/s |
| 4 aparelhos, 2 grupos | ~16,1 Mbit/s |
| 5 aparelhos, 3 grupos | ~16,0 Mbit/s |

Perda de pacotes cresce com a complexidade do encaminhamento, chegando a **19–20%** na região de maior carga do caso de três grupos. A conclusão dos próprios autores é de **viabilidade arquitetural**, não de prontidão para implantação.

Trabalhos anteriores sustentam duas decisões de projeto: **conexões simultâneas superam claramente o compartilhamento de tempo** entre grupos (Funai et al.), e **broadcast originado no Group Owner é o grande gargalo** da espinha dorsal (Casetti et al.).

---

## 5. Arquitetura de referência sugerida

O que a pesquisa converge — malha híbrida de dois planos:

**Plano de controle: BLE, sempre ligado.**
Descoberta, anúncio de presença, roteamento, mensagens curtas e sinalização. Cada nó opera como central e peripheral simultaneamente. Universal, baixo consumo, sem dependência de GMS. Semântica de entrega: *flood* controlado com TTL adaptado à densidade + *seen-set* LRU para deduplicação + *jitter* no relay. Vale adotar os parâmetros do bitchat como ponto de partida — fragmento de ~469 B, TTL 7 limitado a 5 em grafo denso, seen-set de 1000/5 min, jitter de 10–220 ms — por serem valores já exercitados em campo, não estimativas de mesa.

**Plano de dados: Wi-Fi Direct (ou Aware, onde houver) promovido sob demanda.**
Quando dois nós precisam mover volume — nota de voz, foto, o próprio APK —, negocia-se pelo BLE a subida de um enlace Wi-Fi P2P, transfere-se, e derruba-se o enlace. Isso evita pagar o custo de energia e de exclusividade de interface do Wi-Fi Direct no estado de repouso, e contorna a limitação de multi-hop: o Wi-Fi Direct nunca precisa rotear, só carregar um salto.

**Semântica de entrega: *store-and-forward*, não sessão.**
O app deve assumir que o destinatário está fora de alcance no momento do envio. Outbox persistente no remetente + *courier* oportunista com orçamento de cópias (*spray-and-wait*) + histórico público sincronizado por gossip. Mensagem entregue em minutos é sucesso; a UI precisa comunicar isso honestamente, e não imitar a expectativa de um mensageiro online.

**Criptografia desde o primeiro commit.** O levantamento sobre apps de malha no mercado aponta repetidamente segurança insuficiente para o modelo de ameaça real desses usos. Noise XX/X com identidade por chave estática é o padrão já validado.

---

## 6. Distribuição do app (R3) — risco aberto

Instalar "de smartphone para smartphone" tem prova de existência: o *offline sharing* do Briar sobe um hotspot local e serve o próprio APK por uma página web, sem internet.

O que mudou e complica: o **Android passou a exigir verificação de identidade do desenvolvedor** para que um app instale em aparelho certificado. A primeira rodada de enforcement entrou em vigor em **30 de setembro de 2026**, limitada a quatro países, com expansão global prevista incluindo os EUA em **2027**. Apps de desenvolvedor não verificado passam por um fluxo com **atraso de 24 h e avisos extras**, ou simplesmente não instalam; o **ADB permanece como via de escape** para usuário técnico ([Android Authority — como funciona](https://www.androidauthority.com/how-android-app-verification-works-3603559/), [rollout](https://www.androidauthority.com/google-android-advanced-flow-sideloading-rollout-begins-3700073/), [requisitos](https://www.androidauthority.com/android-developer-verification-requirements-3590911/)).

Consequência direta para este projeto: **um atraso de 24 horas no fluxo de instalação é incompatível com o cenário de uso** — pane de rede é exatamente quando se precisa instalar agora. A implicação de projeto é que a verificação de desenvolvedor deixa de ser burocracia e passa a ser requisito funcional, e que a propagação viral do APK em plena emergência não pode ser o plano principal. **Pendência:** confirmar o status do Brasil na expansão do enforcement e o comportamento exato do fluxo para app verificado distribuído fora da Play Store.

---

## 7. Contexto regulatório (Brasil)

A Anatel aprovou o **Regulamento sobre o Uso de Serviços de Telecomunicações em Desastres, Situações de Emergência e Estado de Calamidade Pública**, que obriga as prestadoras a se colocarem à disposição das autoridades e agentes de defesa civil ([Anatel](https://www.gov.br/anatel/pt-br/dados/utilidade-publica/alertas-de-desastres/defesa-civil-alerta)).

O canal oficial de alerta à população é o **Defesa Civil Alerta**, sobre **Cell Broadcast** — determinado pela Anatel em outubro de 2022 e que entrega mensagem direta aos celulares da área de risco **sem app nem cadastro prévio**. É um canal unidirecional de autoridade para população; este projeto é ortogonal e complementar a ele (bidirecional, entre pessoas, sem rede).

**A busca não encontrou norma da Anatel que regule especificamente aplicativos de comunicação direta entre aparelhos ou redes mesh.** Isso é ausência de resultado, não confirmação de que o espaço é livre: a operação se dá em faixas não licenciadas (Bluetooth/Wi-Fi) com hardware já homologado, o que *sugere* não haver exigência adicional — mas isso precisa de verificação jurídica antes de qualquer afirmação em material de projeto.

---

## 8. Lacunas de pesquisa — o que ainda não se sabe

| Lacuna | Por que importa |
|---|---|
| **Consumo de bateria** em *advertise* + *scan* BLE contínuo | Nenhum dado no levantamento. Para app de desastre é possivelmente a métrica mais crítica: um nó que drena a bateria em 4 h é um nó que não existe quando precisa |
| **Latência** em malha multi-salto | O SWARNET reporta só vazão e perda de pacotes; sem latência não se dimensiona nem PTT gravado |
| **Mobilidade e falha de relay** | Todo o testbed do SWARNET é estático, indoor, cinco aparelhos **idênticos** (Galaxy A10s), em proximidade. Não captura mobilidade, interferência nem queda de nó |
| **Heterogeneidade de fabricante/chipset** | A lição do Serval é justamente essa: a interoperabilidade entre vendors é onde esse tipo de projeto morre |
| **Alcance real por salto** | O whitepaper do bitchat não especifica alcance de rádio nem número máximo de pares |
| **Escala** | Nenhuma fonte do levantamento mede malha além de poucas dezenas de nós |

---

## 9. Próximos passos sugeridos

1. **Decidir a semântica de "rádio"** — PTT com áudio gravado (viável) ou voz ao vivo (exige abandonar R2). Essa decisão precede todas as outras.
2. **Medir bateria antes de arquitetar** — protótipo mínimo de BLE advertise+scan contínuo em 3–4 aparelhos de fabricantes diferentes, 24 h, só para levantar a curva de consumo.
3. **Protótipo de dois planos** — descoberta e texto sobre BLE; promoção para Wi-Fi Direct sob demanda para uma nota de voz. Dois saltos já expõem a maior parte dos problemas reais.
4. **Avaliar reuso vs. implementação** — o bitchat é a base mais próxima e está documentado o suficiente para ser reimplementado ou adaptado; checar licença e viabilidade de fork antes de escrever protocolo próprio.
5. **Resolver a verificação de desenvolvedor** cedo, porque é um prazo externo que não se negocia.

> **Encaminhamento (nov/2026):** os passos 1 e 4 foram resolvidos. A semântica adotada é **PTT com áudio gravado**, e a decisão de reuso é **fork do `bitchat-android`** (domínio público/Unlicense). Ver:
> - [`kit-emergencia.md`](kit-emergencia.md) — solução pronta para uso hoje, com preparação pré-crise
> - [`plano-fork.md`](plano-fork.md) — escopo do fork, pré-requisitos de build e fases de validação

---

## Referências

**Implementações e projetos**
- bitchat — [repositório](https://github.com/permissionlesstech/bitchat) · [whitepaper](https://github.com/permissionlesstech/bitchat/blob/main/WHITEPAPER.md)
- Briar — [release 1.4: offline sharing e sync por SD/USB](https://briarproject.org/news/2021-briar-1.4-released/)
- Bridgefy — [verbete](https://en.wikipedia.org/wiki/Bridgefy)
- Reticulum — [manual de interfaces](https://reticulum.network/manual/interfaces.html) · [Sideband](https://github.com/markqvist/Sideband) · [awesome-reticulum](https://github.com/lorien/awesome-reticulum)
- Serval Project — [wiki técnica](http://developer.servalproject.org/dokuwiki/doku.php?id=content%3Atech%3Aserval_mesh) · [batphone](https://github.com/servalproject/batphone) · [verbete](https://en.wikipedia.org/wiki/Serval_Project)
- Panorama — [Smartphone ad hoc network](https://en.wikipedia.org/wiki/Smartphone_ad_hoc_network)

**Camadas de rádio**
- [Wi-Fi Aware — Android Developers](https://developer.android.com/develop/connectivity/wifi/wifi-aware) · [AOSP](https://source.android.com/docs/core/connect/wifi-aware)
- [Nearby Connections — Strategy (P2P_STAR / P2P_CLUSTER)](https://developers.google.com/android/reference/com/google/android/gms/nearby/connection/Strategy)
- [Nearby Threats: Reversing, Analyzing, and Attacking Nearby Connections (NDSS'19)](https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_06B-3_Antonioli_paper.pdf)
- [Wi-Fi Aware no iOS 26 por exigência da Comissão Europeia](https://www.heise.de/en/news/Peer-to-peer-WLAN-by-order-of-the-EU-Apple-integrates-Wi-Fi-Aware-10446649.html)

**Vazão e voz**
- Edwards, Goodridge, Khan & Ramkissoon (Univ. of the West Indies). *Design and empirical validation of a stock-Android software architecture for Wi-Fi Direct multi-group communication* — [arXiv:2604.07889](https://arxiv.org/abs/2604.07889)
- [Supporting Multi-hop D2D Networks Through WiFi Direct Multi-group Networking](https://arxiv.org/pdf/1601.00028)
- [Argenox — maximizando vazão BLE](https://argenox.com/blog/bluetooth-le-throughput-max-performance) · [Novel Bits — cálculo de vazão BLE](https://novelbits.io/bluetooth-5-speed-maximum-throughput/)
- [ST UM2382 — voz full-duplex sobre BLE com compressão Opus](https://www.st.com/resource/en/user_manual/um2382-getting-started-with-the-stm32cube-function-pack-for-fullduplex-voice-streaming-over-bluetooth-low-energy-using-opus-compression-stmicroelectronics.pdf)
- [TI — Creating a Voice Enabled Application (BLE)](https://software-dl.ti.com/lprf/simplelink_cc2640r2_sdk/1.30.00.25/exports/docs/blestack/ble_sw_dev_guide/html/voice/voice.html)

**Distribuição e regulação**
- [Como funciona a verificação de app do Android](https://www.androidauthority.com/how-android-app-verification-works-3603559/) · [início do rollout](https://www.androidauthority.com/google-android-advanced-flow-sideloading-rollout-begins-3700073/) · [requisitos de verificação](https://www.androidauthority.com/android-developer-verification-requirements-3590911/)
- [Anatel — Defesa Civil Alerta](https://www.gov.br/anatel/pt-br/dados/utilidade-publica/alertas-de-desastres/defesa-civil-alerta)
