# CLAUDE.md — Mini Survivors: Справочник для ИИ и разработчиков

## Обзор проекта

Браузерный top-down авто-шутер в одном файле `index.html` (~6,600 строк), без сборщика и зависимостей. Единственный внешний скрипт — Яндекс Игры SDK v2. Все системы завёрнуты в один IIFE.

**Цель:** Короткие забеги 3–7 минут с нарастающей сложностью, мета-прогрессия через магазин кредитов, rewarded-реклама Яндекса, полная локализация RU/EN.

---

## Архитектура

### Структура файла

```
index.html
├── <head>
│   ├── Яндекс SDK: <script src="https://yandex.ru/games/sdk/v2">
│   └── CSS (~1,500 строк): все стили включая 7 медиа-брейкпоинтов
├── <body>
│   ├── canvas#gc
│   ├── DOM-экраны: #hud, #startScreen, #upgradeScreen, #shopScreen,
│   │               #deathScreen, #reviveScreen, #settingsScreen,
│   │               #classScreen, #dailyModal
│   └── Fixed-элементы: #adBuffHud, #btnAdBuff, #dashIndicator
└── <script> — единый IIFE
    ├── Константы: SKINS, BIOMES, ARTIFACT_POOL, CLASSES,
    │              UPGRADES, SKILLS, SHOP_ITEMS, ICONS
    ├── Утилиты: LS, $, clamp, dist, fmtTime, todayKey,
    │            mulberry32, hashString, lineBurst
    ├── Яндекс SDK: _ysdk, _adLock, Ads{showInterstitial,
    │              showRewarded, submitScore}, _adSim
    ├── Audio: процедурный Web Audio API
    ├── Input IIFE: K[], mouse, touch, dashPressed()
    ├── Particles, FloatingText, XpOrbs
    ├── Player class
    ├── Enemy → Fast, Ranged, Splitter, Tank, Dasher, Gunner, MiniBoss
    ├── upgradeChoices(), SKILLS, buildSkillChoices(), applySkill()
    ├── I18N, tr()
    ├── UI{update, showDeath, showRevive, waveAlert, toast}
    ├── drawBg()
    ├── Shop: openShop(), renderSkinsSection(), renderLeaderboard(),
    │          makeSkinSvg(), getDailyObjectives(), openDailyModal()
    ├── Game IIFE (const Game = (() => { ... })())
    │   ├── Переменные состояния
    │   ├── Spawn: getSpawnInterval(), classWaveBoost(), spawnEnemy()
    │   ├── loop(), drawFrame(), triggerUltimate(), offerUpgrade()
    │   ├── onDie(), showDeath(), skipRevive(), doRestart()
    │   └── init() + привязка кнопок
    └── applyLanguage()
```

---

## Игровой цикл (`loop(ts)`)

Выполняется через `requestAnimationFrame`. Заблокирован при `state !== "playing"`. Порядок шагов:

1. `dt` (кэп 0.05с)
2. `hitStop` → ранний выход без обновлений
3. `elapsed += dt`, `tickBiome(dt)`, `artifacts.update()`
4. Пассивная регенерация (`player.regenRate`)
5. Проверка surge-окна + таймер спавна
6. Проверка спавна босса (`elapsed >= world.nextBossAt`)
7. `player.update()` → результат атаки
8. Камера: `camX = player.x - viewW/2`, `camY = player.y - viewH/2`
9. Граница мира r=1200: зажим позиции + shake + частицы при ударе
10. Обновление screen shake
11. Цикл врагов: `e.update()`, текст урона, обработка смерти
12. `enemyBullets` (снаряды Gunner)
13. `playerBullets` (только визуал)
14. `orbs.update()` → XP → `player.addXP()` → `offerUpgrade()` при левел-апе
15. Проверка overheat → `triggerUltimate()`
16. `parts.update()`, `texts.update()`
17. Тик ad-баффа (`adBuffT`)
18. `UI.update()`
19. Триггеры wave-алертов (10s, 25s, 45s, 60s, 90s, 120s, 180s, 240s, 300s)
20. Surge-алерты
21. Проверка HP → `onDie()`
22. `drawFrame()`

