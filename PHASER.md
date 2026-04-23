# Phaser

**Phaser** é um framework JavaScript open-source para **jogos 2D HTML5**, desktop e mobile. Desenvolvido por Richard Davey (Photon Storm), é um dos frameworks de jogos web mais usados — atrás apenas de engines grandes (Unity WebGL, Godot Web). Ideal quando o alvo é **navegador** com ciclo de desenvolvimento rápido, JS/TS nativo, sem build pesada de Unity/Unreal, e distribuição por URL.

Ecossistema maduro (Phaser 3 desde 2018, Phaser 4 em transição 2024-2025), comunidade ativa, aprendizado relativamente suave para quem conhece JS.

## Por Que Phaser

- **Canvas + WebGL** com fallback automático: renderiza em qualquer navegador moderno.
- **API rica** cobrindo sprites, input, física, áudio, partículas, tweens, câmeras, tilemaps, scenes, UI.
- **TypeScript definitions** completas.
- **Distribuição trivial**: arquivo JS + assets, hospedagem estática. Sem app store, sem instalação.
- **Mobile via wrap**: Capacitor, Cordova, Ionic para iOS/Android nativos; Electron/Tauri para desktop.
- **Ótimo para game jams, prototipagem, jogos educacionais, publicidade interativa, Web3 mini-jogos**.

Limitações:
- **Só 2D**. Para 3D: Three.js, Babylon.js, PlayCanvas.
- **Performance móvel** decente mas inferior a engines nativas em jogos pesados.
- **Distribuição em app stores** requer wrapping + compliance (Apple é rígido com jogos HTML5).

## Versões

- **Phaser 2 (CE)**: legado (2014-2017), muitos tutoriais antigos. Baseado em Pixi.js 3.
- **Phaser 3** (2018-atual): reescrita completa, modular, ES6+, scenes como primeira classe. **Versão dominante**.
- **Phaser 4**: em transição (2024-2025) — performance, TypeScript-first, melhor bundling.

Este arquivo cobre **Phaser 3** (estado da arte em 2025-2026).

## Estrutura Mínima

```js
import Phaser from 'phaser';

const config = {
  type: Phaser.AUTO,       // WebGL com fallback Canvas
  width: 800,
  height: 600,
  parent: 'game-container',
  physics: {
    default: 'arcade',
    arcade: { gravity: { y: 300 }, debug: false }
  },
  scene: { preload, create, update }
};

const game = new Phaser.Game(config);

function preload() {
  this.load.image('logo', 'assets/logo.png');
}

function create() {
  this.add.image(400, 300, 'logo');
}

function update(time, delta) {
  // lógica por frame
}
```

Tudo roda a ~60 FPS; `update(time, delta)` é o loop.

## Scenes

Unidade organizacional fundamental. Uma scene é conjunto de lógica + assets + objetos (ex.: menu, jogo, game over).

### Ciclo de vida

```js
class GameScene extends Phaser.Scene {
  constructor() { super('GameScene'); }
  init(data) { /* recebe params */ }
  preload() { /* carrega assets */ }
  create(data) { /* instancia objetos */ }
  update(time, delta) { /* lógica por frame */ }
}
```

### Transições entre scenes

```js
this.scene.start('GameOver', { score: 100 });
this.scene.launch('HUD');     // rodar em paralelo (overlay)
this.scene.pause();
this.scene.resume();
this.scene.stop();
```

**Padrão**: `Boot` → `Preload` → `Menu` → `Game` → `GameOver`, com HUD paralelo.

## Game Objects

Entidades visuais + lógicas. Herdam de `GameObject`:

- **Sprite**: imagem animável. `this.add.sprite(x, y, 'key')`.
- **Image**: estático.
- **Text / BitmapText**: texto vetorial ou bitmap font.
- **Graphics**: desenho procedural (linhas, formas, fills).
- **TileSprite**: textura repetida (background rolante).
- **RenderTexture**: desenhar em textura.
- **Container**: agrupa objetos; transform aplicado a todos.
- **Rope**: deformação em curva.
- **Zone**: área invisível (input, colisão lógica).

