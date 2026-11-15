# comunica_cel

Estudo de viabilidade e plano de implementação para **comunicação entre smartphones sem infraestrutura** — sem LTE, sem rede Wi-Fi, sem servidor. O caso de uso é pane da rede de telefonia: um aplicativo que funcione como rádio entre aparelhos, usando apenas o hardware já presente em smartphones de 2020 em diante.

## A proposta, em três restrições

| # | Restrição |
|---|---|
| R1 | Sem infraestrutura: nem LTE, nem ponto de acesso Wi-Fi, nem servidor |
| R2 | Só hardware já presente em aparelhos 2020+ (elimina LoRa e acessórios) |
| R3 | Instalável de aparelho para aparelho, sem internet |

> R1 proíbe **rede** Wi-Fi, não o **chip** Wi-Fi. Wi-Fi Direct e Wi-Fi Aware falam direto com o aparelho ao lado, sem roteador — e portanto são permitidos.

## Conclusão do estudo

**Viável** em hardware de prateleira: texto, nota de voz e **PTT ao vivo** (half-duplex, melhor-esforço) em malha multi-salto.
**Não viável:** chamada full-duplex com garantia de entrega.

A proposta **já tem implementação madura e de código aberto** — o [bitchat](https://github.com/permissionlesstech/bitchat-android), em domínio público (Unlicense), com malha BLE, Wi-Fi Aware, PTT ao vivo interoperável com iOS e compartilhamento do próprio APK por hotspot com QR code. A recomendação deste estudo é **usar e, se necessário, derivar** — não reimplementar.

## Documentos

| Documento | O que traz |
|---|---|
| [`contexto.md`](contexto.md) | Estudo de viabilidade: estado da arte, camadas de rádio comparadas, orçamento de banda para voz, contexto regulatório brasileiro e lacunas de pesquisa |
| [`kit-emergencia.md`](kit-emergencia.md) | **Solução pronta para uso.** Procedimento de preparação pré-crise em 6 passos, com verificação de hash e ensaio |
| [`plano-fork.md`](plano-fork.md) | Plano de implementação por derivação do bitchat-android: escopo, pré-requisitos de build e fases de validação |

## Se você só quer comunicação de emergência funcionando

Vá direto ao [`kit-emergencia.md`](kit-emergencia.md). **Compilar não é requisito para usar** — e a preparação precisa ser feita **antes** da emergência, porque instalar um aplicativo durante o apagão pode simplesmente não funcionar.

## Aviso

Este estudo **não** substitui o canal oficial de alerta. No Brasil, o **Defesa Civil Alerta** (Cell Broadcast) entrega aviso direto ao celular na área de risco, sem aplicativo e sem cadastro. O que está aqui trata do caso oposto: comunicação lateral, entre pessoas, quando não há rede alguma.

Pesquisa de base conduzida em **novembro de 2026**. Prazos regulatórios e versões de software envelhecem — reconfirme antes de treinar um grupo.