---

## Рендеринг Canvas

`drawFrame()` — порядок отрисовки (back to front):

1. `ctx.clearRect(0, 0, viewW, viewH)`
2. `drawBg()` — фон, hex-сетка, узлы границы мира
3. `orbs.draw()` — XP-орбы
4. Снаряды врагов (`enemyBullets`)
5. Артефакты (`artifacts.draw()`)
6. Снаряды игрока (`playerBullets`, визуал)
7. Частицы (`parts.draw()`)
8. Враги (`e.draw()`)
9. Игрок: скин, форма, шлейф, орбиталы, комбо-счётчик
10. Плавающие числа (`texts.draw()`)
11. Ult vfx ring
12. Flash overlay
13. Туман мира
14. Комбо flash overlay
15. Panic vignette (HP < 25%)

### КРИТИЧЕСКИ ВАЖНО: Canvas DPR

```js
// В resize():
viewW = innerWidth;
viewH = innerHeight;
canvas.width = Math.round(innerWidth * dpr);
canvas.height = Math.round(innerHeight * dpr);
canvas.style.width = innerWidth + "px";
canvas.style.height = innerHeight + "px";
ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
```

**Никогда** не используйте `canvas.width / 2` для расчёта камеры.  
`canvas.width` — буферный размер (= innerWidth × DPR).  
Для координат игры всегда используйте `viewW` / `viewH`.

---

## Система ввода

`Input` IIFE предоставляет:

- `dir()` → `{dx, dy}` нормализованный вектор движения
- `aim()` → `{dx, dy, active}` направление мыши
- `dashPressed()` → потребляет Space (K["Space"] = false после чтения)
- Touch-джойстик: `tdx`, `tdy` из touch-событий
- `isMobile` — флаг по userAgent

---

## Класс Player

Ключевые свойства:

```js
// Dash (только Scanner)
hasDash, dashCd, dashActive, dashT, dashVx, dashVy;
dashCdMax; // уменьшается при синергии Phase Runner

// Навыки
skillStacks; // { skillId: N } — новая система SKILLS
takenUpgrades; // { id: N } — legacy для UPGRADES

// Боевые множители
armor; // снижение урона, кэп 60% в damage()
regenRate; // HP/сек, применяется каждый тик
xpMult; // множитель XP-дропа
bulletSpd; // множитель скорости playerBullets
multishot; // количество визуальных снарядов за атаку
piercing; // визуальное пробитие врагов

// Overheat
overheatGain; // множитель заряда (базовый 0.65)
overheatMax; // максимум заряда (базовый 100, Firewall 140)
```

**`damage(amount, withIframes=true)`:** проверяет щит → применяет `armor` → устанавливает hitFlash → сбрасывает комбо.

**`attack(enemies, parts, aim)`:** ищет ближайшего врага в `atkR`, наносит урон мгновенно, спавнит `playerBullets` (визуал), устанавливает `atkCd`.

---

## Иерархия врагов

Все враги наследуют `Enemy`. Общие методы:

- `hit(dmg, parts, player, enemies)` → returns killed bool
- `moveToward(player, dt, accel)`
- `drawHpBar(ctx, sx, sy)`

### Таблица типов врагов