Cada objeto tem propriedades comuns: `x, y, scale, rotation, alpha, visible, depth, origin, flipX, flipY, blendMode, tint`.

### Animação (Sprite Animations)

```js
this.anims.create({
  key: 'walk',
  frames: this.anims.generateFrameNumbers('player', { start: 0, end: 7 }),
  frameRate: 10,
  repeat: -1      // loop
});

sprite.play('walk');
sprite.stop();
```

## Física

Três motores integrados:

### Arcade Physics (default)

- Simples, rápido, AABB (axis-aligned bounding boxes) ou circular.
- Ideal para platformers 2D, top-down, shoot'em up.
- Sem rotação dinâmica em colisão (objeto colide como retângulo fixo).
- **Overlap** (detecção sem resposta) vs **Collide** (com resposta).

```js
this.physics.add.collider(player, platforms);
this.physics.add.overlap(player, coins, collectCoin, null, this);

player.setVelocity(160, -330);
player.setBounce(0.2);
player.setCollideWorldBounds(true);
```

### Matter.js

- **Físicamente realista**: corpos rígidos com rotação, constraints, joints, molas.
- Polígonos arbitrários.
- Mais caro.
- Bom para puzzles físicos (Angry Birds-like, World of Goo-like).

### Impact

Legado, menos usado. Para compatibilidade com jogos Impact.js.

## Input

### Teclado

```js
this.cursors = this.input.keyboard.createCursorKeys();
// em update:
if (this.cursors.left.isDown) player.setVelocityX(-160);

this.input.keyboard.on('keydown-SPACE', () => player.jump());
```

### Pointer (mouse + touch)

```js
this.input.on('pointerdown', (pointer) => { ... });
sprite.setInteractive();
sprite.on('pointerover', () => sprite.setTint(0xff0000));
sprite.on('pointerout', () => sprite.clearTint());
```

### Gamepad

```js
this.input.gamepad.once('connected', (pad) => { ... });
// em update:
if (pad.A) jump();
if (pad.leftStick.x > 0.1) moveRight();
```

### Drag

```js
this.input.setDraggable(sprite);
sprite.on('drag', (pointer, x, y) => { sprite.x = x; sprite.y = y; });
```

## Audio

Web Audio (fallback HTML5 Audio):

```js
this.load.audio('music', 'assets/theme.mp3');
// ...
const music = this.sound.add('music', { loop: true, volume: 0.5 });
music.play();

const sfx = this.sound.add('jump');
sfx.play({ detune: Math.random() * 200 - 100 }); // variação tonal
```

### Sound Manager

- **WebAudio**: moderno; `SoundManager.context` expõe AudioContext (permite efeitos custom — filters, convolvers; útil para phaser **effect** coincidentemente, mas não confunda — são nomes diferentes).
- **HTML5Audio**: legado; menos features.
- **Spatial** (2D sound positioning): suportado.

### iOS unlock

Browsers mobiles bloqueiam audio até **primeira interação do usuário**. Phaser lida automaticamente, mas observe.

## Tweens

Interpolação de propriedades ao longo do tempo — animação sem spritesheet.

```js
this.tweens.add({
  targets: sprite,
  x: 400,
  y: 300,
  alpha: 0,
  duration: 1000,
  ease: 'Cubic.easeOut',
  yoyo: true,
  repeat: 2,
  onComplete: () => console.log('done')
});
```

Múltiplos objetos, múltiplas propriedades, easing customizado, timelines.

## Partículas

Sistema de partículas integrado para explosões, faíscas, chuva, fumaça:

```js
const particles = this.add.particles('spark');
const emitter = particles.createEmitter({
  speed: { min: 100, max: 200 },
  scale: { start: 1, end: 0 },
  lifespan: 1000,
  quantity: 20,
  blendMode: 'ADD'
});
```

