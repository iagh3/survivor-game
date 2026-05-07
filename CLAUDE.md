# CLAUDE.md — Mini Survivors: Системный справочник (актуально)

> Файл: `index.html` — ~8,050 строк, один IIFE, без сборщика, без зависимостей.
> Единственный внешний скрипт — Яндекс Игры SDK v2.

---

## Структура файла

```
index.html
├── <head>
│   ├── <script src="https://yandex.ru/games/sdk/v2">
│   └── <style> — весь CSS (~1,800 строк)
│       ├── Базовые стили, экраны, HUD, кнопки
│       ├── Upgrade/Class/Shop/Death/Revive/Settings экраны
│       ├── Location carousel (ls-*)
│       ├── Pause overlay (pause-*)
│       └── Media queries: 320/480/768/1024/1280/1440/1920px
├── <body>
│   ├── canvas#gc
│   ├── #hud (opacity:0 по умолчанию, .hud-visible во время игры)
│   │   ├── #btnPause (⏸, по центру сверху)
│   │   ├── #hud-left (HP/XP/Overheat бары)
│   │   ├── #hud-right (время, уровень, убийства, цель)
│   │   └── #hudCredits, #hudCombo (статические элементы)
│   ├── #adBuffHud, #btnAdBuff (position:fixed на body, z-index:50)
│   ├── Все .screen оверлеи (show/hide через classList.add/remove "active")
│   ├── #dailyModal, #locationScreen  ← ОБЯЗАНЫ быть ДО <script>
│   ├── #pauseOverlay (position:fixed, z-index:500)
│   └── <script> — единый IIFE
│       ├── Константы: SKINS, LOCATIONS, CLASSES, UPGRADES, SKILLS, SHOP_ITEMS, ICONS
│       ├── Утилиты: LS, $, clamp, dist, fmtTime, todayKey, mulberry32, hashString
│       ├── Яндекс SDK: _ysdk, _adLock, Ads{showInterstitial, showRewarded, submitScore}
│       ├── Audio: masterGain, osc(), noise(), 10 звуков + setVolume()
│       ├── Input IIFE: K[], mouse, touch joystick, dashPressed()
│       ├── Particles, FloatingText, XpOrbs, ArtifactSystem
│       ├── Bg state: _fireBuf, _voroSeeds, _voroCache, _flowParticles
│       ├── drawBg() + drawTiles() внутри
│       ├── Player, Enemy, Fast, Ranged, Splitter, Tank, Dasher, Gunner, MiniBoss
│       ├── upgradeChoices(), buildSkillChoices(), applySkill()
│       ├── I18N, tr(), show/hide, hudShow/hudHide, UI{}
│       ├── Shop, Daily, Location carousel, Skin preview (SVG)
│       ├── Game IIFE: loop, drawFrame, onDie, doRestart, openPause, closePause
│       └── Game.applyLanguage()  ← вызов снаружи
```

---

## Игровой цикл (`loop(ts)`)

Порядок шагов (только при `state === "playing"`):

1. `dt` (кэп 0.05с)
2. `hitStop` → ранний return
3. `elapsed += dt`; `artifacts.update()`
4. Пассивная регенерация
5. Опасности локации (`hazard:"fire"` → урон 8 каждые 15с; `"freeze"` → `player._freezeT=3`)
6. Surge-окно + таймер спавна → `spawnEnemy()`
7. Проверка босса → `spawnBoss()`
8. `player.update()` → результат атаки
9. Камера: `camX = player.x - viewW/2`, `camY = player.y - viewH/2`
10. Граница мира r=1200 + shake
11. Цикл врагов: `e.update()`, текст урона, смерть
12. `enemyBullets` (Gunner), `playerBullets` (визуал)
13. `orbs.update()` → XP → `offerUpgrade()` при level-up
14. Overheat → `triggerUltimate()`
15. `parts.update()`, `texts.update()`, ad-бафф тик
16. `UI.update()`
17. Wave-алерты (10/25/45/60/90/120/180/240/300с)
18. HP < 0 → `onDie()`
19. `drawFrame()`

---

## Рендеринг (`drawFrame()`)

Back-to-front:
1. `clearRect(0, 0, viewW, viewH)`
2. `drawBg()` — тайлы + процедурный паттерн
3. Орбы XP
4. `enemyBullets`, `artifacts.draw()`, `playerBullets`
5. `parts.draw()`
6. Враги, игрок (скин/форма/шлейф/орбиталы/комбо-счётчик)
7. `texts.draw()`
8. Ult vfx, flash overlay, туман мира, panic vignette

### КРИТИЧНО: Canvas DPR

