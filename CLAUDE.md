# CLAUDE.md — Mini Survivors: Системный справочник (актуально)

> Файл: `index.html` — ~17,000 строк, один IIFE, без сборщика, без зависимостей.
> Внешние ресурсы: Яндекс Игры SDK v2 + `track_music.m4a` (фоновая музыка).

---

## Структура файла

```
index.html
├── <head>
│   ├── <script src="https://yandex.ru/games/sdk/v2">
│   └── <style> — весь CSS (~3,000+ строк)
│       ├── Базовые стили, экраны, HUD, кнопки
│       ├── Upgrade/Class/Shop/Death/Revive/Settings/Relic/Mutation/Market экраны
│       ├── Location carousel (ls-*)
│       ├── Pause overlay (pause-*)
│       ├── DNA grid (.dna-node), Relic cards (.relic-card), Market cards (.mkt-card)
│       └── Media queries: 320/480/768/1024/1280/1440/1920px + touch/mobile overrides
├── <body>
│   ├── canvas#gc
│   ├── #hud (opacity:0 по умолчанию, .hud-visible во время игры)
│   │   ├── #btnPause (⏸, по центру сверху)
│   │   ├── #hud-left (HP/XP/Overheat бары)
│   │   ├── #hud-right (время, уровень, убийства, цель)
│   │   ├── #hudCredits, #hudCombo (статические элементы)
│   │   └── #relicBar (иконки активных реликвий, bottom-center)
│   ├── #adBuffHud, #btnAdBuff (position:fixed на body, z-index:50)
│   ├── Все .screen оверлеи (show/hide через classList.add/remove "active")
│   ├── #dailyModal, #locationScreen  ← ОБЯЗАНЫ быть ДО <script>
│   ├── #pauseOverlay (position:fixed, z-index:500)
│   ├── <audio id="menuMusic"> (track_music.m4a, loop, preload=auto)
│   └── <script> — единый IIFE
│       ├── Константы: SKINS, LOCATIONS, CLASSES, UPGRADES, SKILLS, SHOP_ITEMS, ICONS
│       ├── RELICS[12], DNA_NODES[20] — перед Game IIFE
│       ├── Утилиты: LS, $, clamp, dist, fmtTime, todayKey, mulberry32, hashString
│       ├── Яндекс SDK: _ysdk, _adLock, GameplayAPI, Ads{showInterstitial, showRewarded, submitScore}
│       ├── MenuMusic IIFE: HTML <audio> элемент, VOLUME=0.08, ON/OFF toggle
│       ├── Audio (Web Audio API): masterGain, osc(), noise(), 10 звуков + setVolume()
│       ├── Input IIFE: K[], mouse, touch joystick, dashPressed()
│       ├── Particles, FloatingText, XpOrbs, ArtifactSystem
│       ├── Bg state: _fireBuf, _voroSeeds, _voroCache, _flowParticles
│       ├── drawBg() + drawTiles() внутри
│       ├── Player, Enemy, Fast, Ranged, Splitter, Tank, Dasher, Gunner, MiniBoss
│       ├── upgradeChoices(), buildSkillChoices(), applySkill()
│       ├── I18N, tr(), show/hide, hudShow/hudHide, UI{}
│       ├── Shop, Daily, Location carousel, Skin preview (SVG)
│       ├── Game IIFE: loop, drawFrame, onDie, doRestart, openPause, closePause
│       │   ├── triggerUltimate() — 4 класс-специфичных ульта
│       │   ├── offerRelic() / pickRelic() / resumeFromRelic()
│       │   ├── checkDnaUnlocks() — вызывается в showDeath()
│       │   └── openMetaScreen() — табы [ANTI-DDoS] [DNA TREE]
│       └── Game.applyLanguage()  ← вызов снаружи
```

---

## Игровой цикл (`loop(ts)`)

Порядок шагов (только при `state === "playing"`):