| Тип      | Форма          | r   | Спавн | xpVal | Особенность                 |
| -------- | -------------- | --- | ----- | ----- | --------------------------- |
| Enemy    | Шестиугольник  | 16  | 0с    | 12    | Базовый                     |
| Fast     | Треугольник    | 11  | 0с    | 9     | Быстрый                     |
| Ranged   | Ромб           | 14  | 20с   | 18    | Дистанционные снаряды       |
| Splitter | Восьмиугольник | 14  | 50с   | 22    | Делится на 2                |
| Tank     | Шестиугольник  | 28  | 60с   | 55    | HP×4, взрыв r=200           |
| Dasher   | Ромб           | 13  | 80с   | 26    | Рывок каждые 2.5с           |
| Gunner   | Треугольник    | 16  | 100с  | 30    | 3 снаряда, держит дистанцию |
| MiniBoss | Уникальная     | 34  | 60с   | 180×  | 6 вариантов, Phase 2        |

---

## MiniBoss

6 вариантов (bossNum % 6). Конструктор: `(x, y, wave, variant, bossNum)`.

```js
// HP
Math.round((350 + wave * 30) * (1 + Math.min(3.2, bossNum * 0.3)));

// Скорость (enrage ×1.5, кэп 160)
Math.min(108, 68 + bossNum * 5);

// Урон (enrage ×1.4)
11 + wave * 0.3 + bossNum * 0.9;
```

**Phase 2** срабатывает при `!this.enraged && this.hp <= this.maxHp * 0.5`.

**`this.projs`** — пул снарядов босса. Обновляется и проверяет коллизию в `MiniBoss.update()`.  
**Не использовать** глобальный `enemyBullets` для снарядов босса.

---

## Система навыков (SKILLS)

```js
const SKILLS = [
    /* 15 записей */
];
// Каждая запись: { id, tier, icon, name, nameRu, descFn, descFnRu, apply(player, stackN) }
```

### `buildSkillChoices(player)`

1. Фильтр: только навыки с `(skillStacks[id] || 0) < 3`
2. 1 гарантированный high-tier (red/violet)
3. 2 случайных из остатка
4. Возвращает объекты с `currentStack`, `nextStack`, `name`, `desc` (с учётом `currentLang`)

### `applySkill(player, skill)`

Инкрементирует `player.skillStacks[skill.id]`, вызывает `skill.apply(player, nextStack)`.

**Синергии** проверяются в `offerUpgrade()` после `applySkill()`:

```js
const sk = player.skillStacks || {};
if ((sk.critCore||0)>=1 && (sk.lifesteal||0)>=1 && !player.vampireCoreDone) { ... }
```

---

## Экономика

```js
// Кредиты за забег
Math.floor(
    kills * 0.7 +
        elapsed * 0.6 +
        player.lvl * 4 +
        player.comboBest * 2 +
        (orbitals > 0 ? 6 : 0),
);

// Стоимость покупки в магазине
Math.round(item.baseCost * Math.pow(1.5, currentStack));
```

Кредиты хранятся в `META.credits` → `LS.set(META_KEY, META)` после каждой транзакции.

Daily Challenge: ключ `"mini_surv_daily_done_v1"` в localStorage. Один раз за `world.dailyKey` (YYYY-MM-DD).

---

## Яндекс Игры SDK

```js
// Инициализация (async IIFE при загрузке)
_ysdk = await YaGames.init();

// Интерстишиал (экран смерти)
_ysdk.adv.showFullscreenAdv({ callbacks: { onClose, onError } });

// Rewarded (возрождение, буст, XP)
_ysdk.adv.showRewardedVideo({ callbacks: { onRewarded, onClose, onError } });

// Лидерборд
_ysdk
    .getLeaderboards()
    .then((lb) => lb.setLeaderboardScore("miniSurvivorsMain", score));
```

`_adLock` предотвращает параллельные вызовы рекламы. Всегда устанавливайте `_adLock = true` перед вызовом и `_adLock = false` в **обоих** `onClose` и `onError`. Иначе реклама не будет показываться до конца сессии.

Fallback для локальной разработки: `_adSim(duration, callback)`.

---

## История критических исправлений

### 1. Дублирующий `<script>` (SyntaxError + сломанная игра)