Em Phaser 3.60+: API simplificada via `this.add.particles(x, y, 'key', config)`.

## Câmeras

Controle de viewport:

```js
this.cameras.main.setBounds(0, 0, 2000, 600);
this.cameras.main.startFollow(player);
this.cameras.main.setZoom(2);
this.cameras.main.shake(500, 0.01);       // screen shake
this.cameras.main.flash(200, 255, 0, 0);  // flash vermelho
this.cameras.main.fade(500, 0, 0, 0);     // fade to black
```

Múltiplas câmeras (HUD em segunda câmera sobre main).

## Tilemaps

Mapas baseados em tiles — padrão em platformers, top-down. Ferramenta canônica: **Tiled** (mapeditor.org), exporta JSON.

```js
this.load.tilemapTiledJSON('map', 'assets/map.json');
this.load.image('tiles', 'assets/tileset.png');
// ...
const map = this.make.tilemap({ key: 'map' });
const tileset = map.addTilesetImage('tileset', 'tiles');
const ground = map.createLayer('Ground', tileset, 0, 0);
ground.setCollisionByProperty({ collides: true });
this.physics.add.collider(player, ground);

// objetos em object layer do Tiled
const spawns = map.getObjectLayer('Spawns').objects;
```

Suporta ortogonal, isométrica, staggered.

## Loader

Assets carregam em `preload()`. Tipos: `image`, `spritesheet`, `audio`, `json`, `atlas` (textura + metadata JSON), `bitmapFont`, `tilemapTiledJSON`, `plugin`, `script`, `video`.

### Progress

```js
this.load.on('progress', (value) => {
  loadingBar.width = 300 * value;
});
```

### Texture atlases

Múltiplos sprites em uma textura + JSON descritor. Produzidos por TexturePacker, Free Texture Packer. Reduz draw calls.

## Plugins

Phaser suporta plugins escopados em scene ou globais:

- **rexUI** (Rex): botões, listas, input, dialogs — UI ausente de Phaser core.
- **rexPlugins** diversos: grid table, virtualização, gesture.
- **Phaser-Nav**: pathfinding A*.
- **phaser3-postfxplugins**: efeitos de shader (glow, blur, CRT, glitch).
- **Spine plugin** para animação esqueletal.

Comunidade ativa em GitHub + Phaser Discourse forum.

## Shaders e Post-Processing

Phaser 3.50+ suporta pipelines custom em WebGL:

```js
class MyPipeline extends Phaser.Renderer.WebGL.Pipelines.PostFXPipeline {
  constructor(game) {
    super({
      game,
      fragShader: `... GLSL ...`
    });
  }
}

this.game.renderer.pipelines.addPostPipeline('MyPipeline', MyPipeline);
camera.setPostPipeline('MyPipeline');
```

Clássicos: glow, blur, pixelate, CRT, chromatic aberration, bloom.

## State / Data

- **Registry**: key-value global acessível de qualquer scene. `this.registry.set('score', 100)`.
- **Scene data**: passada em `this.scene.start('Next', { foo: 1 })`.
- **Events**: `this.events.emit('score', val)` / `on`.
- **Local storage**: para save games persistentes. Sem API Phaser; use `localStorage` direto.

## Mobile / Responsivo

```js
const config = {
  scale: {
    mode: Phaser.Scale.FIT,        // ou RESIZE, ENVELOP
    autoCenter: Phaser.Scale.CENTER_BOTH,
    width: 800,
    height: 600
  },
  ...
};
```

Modos:
- **FIT**: mantém aspect ratio, barras pretas se necessário.
- **RESIZE**: game size ajusta ao canvas.
- **ENVELOP**: preenche sem barras, pode cortar.
- **WIDTH_CONTROLS_HEIGHT**, **HEIGHT_CONTROLS_WIDTH**.