1. `dt` (кэп 0.05с)
2. FPS smooth → `_pQuality` (0.35/0.65/1.0) + адаптивный DPR
3. `hitStop` → ранний return
4. `elapsed += dt`; `artifacts.update()`
5. Пассивная регенерация
6. Опасности локации (`hazard:"fire"` → урон 8 каждые 15с; `"freeze"` → `player._freezeT=3`)
7. Surge-окно + таймер спавна → `spawnEnemy()`
8. Проверка босса → `spawnBoss()`
9. `player.update()` → результат атаки
10. Камера: `camX = player.x - viewW/2`, `camY = player.y - viewH/2`
11. Граница мира r=1200 + shake
12. Цикл врагов: `e.update()`, elite-поведение (enrage/spectral), текст урона, смерть
13. Токсичные лужи (OVERCLOCKED elite): update + урон с накоплением
14. `relicPending` → `offerRelic()`
15. `enemyBullets` (Gunner), `playerBullets` (визуал + soft homing)
16. `orbs.update()` → XP → `offerUpgrade()` при level-up
17. Overheat → `triggerUltimate()`
18. `parts.update()`, `texts.update()`, ad-бафф тик
19. `UI.update()`
20. Wave-алерты (10/25/45/60/90/120/180/240/300с)
21. HP < 0 → `onDie()`
22. `drawFrame()`

---

## Рендеринг (`drawFrame()`)

Back-to-front:
1. `clearRect(0, 0, viewW, viewH)`
2. `drawBg()` — тайлы + процедурный паттерн (пропускается при `_pQuality < 0.65`)
3. Орбы XP
4. `enemyBullets`, `artifacts.draw()`, `playerBullets`
5. `parts.draw()`
6. Токсичные лужи (зелёный glow)
7. Враги, игрок (скин/форма/шлейф/орбиталы/комбо-счётчик)
8. `texts.draw()`
9. Ult vfx (кольцо расширяется до 320px), flash overlay, туман мира, panic vignette

### КРИТИЧНО: Canvas DPR

```js
// resize():
viewW = innerWidth; viewH = innerHeight;
canvas.width  = Math.round(innerWidth  * dpr);
canvas.height = Math.round(innerHeight * dpr);
ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
```

**Никогда** `canvas.width/2` для камеры. Только `viewW/viewH`.

### Адаптивный DPR

```js
let _dprOverride = null, _dprSwitchCd = 0;
// В loop(): при fpsSmooth < 25 → _dprOverride = 1, resize(); throttle 3s
```

---

## Производительность (`_pQuality`)

| FPS | `_pQuality` | Эффект |
|-----|------------|--------|
| < 30 | 0.35 | drawBg skip, тени выключены |
| 30–45 | 0.65 | Тени выключены |
| > 45 | 1.0 | Полное качество |

`const _shadOn = _pQuality >= 0.65;` — тени пуль/частиц.  
Туман мира: только при `_pdist > WORLD_R * 0.55 || _pQuality >= 0.65`.  
Caps: `MAX_ENEMY_BULLETS = 96`, `MAX_PLAYER_BULLETS = 80`.

---

## Фоны (`drawBg`)

### `drawTiles(tW, tH, baseCol, gridCol, accentCol, style)`

```js
// ПРАВИЛЬНО — float offset, round только при рисовании:
const ox = ((-cx % tW) + tW) % tW;     // float, плавный скролл
const tx = Math.round(gx * tW - ox);   // round ЗДЕСЬ
ctx.rect(tx + 0.5, ty + 0.5, tW-1, tH-1); // +0.5 = чёткие 1px линии

// НЕПРАВИЛЬНО (создаёт 1px рывки):
const icx = Math.round(cx);  // НЕ ДЕЛАТЬ
```

| Стиль | Локация |
|-------|---------|
| `metal` | Server — инсет + болты |
| `lava` | Core — трещины + мерцание |
| `ice` | Cryo — glint |
| `void` | Void — чередование + точки |

### Процедурные паттерны

| Локация | Паттерн | State |
|---------|---------|-------|
| Server | PCB-трассы 224px + бегущие пакеты | нет |
| Core | Клеточный автомат огня | `_fireBuf` 120×70 Float32Array |
| Cryo | Voronoi живые кристаллы | `_voroSeeds[12]`, `_voroCache` ImageData |
| Void | Flow field частицы | `_flowParticles[320]` |