**Баг:** Второй `<script>` в конце файла создавал второй canvas, второй `player` и вызывал `loop()` параллельно.  
**Исправление:** Удалён полностью.  
**Правило:** Никогда не добавлять второй `<script>` в этот файл.

### 2. Потеря `class Enemy {` (SyntaxError)

**Баг:** `str_replace` захватил `class Enemy {` как часть old_str, но не включил его в new_str.  
**Исправление:** Вставлено `class Enemy {` перед первым `constructor` класса Enemy.  
**Правило:** При вставке кода рядом с границами класса всегда включайте объявление класса в оба str.

### 3. `let artifacts` не объявлен (ReferenceError)

**Баг:** `artifacts.reset()` и `artifacts.update()` вызывались, но `let artifacts` никогда не был объявлен.  
**Исправление:** Добавлено `let artifacts = new ArtifactSystem();` в блок переменных состояния.

### 4. Камера смещена при DPR-масштабировании

**Баг:** `camX = player.x - canvas.width / 2` использовал буферный размер (например, 3840px на Retina). После `ctx.setTransform(dpr, 0, 0, dpr, 0, 0)` координаты игры в CSS-пикселях, но camX считался в буферных.  
**Исправление:** Введены `let viewW = innerWidth, viewH = innerHeight`, обновляемые в `resize()`. Везде используются `viewW`/`viewH`.

### 5. Overheat убивал всех мгновенно

**Баг:** `triggerUltimate()` вызывал `e.hit(99999, ...)`.  
**Исправление:** `baseDmg = player.dmg * 3.5` с falloff: `1 - (d / (blastR + e.r)) * 0.7`.

### 6. `world.elapsed` в Summoner (ReferenceError)

**Баг:** `Math.floor(world.elapsed / 12)` — у `world` нет свойства `elapsed`.  
**Исправление:** Заменено на `Math.floor(elapsed / 12)` (переменная замыкания).

### 7. Canvas-превью скинов не рендерились

**Баг:** `requestAnimationFrame` срабатывал до добавления canvas в DOM. `document.body.contains(cvs)` возвращал `false` и рендеринг пропускался.  
**Исправление:** Вся система canvas заменена на inline SVG через `makeSkinSvg(skin)`. rAF не нужен.

### 8. `#btnAdBuff` перекрывал контент

**Баг:** `#btnAdBuff` находился внутри `#hud` (z-index: 10) с `pointer-events: none`. Кнопка не работала поверх магазина.  
**Исправление:** Перемещено в прямой дочерний элемент `<body>` с `position: fixed; z-index: 50`.

### 9. Магазин на английском при RU-локали

**Баг:** `SHOP_ITEMS` имел только `title`/`desc` на английском.  
**Исправление:** Добавлены `titleRu`/`descRu` во все 9 товаров. Шаблон: `currentLang==="ru"&&item.titleRu ? item.titleRu : item.title`.

### 10. Возрождение через рекламу не возобновляло игру

**Баг:** Callback устанавливал `state = "playing"`, но `reviveScreen` оставался поверх canvas.  
**Исправление:** Добавлены явный `hide("reviveScreen")`, `player.shieldUsed = false` и toast-уведомление.

---

## Правила проектирования — нельзя нарушать

1. **`canvas.width`/`height` ≠ CSS-размеры.** Использовать `viewW`/`viewH` для всей игровой математики координат.

2. **`class Enemy {` обязан существовать** перед всеми подклассами врагов. При редактировании закрывающей скобки Player убедитесь, что объявление Enemy следует сразу.

3. **`playerBullets` — только визуал.** Реальный урон наносится в `Player.attack()` мгновенно. Не добавляйте коллизию к `playerBullets` без удаления мгновенного урона.

4. **`this.projs` для MiniBoss, `enemyBullets` для Gunner.** Не смешивать пулы.

5. **`_adLock` обязан сниматься.** В `onClose` И в `onError`. Иначе реклама не покажется до конца сессии.