Touch controls: analog virtual stick, botões. Plugin `phaser3-rex-plugins/plugins/virtualjoystick`.

## Performance

- **Texture atlases** reduzem draw calls (GPU envia batch).
- **Object pools**: reutilize sprites em vez de criar/destruir (`this.physics.add.group`).
- **setVisible(false)** melhor que destruir/recriar.
- **Culling automático** fora da camera.
- **Física Arcade** < Matter em custo.
- **Quantos objetos ativos?** Phaser lida bem com 100-1000; acima disso, otimizar.
- **Chrome DevTools Performance tab** + `game.renderer.drawCount` para medir.
- **WebGL Stats** plugin.

## Distribuição

### Web

Bundle via **Webpack, Vite, Parcel, esbuild, Rollup**. Templates oficiais Phaser 3 + Vite / Webpack / Next em `github.com/phaserjs/template-vite`, etc.

Hospedagem: **GitHub Pages, Netlify, Vercel, itch.io** (ótimo para jogos indie), **CrazyGames, Poki, Kongregate** (monetização via ads).

### Mobile app stores

- **Capacitor** (Ionic): mais moderno, webview nativa + plugins.
- **Cordova**: legado.
- **Cocoon / Ludei**: obsoleto.

Performance aceitável para jogos moderadamente complexos; jogos pesados fazem mais sentido em Unity/Godot.

### Desktop

- **Electron**: wrap de Chromium.
- **Tauri**: Rust-based, menor bundle.
- **NW.js**: similar a Electron.
- Distribuição via Steam, itch.io, releases GitHub.

## TypeScript

Phaser 3 tem `.d.ts` completo. Uso idiomático:

```ts
import Phaser from 'phaser';

class GameScene extends Phaser.Scene {
  private player!: Phaser.Physics.Arcade.Sprite;

  constructor() { super('GameScene'); }

  create() {
    this.player = this.physics.add.sprite(100, 100, 'player');
  }
}
```

## Ecossistema e Ferramentas

### Editores e assets

- **Tiled** (mapeditor.org): tilemaps, object layers, properties.
- **TexturePacker, Free Texture Packer**: atlas.
- **Aseprite, Piskel**: pixel art + animação.
- **Spine, DragonBones**: skeletal animation.
- **bfxr, sfxr**: sound effects procedurais.
- **BeepBox, Bosca Ceoil**: música chiptune simples.
- **Audacity, FL Studio, Ableton**: áudio pro.

### Development

- **Phaser Editor 2D** (Arian Fornaris): IDE dedicada para Phaser. Comercial.
- **Visual Studio Code** + TypeScript + Phaser types: stack padrão.
- **Phaser Studio** (official, em beta): futuro editor oficial.

### Debugging

- `game.renderer` expõe draw stats.
- `scene.physics.world.drawDebug = true` em Arcade.
- Chrome DevTools + Phaser Debug plugin.

## Alternativas

| Framework | Foco | Trade-off |
|---|---|---|
| **Phaser** | 2D HTML5 generalista | equilíbrio; curva moderada |
| **PixiJS** | rendering 2D puro | rápido; sem física/input — DIY |
| **Three.js** | 3D WebGL | 3D, não 2D-optimized |
| **Babylon.js** | 3D WebGL pro | feature-rich; maior curva |
| **PlayCanvas** | 3D + editor online | bom para mobile |
| **Godot** (Web export) | 2D + 3D, engine completa | editor rico; WebAssembly; bundle maior |
| **Unity WebGL** | 2D/3D, industrial | alto polish; bundle grande; slow boot web |
| **Construct 3** | no-code HTML5 | rápido para não-devs; menos flexível |
| **Defold** | 2D, engine nativa + web | Lua; performance boa |
| **Kaboom.js** | 2D simples, didático | ótimo para iniciantes |

## Casos de Uso

### Bom para