---

## Система локаций

### 4 локации

| ID | Разблокировка | HP врагов | SPD врагов | XP | Кредиты | Хазард |
|----|--------------|-----------|------------|-----|---------|--------|
| `server` | Сразу | ×1.0 | ×1.0 | ×1.0 | ×1.0 | нет |
| `core` | Выжить 3 мин | ×1.25 | ×1.15 | ×1.3 | ×1.25 | огонь 15с |
| `cryo` | Level 10 | ×1.5 | ×0.75 | ×1.5 | ×1.4 | заморозка 20с |
| `void` | 200 убийств | ×1.8 | ×1.3 | ×2.0 | ×1.8 | нет |

---

## Элитные враги (Elite Affixes)

Спавнятся через `spawnEnemy()` с `elitePrefix`. Поведение вешается в том же месте:

| Prefix | Поле | Поведение |
|--------|------|-----------|
| `HARDENED` | `_armorShield` | Поглощает ~35% HP как щит (cyan-полоса над HP) |
| `ALPHA` | `_willEnrage` | На 50% HP: скорость ×1.9, цвет → `#ff4400` |
| `OVERCLOCKED` | `_poisonOnDeath` | При смерти: токсичная лужа r=55, 5с, 10 дмг/с |
| `CORRUPTED` | `_spectralCd` | Телепортируется к игроку каждые 3.5–6с |

**Токсичные лужи**: `toxicPuddles[]` state var. Урон через накопитель `tp._dmgAcc` — без него `Math.round(10*dt) = 0`.

```js
tp._dmgAcc = (tp._dmgAcc || 0) + 10 * dt;
if (tp._dmgAcc >= 1) { player.damage(Math.floor(tp._dmgAcc), false); tp._dmgAcc -= Math.floor(tp._dmgAcc); }
```

---

## Система реликвий (Relic System)

- `RELICS[12]` — массив перед Game IIFE
- Триггер: убийство MiniBoss → `relicPending = true`
- В loop: `if (relicPending && state==="playing") { relicPending=false; offerRelic(); }`
- `offerRelic()`: `state="relic"`, показывает `#relicScreen` с 3 случайными реликвиями
- `pickRelic(r)`: применяет, обновляет `#relicBar` в HUD
- Максимум 4 реликвии за забег. `player.relics[]` — массив ID
- `doRestart()` сбрасывает автоматически (новый `Player()`)

---

## ДНК мета-прогрессия (DNA Meta-Progression)

- `DNA_NODES[20]` — массив перед Game IIFE
- `META` поля: `totalKills`, `totalBossKills`, `totalEliteKills`, `dna: {}`
- `checkDnaUnlocks()` вызывается в `showDeath()` после `saveMeta()`
- Бонусы применяются в конструкторе `Player` через цикл `DNA_NODES`
- Отображение: таб `[DNA TREE]` в `openMetaScreen()`

---

## Классы и ультиматы (`triggerUltimate()`)

| Класс | Ульт | Радиус / Эффект |
|-------|------|-----------------|
| `antivirus` | Chain Nova | До 10 врагов в 320px, dmg × 2.8 |
| `firewall` | Iron Bunker | 3с iframes + аура 180px, dmg × 2.5 |
| `scanner` | Phase Burst | 5 целей в 320px + AoE 80px вокруг каждой |
| `hacktivist` | Critical Mass | Следующие 5 атак: крит × 4 |

Визуальное кольцо: `(1 - ultVfx) * 320` — совпадает с реальным радиусом.

---

## Снаряды игрока (`playerBullets`)

**Только визуал** — урон наносится мгновенно в `player.attack()`.  
Скорость: `bulletSpd * 680` px/s. Жизнь: 0.38с.  
**Soft homing**: пока цель жива (`deathT < 0`), снаряд плавно корректирует направление (steer 14×dt).

```js
if (b.targetId && b.targetId.deathT < 0) { /* steer toward target */ }
```

