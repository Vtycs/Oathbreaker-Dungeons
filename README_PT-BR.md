# Oathbreaker Dungeons V0.0.2 :)

> Roguelite 2D com co-op online publicado na Steam — desenvolvido de forma independente.

[![Steam](https://img.shields.io/badge/Steam-Publicado-1b2838?style=flat&logo=steam)](https://store.steampowered.com/app/4461820/Oathbreak_Dungeons/)
[![Stack](https://img.shields.io/badge/Stack-Electron%20%2B%20p5.js-61DAFB?style=flat)](#stack)
[![Rede](https://img.shields.io/badge/Rede-P2P%20Steam-informational?style=flat)](#sistema-de-rede-p2p)

---

## Sobre o projeto
![2222](https://github.com/user-attachments/assets/9b53003e-c405-4a3e-ac3c-b9efbdb625e4)

Oathbreaker Dungeons é um roguelite de ação 2D com geração procedural de salas, sistema de buffs/runas e arquétipos, e **co-op online via Steam** para até 4 jogadores. O projeto foi desenvolvido de forma independente — da engine ao deploy na plataforma — e está disponível na Steam.

> **Este repositório é um portfólio técnico.** Apresenta a arquitetura, as decisões de engenharia e trechos representativos do código. O código-fonte completo é privado.

---

## Stack

| Camada | Tecnologia | Motivo da escolha |
|---|---|---|
| Runtime | **Electron** (Node.js) | Acesso nativo à API do Steam via bindings C++ |
| Render | **p5.js 1.9** | Loop de jogo controlável, API de canvas simples |
| Networking | **steamworks.js** | Bindings para Steamworks SDK (lobby, P2P, achievements) |
| Distribuição | **Steam** | Pipeline de build + deploy com Steamworks |

Usar p5.js como engine de jogo publicado na Steam é uma escolha não convencional que exigiu resolver problemas que a biblioteca não foi projetada para suportar — desde shaders de iluminação com correção de pixel density em telas HiDPI/Retina, até sincronização de estado de jogo em tempo real via protocolo P2P customizado.

---

## Arquitetura de módulos

```
├── main_js.js      # Processo principal Electron — Steamworks init, overlay, invites
├── steam.js        # Estado de lobby, protocolo de rede, achievements, stats
├── globals.js      # Dicionário de armas, buffs, configurações, controles
├── sketch.js       # Game loop principal (setup/draw), câmera, HUD, state machine
├── player.js       # Classe Player — física, combate, buffs, rendering
├── enemies.js      # Hierarquia de entidades — projéteis, inimigos, bosses
├── dungeon.js      # Geração procedural de salas, plataformas, lojas, arenas
├── ui.js           # Menus, lobby screen, tela de buffs, settings
└── audio.js        # Sound manager com contexto Web Audio
```

---

## Sistemas técnicos

### Sistema de rede P2P

O jogo implementa um protocolo P2P customizado sobre a camada de sockets do Steam, com os seguintes tipos de pacote:

```js
const PKT = {
    // Sessão
    HANDSHAKE: 10, HANDSHAKE_ACK: 11,
    LOBBY_SETTINGS: 1, GAME_START: 2,

    // Estado de jogo
    HOST_STATE: 20,      // Snapshot completo enviado pelo host
    CLIENT_INPUT: 21,    // Inputs do cliente com sequência
    PLAYER_STATES: 22,   // Posição/velocidade de todos os jogadores

    // Eventos de mundo
    ENEMY_SPAWN: 30, ENEMY_DEATH: 31,
    BOSS_SPAWN: 32, BOSS_PHASE: 33,
    ROOM_CHANGE: 34, BUFF_SELECT: 39,

    // Efeitos sincronizados
    TIME_STOP: 70, SLOW_MO: 73, SCREEN_EFFECT: 71,

    // Heartbeat e reconexão
    HEARTBEAT: 50, PING_REQ: 51, PING_RES: 52,
    HOST_LOST: 54, PLAYER_DROPPED: 55
};
```

**Características do protocolo:**
- Autoridade no host com reconciliação de estado nos clients
- Snapshot buffer com interpolação (buffer de 5 frames, delay de 100ms)
- Heartbeat com timeout de 8 segundos e detecção de host perdido
- Sequenciamento de inputs do cliente para evitar desync
- Efeitos de slow-motion e time stop sincronizados via pacotes dedicados
- Camada de confiabilidade reforçada: correção de bugs de Buffer/contextIsolation, correção da lógica de reliable-send, resolução de bugs de parsing de pacotes e sincronização da seed de sala entre clients

---

### Integração com o Steamworks

A integração vai além de inicializar a API. O processo principal trata **convites de lobby em runtime** recebidos via callback `GameLobbyJoinRequested`, com parsing defensivo do SteamID64:

```js
// Extrai SteamID64 de qualquer formato retornado pela API
// (bigint, number, string, objeto com campos variáveis)
function _extractId(v) {
    if (typeof v === 'bigint') return String(v);
    if (typeof v === 'string') {
        let s = v.replace(/n$/, '');
        if (/^\d{10,25}$/.test(s)) return s;
    }
    if (typeof v === 'object') {
        // Tenta campos conhecidos da API
        let candidates = [
            v.steamId64, v.steamId, v.m_steamid,
            v.lobbyId, v.m_steamIDLobby, v.lobby
        ];
        for (let c of candidates) {
            let r = _extractId(c);
            if (r) return r;
        }
    }
    return null;
}
```

Esse tratamento foi necessário porque diferentes versões do steamworks.js retornam o mesmo dado em formatos distintos (bigint nativo, string com sufixo `n`, objeto com campos variáveis).

**Outros pontos da integração:**
- Steam Overlay habilitado com `electronEnableSteamOverlay()`
- Callback loop rodando a 30hz via `setInterval`
- `+connect_lobby` via argv para join direto ao abrir o jogo por convite
- Inject de JavaScript no renderer via `executeJavaScript` para bridge main↔renderer

---

### Game Feel

Detalhes de física e feedback que tornam o movimento responsivo:

| Sistema | Implementação |
|---|---|
| **Coyote time** | 8 frames de tolerância após sair de plataforma |
| **Jump buffer** | Input de pulo armazenado por 8 frames |
| **Hit stop** | Congelamento de frames no impacto (`hitStopFrames`) |
| **Screen shake** | Intensidade máxima com decay de 0.80 por frame |
| **Squash & stretch** | Scale do personagem baseado em velocidade vertical e dash |
| **Afterimages** | Rastro visual durante o dash |
| **Slow motion** | Fator multiplicador no delta time, sincronizado via rede |

---

### Escala de renderização & suporte a alta taxa de atualização

Após a migração para um monitor 2K (2560x1440) 144Hz, o renderer ganhou uma escala de renderização com supersampling (`RENDER_SCALE`) desacoplada da lógica do jogo: a simulação continua rodando em um `frameCount` lógico fixo a 60 ticks/segundo, enquanto a renderização destrava para a taxa de atualização nativa do monitor.

---

### Hierarquia de entidades de plataforma

Todas as plataformas herdam de uma classe base e sobrescrevem `update()` e `display()` com comportamento próprio:

```
Platform (base)
├── LavaFloor     — sobe/desce com seno, causa dano e knockup
├── HazardSpike   — máquina de estados: dormindo → alertando → disparando
├── BouncyPad     — detecta colisão vertical e aplica impulso
├── IcePlatform   — reduz fricção horizontal do jogador
├── ConveyorBelt  — aplica velocidade horizontal contínua
└── DisappearPad  — some após contato, respawna com timer
```

---

### Sistema de buffs, runas e arquétipos

O jogo possui mais de 60 buffs únicos, 8 arquétipos, e um sistema expandido de combos de runas com 113 combinações (**Caos Verdadeiro**), além de um sistema de níveis de conta persistente para meta-progressão. O método `addBuff()` coordena todos os efeitos em cadeia:

1. Adiciona o buff ao array do jogador
2. Dispara partículas e texto flutuante
3. Aplica modificadores de stat imediatamente (HP, dano, velocidade)
4. Sincroniza via rede para todos os clients
5. Checa conquistas Steam vinculadas ao arquétipo
6. Persiste metaestats em disco

Alguns buffs afetam **todos os jogadores simultaneamente** — `B_LAST_STAND` reduz todos para 1 HP e triplica o multiplicador de dano; `B_LIFE_DEBT` dá um buff aleatório a todos mas duplica o HP e o dano dos inimigos da sala.

---

### Fusão de armas — sistema de FORMAS

A fusão de armas originalmente empilhava buffs simples de stat um sobre o outro — "um buff sobre buff, sem graça". Esse design foi abandonado em favor de identidades mecânicas e visuais distintas por combinação: cada fusão vira seu próprio arquétipo de arma (uma FORMA), não apenas um upgrade de números.

Exemplos:
- Pistola + Pistola → duas pistolas revezando tiros
- Escopeta + Lança → escopeta que atira lanças com pierce

FORMAS implementadas incluem: **Akimbo**, **Rajada**, **Munição Viva**, **Baioneta**, **Cano Duplo**, **Colosso** e **Lâminas Gêmeas**.

---

### Geração procedural de salas

O dungeon é gerado como um mapa infinito em grade (X, Y). Cada célula é construída sob demanda ao ser visitada pela primeira vez, com tipo definido por pesos ajustados conforme progressão:

- **Salas de combate** com 10+ tipos de inimigos e composições variadas
- **Miniboss rooms** com 4 chefes intermediários únicos (Ferreiro, Sacerdotisa, Centopeia, Polvo)
- **Boss final** determinístico, com 2 opções (Dragão / Coringa) — música e fase específicas
- **Lojas** com sistema de preços, itens aleatórios e interação via rede
- **Salas de buff** com seleção de cartas — sincronizadas entre todos os jogadores

---

### Shader de iluminação e correção HiDPI

O sistema de iluminação usa um graphics buffer separado (`lightBuf`) com blend mode para simular escuridão ao redor dos jogadores. Um problema não óbvio surgiu em telas HiDPI/Retina: o p5.js aplica `pixelDensity(2)` por padrão, fazendo o buffer de luz desalinhar com o canvas principal. A correção exigiu forçar `pixelDensity(1)` globalmente e nos buffers auxiliares.

```js
function setup() {
    pixelDensity(1); // Crítico: força 1:1 — corrige shader em telas HiDPI/Retina
    createCanvas(GAME_WIDTH, GAME_HEIGHT);

    let lightBuf = createGraphics(GAME_WIDTH, GAME_HEIGHT);
    lightBuf.noSmooth(); // Mesma pixel density que o canvas principal
    if (typeof initLightSystem !== 'undefined') initLightSystem(lightBuf);
}
```

---

### State machine de música

A trilha sonora muda dinamicamente de acordo com o contexto de jogo:

```
MENU / LOBBY      → faixa de menu
PLAY - sala comum → faixa de exploração
PLAY - MINIBOSS   → faixa específica por tipo de boss
    SMITH         → miniboss_smith
    PRIESTESS     → miniboss_priest
    CENTIPEDE     → miniboss_centi
    OCTOPUS       → miniboss_octo
PLAY - FINAL_BOSS → faixa específica
    DRAGON        → boss_dragon
    JOKER         → boss_joker
PLAY - SHOP       → faixa calma de loja
GAMEOVER          → faixa de game over
CREDITS           → faixa de créditos
```

---

## Conquistas Steam implementadas

20 conquistas integradas via Steamworks, incluindo:

| Conquista | Condição |
|---|---|
| Primeiro Sangue | Primeiro inimigo morto |
| Mestre do Parry | 50 parries em uma run |
| Za Warudo! | Usar time stop pela primeira vez |
| JACKPOT! | Tirar 777 no slot machine do arquétipo Apostador |
| Intocável | Derrotar um miniboss sem levar dano |
| Versátil | Jogar com todos os 8 arquétipos |
| Irmãos de Armas | Completar uma run em co-op online |
| Relâmpago | Completar uma run em menos de 15 minutos |
| Exagero | Causar mais de 500 de dano em um único golpe |

---

## Desafios resolvidos

- **Pixel density + shader desalinhado** — corrigido forçando `pixelDensity(1)` global
- **SteamID64 em múltiplos formatos** — parser defensivo que trata bigint, string, objeto
- **Convites Steam em runtime** — bridge main process → renderer via `executeJavaScript`
- **Slow motion sincronizado em rede** — pacote dedicado com aplicação client-side
- **P2P sem servidor dedicado** — protocolo host-authoritative sobre Steamworks sockets
- **Resync ao trocar de sala** — host envia layout completo da nova sala ao client após geração
- **Engine não projetada para jogos** — p5.js adaptado com hit stop, coyote time, múltiplos buffers gráficos
- **Reforço de confiabilidade de rede** — correção de bugs de Buffer/contextIsolation, correção da lógica de reliable-send, resolução de bugs de parsing de pacotes e sincronização da seed de sala entre clients
- **Renderização em alta taxa de atualização desacoplada da simulação** — escala de renderização com supersampling (`RENDER_SCALE`) com a lógica do jogo mantendo tick fixo a 60/s via frameCount lógico

---

## Imagens

![23334](https://github.com/user-attachments/assets/1caaf8a6-3f1f-4f54-a8d7-710f60b482cf) 
![02299](https://github.com/user-attachments/assets/deb67342-da63-47fd-8715-aca78b43aa01)


*Desenvolvido de forma independente. Publicado na Steam.*
*Developed by Empty TEAM -Poggerz*