```js
// resize():
viewW = innerWidth; viewH = innerHeight;
canvas.width  = Math.round(innerWidth  * dpr);
canvas.height = Math.round(innerHeight * dpr);
ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
```

**Никогда** `canvas.width/2` для камеры. Только `viewW/viewH`.

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
const ox = ((-icx % tW) + tW) % tW;
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

Все мировые координаты → экранные через `Math.round(wx - cx)`.  
При смене локации: `_bgReset(locId)` сбрасывает state.

---

## Система локаций

### 4 локации

| ID | Разблокировка | HP врагов | SPD врагов | XP | Кредиты | Хазард |
|----|--------------|-----------|------------|-----|---------|--------|
| `server` | Сразу | ×1.0 | ×1.0 | ×1.0 | ×1.0 | нет |
| `core` | Выжить 3 мин | ×1.25 | ×1.15 | ×1.3 | ×1.25 | огонь 15с |
| `cryo` | Level 10 | ×1.5 | ×0.75 | ×1.5 | ×1.4 | заморозка 20с |
| `void` | 200 убийств | ×1.8 | ×1.3 | ×2.0 | ×1.8 | нет |

Модификаторы применяются в `spawnEnemy()` после создания врага.  
`xpMul` — при дропе орба. `creditMul` — при подсчёте кредитов после смерти.  
`checkLocationUnlocks()` вызывается после каждого забега.

### Карусель

- `_lsIdx` — текущий индекс, бесконечный: `(_lsIdx ± 1 + N) % N`
- `_lsRender()` — полная перерисовка всех слайдов + dots
- `_lsUpdate()` — translateX трека, цвет dots, фон экрана
- Стрелки вешаются в `openLocationScreen()` каждый раз заново

**DOM:** `#locationScreen` и `#dailyModal` — строго **до** `<script>`.

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

`#hudCredits`, `#hudCombo` — статические в `#hud`, обновляются в `UI.update()`.  
`#adBuffHud`, `#btnAdBuff` — `position:fixed` в `<body>`, **не в `#hud`**.

---

## Пауза

```js
let isPaused = false;
function openPause()  { state = "paused"; cancelAnimationFrame(raf); }
function closePause() { state = "playing"; lastTs = performance.now(); raf = rAF(loop); }
```

ESC → toggle. `lastTs = performance.now()` при closePause — нет прыжка dt.  
`#pauseOverlay`: `position:fixed; z-index:500; backdrop-filter:blur(8px)`.

---

## Audio

```js
// Цепочка: osc/noise → gain → compressor → _master → destination
actx._master = actx.createGain(); // создаётся при первом вызове ac()
Audio.setVolume(0..1);  // masterGain.value
```

Слайдер `#setVolume` (0-100) в настройках → Apply → `Audio.setVolume(val/100)`.  
**Никогда** не вызывать Audio в `drawFrame()`.

---

## Скины

```js
// shape: hex | diamond | triangle | star | cross | ring
// trail: dots | spark | spiral | pulse | chain
// orbit: ellipse | square | triangle | none
```

`makeSkinSvg(skin)` → статичный inline SVG (без rAF, без canvas). Используется в магазине.  
`getActiveSkin()` → `SKINS.find(s => s.id === META.activeSkin)`.

---

## Контактный урон (contactCd)

| Тип | КД | Урон | `withIframes` |
|-----|-----|------|--------------|
| Enemy | 0.28с | dmg × 1.0 | true (0.4с защиты) |
| Fast | 0.22с | dmg × 0.9 | true |
| Tank | 0.38с | dmg × 1.2 | true |
| Dasher | 0.25с | dmg × 0.9 | true |

Push-back: `player.x -= nx * overlap * 0.5` — нельзя зайти внутрь врага.  
Инициализация: `if (!this.contactCd) this.contactCd = 0;` в каждом update().

---

## Яндекс SDK

```js
Ads.showInterstitial(done)     // fullscreen, каждая 2-я смерть
Ads.showRewarded(type, onRew)  // revive / boost / xp
Ads.submitScore(score)         // лидерборд "miniSurvivorsMain"
```

`_adLock = false` **и в `onClose`, и в `onError`** — обязательно.  
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

`applyLanguage()` — внутри Game IIFE, экспортируется: `return { start: doRestart, applyLanguage }`.  
Вызов снаружи: `Game.applyLanguage()`.

---

## История критических багов