---

## HUD

```css
#hud { opacity: 0; pointer-events: none; }
#hud.hud-visible { opacity: 1; pointer-events: auto; }
```

| Событие | Вызов |
|---------|-------|
| `init()` | `hudHide()` |
| `doRestart()` | `hudShow()` |
| `onDie()` | `hudHide()` |

`#hudCredits`, `#hudCombo` — статические в `#hud`.  
`#adBuffHud`, `#btnAdBuff` — `position:fixed` в `<body>`, **не в `#hud`**.  
`#relicBar` — внутри `#hud`, обновляется в `pickRelic()`.

---

## Пауза

```js
function openPause()  { GameplayAPI.stop(); state = "paused"; cancelAnimationFrame(raf); }
function closePause() { state = "playing"; lastTs = performance.now(); raf = rAF(loop); GameplayAPI.start(); }
```

ESC → toggle. `#pauseOverlay`: `position:fixed; z-index:500`.

---

## Audio

### Музыка меню (HTML Audio)
```js
const MenuMusic = (() => {
    const VOLUME = 0.08;  // тихий фон
    // HTML: <audio id="menuMusic" loop><source src="./track_music.m4a" type="audio/mp4"></audio>
})();
```
Кнопка ON/OFF в настройках (`#btnMusicToggle`). Состояние в localStorage.

### Звуковые эффекты (Web Audio API)
```js
actx._master = actx.createGain();
Audio.setVolume(0..1);
```
Слайдер `#setVolume` в настройках. **Никогда** не вызывать Audio в `drawFrame()`.

---

## Яндекс SDK

```js
GameplayAPI.start()  // при старте игры, возобновлении паузы
GameplayAPI.stop()   // при смерти, паузе, показе рекламы
Ads.showInterstitial(done)           // fullscreen, каждая 2-я смерть
Ads.showRewarded(type, onRew, onClose) // revive / boost / xp / skip_wave
Ads.submitScore(score)               // лидерборд "miniSurvivorsMain"
```

`_adLock = false` **и в `onClose`, и в `onError`** — обязательно.  
`btnAdBuff` паузит игровой цикл перед рекламой, возобновляет в `onClose`.  
Язык: `_ysdk.environment.i18n.lang` → `_currentLang` → `applyLanguage()`.  
Fallback: `_adSim(dur, cb)` — для локальной разработки.

---

## Локализация

| Система | EN поля | RU поля |
|---------|---------|---------|
| SKILLS | `name`, `descFn` | `nameRu`, `descFnRu` |
| SHOP_ITEMS | `title`, `desc` | `titleRu`, `descRu` |
| UPGRADES | `name` | `nameRu` |
| LOCATIONS | `name`, `envDescEn`, `unlockDescEn` | `nameRu`, `envDescRu`, `unlockDescRu` |
| SKINS | `name` | `nameRu` |
| RELICS | `name`, `desc` | `nameRu`, `descRu` |
| DNA_NODES | `name`, `bonus` | `nameRu`, `bonusRu` |

`applyLanguage()` сначала синхронизирует `currentLang = _currentLang`, затем применяет переводы.  
Вызов снаружи: `Game.applyLanguage()`.

---

## История критических багов

