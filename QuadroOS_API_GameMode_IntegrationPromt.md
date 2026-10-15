# QuadroOS API GameMode — инструкция для разработчиков

## Цель

Эта инструкция предназначена только для интеграции поддержки **QuadroOS API GameMode** в игру.

После интеграции игра сможет работать с:

- **FastFPS** — передача FPS игры в QuadroOS;
- **FOS Engine** — команда отключения тяжёлого Blur/post-processing;
- **Visual Graphics** — режимы `FPS`, `Visual PLUS`, `Cinema`;
- подтверждением применения настроек через `fos.ack`.

Отдельное приложение или Python-файл для самой игры не требуется: обмен с QuadroOS выполняется через стандартный браузерный `window.postMessage`.

---

## Поддержка FeastOptimizingService: обязательная проверка при запуске

QuadroOS теперь проверяет поддержку FeastOptimizingService сразу после загрузки игры. Игра должна ответить на сообщение:

```js
window.parent.postMessage({
  qosApi: 'QuadroOS',
  version: '1.0',
  protocol: 'FOS Engine',
  action: 'fos.support.response',
  supported: true,
  protocols: ['FastFPS', 'FOS Engine']
}, '*');
```

На запрос QuadroOS:

```js
{
  qosApi: 'QuadroOS',
  version: '1.0',
  protocol: 'FOS Engine',
  action: 'fos.support.request'
}
```

Если игра не отвечает примерно в течение секунды, QuadroOS считает её неподдерживаемой. В окне FeastOptimizingService элементы управления становятся тёмными и недоступными, а отображается сообщение: **«У игры отсутствует поддержка FeastOptimizingService»**.

`supported: false` также явно означает отсутствие поддержки.

# 1. Что нужно добавить в игру

Добавьте один JS-модуль/скрипт в код игры:

```js
const QUADROOS_VERSION = '1.0';

window.addEventListener('message', (event) => {
  const data = event.data;

  if (!data || data.qosApi !== 'QuadroOS') return;
  if (data.qosVersion && data.qosVersion !== QUADROOS_VERSION) return;

  if (data.protocol === 'FOS Engine') {
    handleFOSCommand(data, event);
  }
});
```

---

# 2. FastFPS

## Передача FPS

Игра должна отправлять текущий FPS в родительское окно QuadroOS:

```js
function sendFastFPS(fps) {
  window.parent.postMessage({
    qosApi: 'QuadroOS',
    qosVersion: '1.0',
    protocol: 'FastFPS',
    action: 'fastfps',
    fps: Number(fps)
  }, '*');
}
```

Рекомендуемый интервал:

```js
setInterval(() => {
  sendFastFPS(getCurrentFPS());
}, 500);
```

`getCurrentFPS()` заменяется вашей функцией получения реального FPS.

## Ответ на запрос QuadroOS

QuadroOS также может отправить:

```js
{
  qosApi: 'QuadroOS',
  qosVersion: '1.0',
  protocol: 'FastFPS',
  action: 'fastfps.request'
}
```

Игра должна ответить:

```js
window.addEventListener('message', (event) => {
  const data = event.data;

  if (!data || data.qosApi !== 'QuadroOS') return;
  if (data.protocol !== 'FastFPS') return;
  if (data.action !== 'fastfps.request') return;

  event.source?.postMessage({
    qosApi: 'QuadroOS',
    qosVersion: '1.0',
    protocol: 'FastFPS',
    action: 'fastfps.response',
    fps: Number(getCurrentFPS())
  }, event.origin);
});
```

---

# 3. FOS Engine

При включении FOS Engine QuadroOS отправляет:

```js
{
  qosApi: 'QuadroOS',
  qosVersion: '1.0',
  protocol: 'FOS Engine',
  action: 'fos.enable',
  enabled: true,
  blur: false,
  visual: false,
  mode: 'fps'
}
```

При выключении:

```js
{
  qosApi: 'QuadroOS',
  qosVersion: '1.0',
  protocol: 'FOS Engine',
  action: 'fos.disable',
  enabled: false,
  blur: true
}
```

## Обработчик

```js
function handleFOSCommand(data, event) {
  if (data.action === 'fos.enable') {
    disableHeavyBlur();

    applyVisualMode(
      data.mode || 'fps',
      data.visual === true
    );

    sendFOSAcknowledgement(event, true);
    return;
  }

  if (data.action === 'fos.disable') {
    restoreHeavyBlur();
    sendFOSAcknowledgement(event, false);
    return;
  }

  if (data.action === 'fos.visual') {
    applyVisualMode(data.mode || 'fps', data.enabled === true);
    sendFOSAcknowledgement(event, true);
    return;
  }

  if (data.action === 'fos.visual.mode') {
    applyVisualMode(data.mode || 'fps', data.enabled === true);
    sendFOSAcknowledgement(event, true);
  }
}
```

---

# 4. Отключение Blur

Главная задача FOS Engine — убрать тяжёлый Blur, который создаёт лишнюю нагрузку.

Реализуйте две функции в игровом движке:

```js
function disableHeavyBlur() {
  // Отключить тяжёлый Blur.
  // Отключить ненужные дорогие post-processing passes.
  // Сохранить остальные визуальные эффекты игры.
}

function restoreHeavyBlur() {
  // Вернуть стандартный Blur игры.
}
```

Если Blur сделан через CSS:

```js
document.querySelectorAll('.heavy-blur').forEach(el => {
  el.style.filter = 'none';
  el.style.backdropFilter = 'none';
});
```

Если Blur реализован через WebGL/WebGPU — отключайте соответствующий shader/pass внутри вашего renderer.

---

# 5. Visual Graphics

QuadroOS передаёт выбранный режим через:

```js
{
  qosApi: 'QuadroOS',
  qosVersion: '1.0',
  protocol: 'FOS Engine',
  action: 'fos.visual.mode',
  enabled: true,
  mode: 'fps'
}
```

Допустимые значения:

```text
fps
visualplus
cinema
```

## FPS

Назначение:

**Сделать тёмные детали игры более заметными.**

```js
applyVisualMode('fps', true);
```

Рекомендуется небольшая коррекция brightness/tonemapping без сильного изменения цветов.

## Visual PLUS

Назначение:

**Повысить насыщенность цветов и умеренно увеличить контраст.**

```js
applyVisualMode('visualplus', true);
```

Рекомендуется умеренный saturation + contrast pass.

## Cinema

Назначение:

**Сделать изображение кинематографичным.**

```js
applyVisualMode('cinema', true);
```

Можно использовать собственный лёгкий color grading/post-processing.

---

# 6. Пример Visual Graphics

```js
function applyVisualMode(mode, enabled) {
  if (!enabled) {
    renderer.setColorProfile('default');
    return;
  }

  switch (mode) {
    case 'fps':
      renderer.setColorProfile('fps');
      break;

    case 'visualplus':
      renderer.setColorProfile('visualplus');
      break;

    case 'cinema':
      renderer.setColorProfile('cinema');
      break;

    default:
      renderer.setColorProfile('default');
  }
}
```

`renderer.setColorProfile()` — пример. Замените его вашей системой рендера.

---

# 7. Подтверждение FOS Engine

После применения команды желательно отправлять QuadroOS подтверждение:

```js
function sendFOSAcknowledgement(event, enabled) {
  event.source?.postMessage({
    qosApi: 'QuadroOS',
    qosVersion: '1.0',
    protocol: 'FOS Engine',
    action: 'fos.ack',
    enabled: enabled,
    blurDisabled: enabled
  }, event.origin);
}
```

---

# 8. Полный минимальный адаптер

Этот блок можно практически целиком добавить в игру:

```js
const QUADROOS_VERSION = '1.0';

function sendFastFPS(fps) {
  window.parent.postMessage({
    qosApi: 'QuadroOS',
    qosVersion: QUADROOS_VERSION,
    protocol: 'FastFPS',
    action: 'fastfps',
    fps: Number(fps)
  }, '*');
}

function disableHeavyBlur() {
  // TODO: отключить тяжёлый Blur/post-processing игры
}

function restoreHeavyBlur() {
  // TODO: вернуть стандартный Blur
}

function applyVisualMode(mode, enabled) {
  // TODO: подключить вашу систему color grading
  console.log('QuadroOS Visual Graphics:', enabled, mode);
}

function sendFOSAcknowledgement(event, enabled) {
  event.source?.postMessage({
    qosApi: 'QuadroOS',
    qosVersion: QUADROOS_VERSION,
    protocol: 'FOS Engine',
    action: 'fos.ack',
    enabled,
    blurDisabled: enabled
  }, event.origin);
}

window.addEventListener('message', (event) => {
  const data = event.data;

  if (!data || data.qosApi !== 'QuadroOS') return;
  if (data.qosVersion && data.qosVersion !== QUADROOS_VERSION) return;

  if (data.protocol === 'FastFPS') {
    if (data.action === 'fastfps.request') {
      event.source?.postMessage({
        qosApi: 'QuadroOS',
        qosVersion: QUADROOS_VERSION,
        protocol: 'FastFPS',
        action: 'fastfps.response',
        fps: Number(getCurrentFPS())
      }, event.origin);
    }

    return;
  }

  if (data.protocol !== 'FOS Engine') return;

  if (data.action === 'fos.enable') {
    disableHeavyBlur();
    applyVisualMode(data.mode || 'fps', data.visual === true);
    sendFOSAcknowledgement(event, true);
    return;
  }

  if (data.action === 'fos.disable') {
    restoreHeavyBlur();
    applyVisualMode('default', false);
    sendFOSAcknowledgement(event, false);
    return;
  }

  if (data.action === 'fos.visual' || data.action === 'fos.visual.mode') {
    applyVisualMode(data.mode || 'fps', data.enabled === true);
    sendFOSAcknowledgement(event, true);
  }
});

setInterval(() => {
  sendFastFPS(getCurrentFPS());
}, 500);
```

---

# 9. Что должен предоставить разработчик

Перед публикацией игры необходимо проверить:

- [ ] `getCurrentFPS()` возвращает настоящий FPS;
- [ ] игра отвечает на `fastfps.request`;
- [ ] игра регулярно отправляет `FastFPS`;
- [ ] `fos.enable` отключает тяжёлый Blur;
- [ ] `fos.disable` возвращает стандартные эффекты;
- [ ] `fps` работает;
- [ ] `visualplus` работает;
- [ ] `cinema` работает;
- [ ] отправляется `fos.ack`;
- [ ] обработчик не ломает обычные `postMessage` игры.

---

# 10. Важно про FastAPI

Если разработчики используют **FastAPI на сервере**, GameMode можно дополнительно связать с backend.

Но **FastFPS и FOS Engine не требуют отдельного Python-сервера для самого обмена между QuadroOS и HTML-игрой**.

Основной обязательный интерфейс интеграции — `postMessage`.

FastAPI нужен только если разработчикам требуется серверная часть: например, хранение статистики FPS, профилей GameMode или состояния игры.

Пример серверной схемы может быть добавлен отдельно, но для базовой поддержки QuadroOS GameMode он не нужен.

---

# 11. Безопасность

В production желательно заменить `'*'` на origin QuadroOS:

```js
const QUADROOS_ORIGIN = 'https://your-quadroos-domain.example';

window.parent.postMessage(payload, QUADROOS_ORIGIN);
```

И проверять:

```js
if (event.origin !== QUADROOS_ORIGIN) return;
```

Также проверяйте `mode` по whitelist:

```js
if (!['fps', 'visualplus', 'cinema'].includes(data.mode)) return;
```