| # | Баг | Исправление |
|---|-----|------------|
| 1 | Дублирующий `<script>` — второй canvas + параллельный loop | Удалён |
| 2 | `class Enemy {` потеря при str_replace | Восстановлено вручную |
| 3 | `let artifacts` не объявлен | Добавлен в state vars |
| 4 | Камера смещена при DPR | `viewW/viewH` вместо `canvas.width/2` |
| 5 | Overheat убивал всех | `dmg × 3.5` с distance falloff |
| 6 | `world.elapsed` не существует | Заменено на closure `elapsed` |
| 7 | Skin canvas не рендерился (timing) | Заменено на inline SVG |
| 8 | `#btnAdBuff` не кликался | Перемещён в `<body>` как `position:fixed` |
| 9 | Магазин на EN при RU | Добавлены `titleRu/descRu` поля |
| 10 | Возрождение не возобновляло игру | Явный `hide("reviveScreen")` в callback |
| 11 | `applyLanguage` не найдена снаружи | Экспортирована из Game IIFE |
| 12 | HUD видно на всех экранах | `hudShow/hudHide` по state, CSS `opacity:0` default |
| 13 | Нижняя плашка на экране локаций | `#dailyText/#creditText` удалены, `hudHide()` при menu |
| 14 | Рывки фона при движении | Float `ox/oy`, `Math.round` только при draw, `+0.5` для линий |
| 15 | `#dailyModal` кнопки не работали | HTML перемещён **до** `<script>` |

---

## Правила — нельзя нарушать

1. `canvas.width/height` ≠ CSS. Для игры — только `viewW/viewH`.
2. `class Enemy {` — перед всеми подклассами. При str_replace вблизи — включать в оба str.
3. `playerBullets` — только визуал. Не добавлять коллизию без удаления мгновенного урона.
4. `this.projs` для MiniBoss, `enemyBullets` для Gunner. Не смешивать.
5. `_adLock = false` в `onClose` И `onError`.
6. `offerUpgrade()` возвращает из `loop()`. `return` после — намеренный.
7. `applyLanguage()` идемпотентна.
8. SVG иконки: `stroke="currentColor"`. Цвет через обёртку `style="color:${col}"`.
9. `ctx.setTransform(dpr,0,0,dpr,0,0)` в `resize()`. Не `ctx.scale()`.
10. `#dailyModal` и `#locationScreen` — строго до `<script>`.
11. Tile offset: float `ox/oy`. `Math.round(cx)` перед modulo = рывки фона.
12. HUD — только через `hudShow/hudHide`.

---

## Добавление новых элементов

### Враг

```js
class MyEnemy extends Enemy {
    constructor(x, y, wave = 0) {
        super(x, y, wave);
        this.r = 15; this.maxHp = 30 + wave*2; this.hp = this.maxHp;
        this.spd = 100; this.dmg = 8; this.xpVal = 20;
        this.color = "#ff0000"; this.glow = "#cc0000";
        this.contactCd = 0; // обязательно
    }
    update(dt, player, world, enemies, enemyBullets) {
        if (this.deathT >= 0) { this.deathT += dt; return; }
        this.flash = Math.max(0, this.flash - dt);
        this.contactCd = Math.max(0, this.contactCd - dt);
        this.moveToward(player, dt); // использует contactCd внутри
    }
}
// В spawnEnemy(): добавить в нужную фазу
// В deathCol switch: цвет вспышки
```

### Навык

```js
{ id:"mySkill", tier:"blue", icon:"spd",
  name:"NAME", nameRu:"ИМЯ",
  descFn:(s)=>`+${[22,14,9][s-1]}%`,
  descFnRu:(s)=>`+${[22,14,9][s-1]}%`,
  apply(p, s) { p.myStat *= 1 + [0.22,0.14,0.09][s-1]; }
}
```

### Товар магазина

```js
{ id:"myItem", icon:"dmg",
  title:"NAME", titleRu:"ИМЯ",
  desc:"Desc.", descRu:"Описание.",
  baseCost: 60, stackable: false,
  apply() { META.unlocked.myFeature = true; }
}
// META: добавить myFeature:false в defaults
```

---

## Чеклист перед любым изменением

- [ ] `node --check` — без синтаксических ошибок
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
- [ ] `doRestart()` сбрасывает: enemies, playerBullets, enemyBullets, artifacts, surgeWindow, biomeT, adBuffT, isPaused

---

## Технические пробелы

| Фича | Сложность |
|------|----------|
| HP-полоса босса вверху экрана | Низкая |
| Dash на мобайле (double-tap) | Средняя |
| Прогресс Daily в матче (live HUD) | Низкая |
| Заморозка замедляет движение игрока | Низкая |
| Элитные враги с prefix | Средняя |
| Экран достижений | Низкая |
| Auto-quality scaling (FPS < 30) | Средняя |