| # | Баг | Исправление |
|---|-----|------------|
| 1 | Дублирующий `<script>` | Удалён |
| 2 | `class Enemy {` потеря при str_replace | Восстановлено вручную |
| 3 | `let artifacts` не объявлен | Добавлен в state vars |
| 4 | Камера смещена при DPR | `viewW/viewH` вместо `canvas.width/2` |
| 5 | Overheat убивал всех | `dmg × 3.5` с distance falloff |
| 6 | `world.elapsed` не существует | Заменено на closure `elapsed` |
| 7 | Skin canvas не рендерился | Заменено на inline SVG |
| 8 | `#btnAdBuff` не кликался | Перемещён в `<body>` как `position:fixed` |
| 9 | Магазин на EN при RU | Добавлены `titleRu/descRu` поля |
| 10 | Возрождение не возобновляло игру | Явный `hide("reviveScreen")` в callback |
| 11 | `applyLanguage` не найдена снаружи | Экспортирована из Game IIFE |
| 12 | HUD видно на всех экранах | `hudShow/hudHide`, CSS `opacity:0` default |
| 13 | Нижняя плашка на экране локаций | `#dailyText/#creditText` удалены |
| 14 | Рывки фона при движении | Float `ox/oy`, `Math.round` только при draw |
| 15 | `#dailyModal` кнопки не работали | HTML перемещён до `<script>` |
| 16 | Ult ring 260px ≠ kill range 320px | Ring: `(1 - ultVfx) * 320` |
| 17 | Пули летели мимо движущихся врагов | Soft homing к живой цели |
| 18 | Яд не наносил урон | `Math.round(10*dt)=0` → накопитель `_dmgAcc` |
| 19 | `btnAdBuff` давал бафф без рекламы | Обёрнут в `Ads.showRewarded` с паузой loop |
| 20 | Язык не читался из SDK | `_ysdk.environment.i18n.lang` → `_currentLang` |
| 21 | M4A неверный MIME тип | `audio/mpeg` → `audio/mp4` |

---

## Правила — нельзя нарушать

1. `canvas.width/height` ≠ CSS. Для игры — только `viewW/viewH`.
2. `class Enemy {` — перед всеми подклассами. При str_replace вблизи — включать в оба str.
3. `playerBullets` — только визуал. Не добавлять коллизию без удаления мгновенного урона.
4. `this.projs` для MiniBoss, `enemyBullets` для Gunner. Не смешивать.
5. `_adLock = false` в `onClose` И `onError`.
6. `offerUpgrade()` возвращает из `loop()`. `return` после — намеренный.
7. `applyLanguage()` идемпотентна. Всегда синхронизирует `currentLang = _currentLang` первой.
8. SVG иконки: `stroke="currentColor"`. Цвет через обёртку `style="color:${col}"`.
9. `ctx.setTransform(dpr,0,0,dpr,0,0)` в `resize()`. Не `ctx.scale()`.
10. `#dailyModal` и `#locationScreen` — строго до `<script>`.
11. Tile offset: float `ox/oy`. `Math.round(cx)` перед modulo = рывки фона.
12. HUD — только через `hudShow/hudHide`.
13. DoT урон (яд, аура) — через накопитель `_dmgAcc`, не прямой `Math.round(dmg*dt)`.
14. `GameplayAPI.start()` при старте/resumePause, `GameplayAPI.stop()` при смерти/паузе/рекламе.

---

## Чеклист перед любым изменением

- [ ] `node -e "new Function(script)"` — без синтаксических ошибок
- [ ] Игра стартует без console ошибок
- [ ] Игрок по центру экрана на старте
- [ ] Dash (Space) — один раз за нажатие
- [ ] MiniBoss спавнится на 60с, Phase 2 при 50% HP
- [ ] Upgrade screen: 3 карточки с иконками правильного цвета тира
- [ ] Пауза: ESC и ⏸ → overlay → Resume/Exit работают
- [ ] Громкость: слайдер → Apply → меняет звук
- [ ] Магазин: товары + скины + рекорды, RU локаль полная
- [ ] Локации: карусель бесконечная, фон меняется под локацию
- [ ] HUD: скрыт на меню/магазин/локации, виден только в игре
- [ ] Возрождение: реклама → реально продолжает игру
- [ ] Daily: модал, 3 цели, PLAY запускает матч
- [ ] Фон: плавный скролл без рывков
- [ ] Elite HARDENED: cyan shield bar, поглощает урон
- [ ] Elite OVERCLOCKED: лужа при смерти, наносит урон игроку
- [ ] Relic screen: появляется после босса, 3 карточки, выбор применяется
- [ ] DNA таб в метамагазине: узлы с правильным статусом
- [ ] `doRestart()` сбрасывает: enemies, playerBullets, enemyBullets, artifacts, toxicPuddles, relicPending, relicBar, surgeWindow, adBuffT, isPaused, fpsSmooth, _dprOverride