- Game jams (Ludum Dare, GMTK, Global Game Jam).
- Jogos casuais para web (.io games, hyper-casual).
- Prototipagem rápida antes de portar para engine pesada.
- Advergames, jogos educacionais, serious games.
- Jogos em Facebook Instant Games, Discord Activities, Telegram Mini Apps.
- Gamificação em aplicações web.
- Eventos e marketing interativo.

### Não ideal para

- AAA 3D.
- Jogos com física altamente complexa (use Unity + PhysX).
- Títulos console-primeira (Unreal/Unity).
- Jogos com mod support rico.

## Aprendizado

### Caminho sugerido

1. **JS/TS proficiente** antes de Phaser (Phaser não ensina JS).
2. **Tutorial oficial** (phaser.io/tutorials/making-your-first-phaser-3-game).
3. **Mini-jogo completo**: platformer ou pong. Preload → scenes → physics → input → HUD.
4. **Segundo jogo** com tilemap Tiled + inimigos + game over.
5. **Game jam** para completar algo pequeno sob prazo.
6. **Estudar código open-source**: github topic `phaser3`.

### Padrões arquiteturais

- **ECS leve**: compor comportamentos em grupos/plugins em vez de hierarquias profundas.
- **Scene management consistente**: Boot → Preload → Menu → Game.
- **Eventos > chamadas diretas** entre sistemas desacoplados.
- **Separar dados de lógica**: config em JSON, scripts em JS/TS.
- **Testing**: Jest + Phaser em jsdom é possível mas trabalhoso; a maioria testa visualmente.

## Phaser na Indústria

- **Rovio, Disney, Cartoon Network**: usaram para web-tie-in games.
- **Miniclip, Poki, CrazyGames**: catálogo significativo em Phaser.
- **BBC Bitesize, PBS Kids**: educacional.
- **Facebook Instant Games**: Phaser muito presente.

Escala de produção: jogos de milhões de jogadores têm rodado em Phaser.

## Recursos

- **phaser.io**: site oficial, docs, exemplos (`phaser.io/examples` — biblioteca massiva de snippets).
- **phaser.io/learn**: tutoriais, cursos.
- **Phaser Discourse**: forum oficial.
- **Phaser Discord**: comunidade em tempo real.
- **Ourcade, Emanuele Feronato, GameDev Academy**: blogs e tutoriais aprofundados.
- **"Discover Phaser 3 with ES6"** — Thomas Palef.
- **"HTML5 Game Development with Phaser"** — Travis Faas.
- **YouTube**: Ourcade channel, William Clarkson, Zenva.
- **Templates oficiais** em GitHub: `phaserjs/template-*`.

## Curiosidade Sobre o Nome

**Phaser** em português não traduz bem — é o termo inglês para "phased array" (arranjo em fase, em antenas) e também:

1. **Efeito de áudio** — pedal de guitarra, processador que cria varredura espectral via all-pass filters em cascata (Pink Floyd, Eddie Van Halen). **Outro** significado, não relacionado ao framework.
2. **Arma fictícia Star Trek**.
3. **Framework de jogos HTML5** (este arquivo).

O nome do framework é referência à arma de Star Trek (Richard Davey é fã confesso).

## Princípios

1. **Scene é unidade de composição**. Bem modelar scenes resolve a maior parte da organização.
2. **Física Arcade por default**; Matter quando precisar rotação realista.
3. **Atlas texturas** reduz draw calls — ganho grande em mobile.
4. **Object pools** evitam churn do garbage collector.
5. **Eventos > acoplamento direto** entre sistemas.
6. **TypeScript vale o esforço** em projeto > mini.
7. **Test no alvo real**: mobile iOS é o ambiente mais restritivo (audio, memory, performance).
8. **Versionar assets com build**: stale cache em CDN é bug comum.
9. **Game jam antes de compromisso grande**: descobre se framework cabe no seu fluxo.
10. **Phaser é framework, não engine completa**: sem editor visual (ainda); com liberdade proporcional.