6. **`offerUpgrade()` прерывает `loop()`** через `return`. Выполнение не должно продолжаться к `drawFrame()`.

7. **`applyLanguage()` идемпотентна.** Можно вызвать несколько раз. Не добавлять побочные эффекты внутрь.

8. **SVG-иконки используют `stroke="currentColor"`.** Цвет устанавливается через `<div style="color:${col}">`. Не хардкодить цвета в SVG.

9. **`ctx.setTransform(dpr, 0, 0, dpr, 0, 0)` вместо `ctx.scale()`** в `resize()`. Это сбрасывает матрицу полностью при каждом resize.

10. **Переменные состояния должны быть объявлены через `let`** в блоке состояний (~строки 5350–5370). Если объявить только в `doRestart()` — станут глобалами на `window`.

---

## Руководство по модификациям

### Добавление нового типа врага

```js
class MyEnemy extends Enemy {
    constructor(x, y, wave = 0) {
        super(x, y, wave);
        this.r = 15;
        this.maxHp = 30 + wave * 2;
        this.hp = this.maxHp;
        this.spd = 100 + wave * 2;
        this.dmg = 8 + wave * 0.3;
        this.color = "#ff0000";
        this.glow = "#cc0000";
        this.xpVal = 20;
    }
    update(dt, player, world, enemies, enemyBullets) {
        if (this.deathT >= 0) {
            this.deathT += dt;
            return;
        }
        this.flash = Math.max(0, this.flash - dt);
        // движение / атака
    }
    draw(ctx, cx, cy) {
        const sx = this.x - cx,
            sy = this.y - cy;
        if (
            sx < -80 ||
            sy < -80 ||
            sx > ctx.canvas.width + 80 ||
            sy > ctx.canvas.height + 80
        )
            return;
        // рендеринг
        this.drawHpBar(ctx, sx, sy);
    }
}
```

Затем в `spawnEnemy()` добавить в нужную фазу:

```js
else if (elapsed >= 90 && roll < 0.10) e = new MyEnemy(ex, ey, wave);
```

Добавить цвет смерти в `deathCol` и бонус в `runCredits`.

### Добавление нового навыка

```js
{ id:"mySkill", tier:"blue", icon:"spd",
  name:"MY SKILL",    nameRu:"МОЙ НАВЫК",
  descFn:(s)=>`+${[22,14,9][s-1]}% что-то`,
  descFnRu:(s)=>`+${[22,14,9][s-1]}% что-то`,
  apply(p, s) { p.myStat *= 1 + [0.22,0.14,0.09][s-1]; } },
```

Добавить в массив `SKILLS`. Больше никакой регистрации не нужно.

### Добавление синергии

В `offerUpgrade()` после `applySkill()`:

```js
const sk = player.skillStacks || {};
if (
    (sk.mySkillA || 0) >= 1 &&
    (sk.mySkillB || 0) >= 1 &&
    !player.mySynergyDone
) {
    player.mySynergyDone = true;
    // эффект синергии
    UI.toast(currentLang === "ru" ? "СИНЕРГИЯ: НАЗВАНИЕ ★" : "SYNERGY: NAME ★");
}
```

### Добавление товара в магазин

```js
{
    id: "myItem",
    icon: "dmg",        // ключ из объекта ICONS
    title: "MY ITEM",   titleRu: "МОЙ ТОВАР",
    desc: "Desc.",      descRu: "Описание.",
    baseCost: 60,
    stackable: false,
    apply() { META.unlocked.myFeature = true; },
},
```

Добавить `META.unlocked.myFeature: false` в дефолты META.

### Добавление звука

```js
// В объекте Audio:
mySound() {
    osc(частота, 'square', громкость, атака, спад, конечная_частота);
    noise(громкость, длительность, тип_фильтра, частота_фильтра);
},
```

Вызывать `Audio.mySound()` в нужном игровом событии.  
**Никогда** не вызывать Audio в `drawFrame()`.

### Балансировка волн

- **Ранняя игра (0–30с):** ветки `getSpawnInterval()` для `elapsed < 30`
- **Средняя игра (30–120с):** брейкпоинты в `classWaveBoost()`
- **Поздняя игра (120с+):** ветка `elapsed > 120` в `spawnEnemy()`
- **HP/DMG врагов:** через `hpMul`/`dmgMul` после `e = new EnemyClass(...)`
- **Частота боссов:** `world.nextBossAt += 60` в `spawnBoss()`

---

## Частые ошибки

| Ошибка                                      | Проблема                                                            |
| ------------------------------------------- | ------------------------------------------------------------------- |
| `canvas.width/2` в камере                   | Буферный размер ≠ CSS-размер при DPR > 1. Использовать `viewW`      |
| `world.elapsed`                             | Не существует. Использовать `elapsed` (переменная замыкания)        |
| `innerHTML = ""` после `appendChild`        | Очищает все добавленные узлы. Сначала innerHTML, затем appendChild  |
| Несколько активных `show()` без `hide()`    | Экраны перекрываются. Всегда скрывать предыдущий                    |
| Не сброшен `_adLock` в `onError`            | Реклама не показывается до конца сессии                             |
| Изменение `player` в `drawFrame()`          | Draw вызывается каждый кадр и должен быть без побочных эффектов     |
| rAF-цикл в функции открытия магазина        | Работает параллельно с игровым циклом. Использовать статический SVG |
| Переменная только в `doRestart()` без `let` | Становится глобальной на `window`                                   |

---

## Чеклист перед любым изменением

- [ ] `node --check` показывает отсутствие синтаксических ошибок
- [ ] Игра запускается без ошибок в консоли
- [ ] Игрок появляется по центру экрана (camX = 0 при спавне)
- [ ] Dash Scanner работает (Space срабатывает один раз за нажатие)
- [ ] MiniBoss появляется на 60с, меняет цвет в Phase 2
- [ ] Экран апгрейдов: 3 карточки с SVG-иконками правильных цветов тира
- [ ] Магазин открывается: товары + секция скинов + лидерборд
- [ ] Русская локаль: все тексты магазина, навыков и боссов на русском
- [ ] Английская локаль: нет русского текста
- [ ] Возрождение: просмотр рекламы реально возобновляет игру
- [ ] Daily Challenge: модал открывается, 3 цели, PLAY работает
- [ ] Экран смерти: правильное количество заработанных кредитов
- [ ] `doRestart()`: сбрасывает enemies, playerBullets, enemyBullets, artifacts, surgeWindow, biomeT

---

## Текущие технические пробелы

| Фича                             | Чего не хватает                                  | Сложность |
| -------------------------------- | ------------------------------------------------ | --------- |
| HP-полоса босса вверху           | HTML-элемент оверлея + хук в `UI.update()`       | Низкая    |
| Прогресс Daily Challenge в матче | HUD-элемент с live-отслеживанием целей           | Низкая    |
| Слайдер громкости                | `masterGain` в Audio + слайдер в настройках      | Низкая    |
| Dash на мобайле                  | Двойной тап правой части → `Input.dashPressed()` | Средняя   |
| Элитные враги                    | Prefix-система + спавн в `spawnEnemy()`          | Средняя   |
| Эффекты биомов                   | Чтение `BIOMES[currentBiome].id` в update-циклах | Средняя   |
| Экран достижений                 | DOM-рендер `META.achievements`                   | Низкая    |
| Автоснижение качества            | FPS-монитор + условное `settings.vfx`            | Средняя   |
| Спец. волновые события           | Event pool + активация в `loop()` каждые 120с    | Средняя   |
| Мини-карта                       | Малый canvas-оверлей, позиции боссов/артефактов  | Средняя   |
