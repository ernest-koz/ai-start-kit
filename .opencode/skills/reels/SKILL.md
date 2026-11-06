---
name: reels
description: Сборка вертикального рилса из готовых клипов (телефонные записи, скрин-записи) в Remotion: конверт, сценарий-шотлист, субтитры по речи, музыка, рендер, проверка. Use when the user asks for a рилс/рилсы, reels, shorts, «собрать ролик», вертикальный монтаж, субтитры в видео.
---

# reels: рилс из готовых клипов

Рилс - это монтаж, а не генерация: сырьё уже снято. Ролик делают порядок,
ритм, субтитры и хук. Всё это в Remotion живёт кодом: правки детерминированы,
повторный рендер по тому же таймлайну даёт тот же файл.

## Предпосылки

- Node.js: свежий LTS с nodejs.org (22; для 20.x минимум 20.19, для 22.x -
  22.12+ - на более старых сборка молча откатывается на медленный webpack).
- ffmpeg в PATH (для конверта исходников; сам Remotion тащит свой).
- Клипы-источник: телефонные записи или скрин-записи от ученика.
- Для субтитров агент поставит whisper.cpp сам (пакет `@remotion/install-whisper-cpp`):
  на Windows качается готовый бинарник, на macOS/Linux собирается из исходников
  (нужны git и компилятор). Никаких CLI-команд для whisper нет - только скрипт из шага 4.

## Шаг 1. Конверт исходников (обязательно)

Телефонные записи - переменный framerate и странные цвета. Без конверта будет
рассинхрон субтитров и выцветшая картинка. Каждый клип:

```sh
ffmpeg -y -i ВХОД.mp4 -vf "fps=30,scale=1080:1920:force_original_aspect_ratio=decrease,pad=1080:1920:(ow-iw)/2:(oh-ih)/2" \
  -c:v libx264 -crf 18 -pix_fmt yuv420p -r 30 -c:a aac -ar 48000 ВЫХОД.mp4
```

У горизонтальных исходников pad оставляет чёрные поля слева/справа - это
нормально для рилса. Если ученик просит без полос - вариант crop-to-fill
(кадр обрежется): `scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920`.

Конвертированные складывай в `output/video/src/`, оригиналы не трогай.

## Шаг 2. Сценарий-шотлист до кода

На каждый шот: файл, что происходит, текст-хук (2-4 слова на экране),
длительность. Правила:

- Хук - первый шот: смысл понятен за 2 секунды без звука.
- Ни один шот не длиннее ~3.5 сек (105 кадров @30fps).
- Конец - эндкард с призывом (подписаться / написать в боту).
- Покажи шотлист ученику и утверди до кода.

## Шаг 3. Каркас проекта

Из корня проекта:

```sh
mkdir -p output/video
cd output/video
npx create-video@latest --yes --blank reel
cd reel
npm i @remotion/captions @remotion/install-whisper-cpp
mkdir -p public
cp ../src/*.mp4 public/
```

Лицензия: Remotion бесплатен для команды до 3 человек, дальше - платная
лицензия remotion.pro. Ученику с личным проектом этого достаточно.

Blank-шаблон (template-empty) выглядит так:

```
src/Root.tsx        - обёртка: <MyComposition /> (не трогай)
src/Composition.tsx - регистрация: <Composition id="MyComp" ...> - поменяй id
                      на "Reel", fps={30}, width={1080}, height={1920}
                      и component={Reel} из src/Reel.tsx
src/Reel.tsx        - твой таймлайн: <Sequence from={0} durationInFrames={...}>
```

Каждый шот - `<Sequence>` с `<OffthreadVideo src={staticFile('клип.mp4')}>`
(`staticFile` импортируется из 'remotion', файл лежит в `public/`). Локальные
медиа Remotion отдаёт только из `public/` - поэтому клипы туда и копируются.
Субтитры - `<Series>` из слов (компонент ядра `remotion`). Эндкард - последний
Sequence.

## Шаг 4. Субтитры по речи

Речь в рилсе субтитруется всегда: большинство смотрит в муте.

Whisper в Remotion - программный API (`@remotion/install-whisper-cpp`,
пакет стоит с шага 3), CLI-команд `remotion transcribe` / `remotion
install-whisper-cpp` не существует. Аудио под whisper - строго 16 бит,
16 kHz, mono:

```sh
ffmpeg -y -i output/video/src/клип.mp4 -vn -ac 1 -ar 16000 -acodec pcm_s16le output/video/src/клип.wav
```

Скрипт `output/video/reel/transcribe.mjs`. Все команды шага 4 - из корня
проекта (после шага 3 ты в `output/video/reel`, вернись: `cd ../../..`):

```js
import path from 'node:path';
import {
  installWhisperCpp,
  downloadWhisperModel,
  transcribe,
  toCaptions,
} from '@remotion/install-whisper-cpp';

const to = path.join(process.cwd(), 'output/video/reel/whisper.cpp');
const inputPath = process.argv[2];

await installWhisperCpp({to, version: '1.5.5', printOutput: false}); // Windows: скачает готовый бинарник
await downloadWhisperModel({model: 'base', folder: to, printOutput: false}); // 'base' - мультиязычная, для русского

const {transcription} = await transcribe({
  inputPath,
  whisperPath: to,
  modelFolder: to, // модель в корне whisper.cpp - не полагайся на дефолт transcribe()
  whisperCppVersion: '1.5.5',
  model: 'base',
  tokenLevelTimestamps: true,
  printOutput: false, // иначе прогресс замусорит JSON в stdout
});

const {captions} = toCaptions({whisperCppOutput: transcription});
console.log(JSON.stringify(captions.map((c) => ({text: c.text.trim(), startMs: c.startMs, endMs: c.endMs}))));
```

```sh
node output/video/reel/transcribe.mjs output/video/src/клип.wav > output/video/src/клип.json
```

Полученные слова `{text, startMs, endMs}` рендери по 1-3 слова на экране,
крупно, в безопасной зоне: выше нижних 320-420px (там описание, юзернейм и
аудио-строка Instagram) и не заходи на правые ~120px - там колонка кнопок.

## Шаг 5. Музыка

Правило курса: трендовый звук добавляется в самом Instagram при публикации -
алгоритм продвигает видео под свои звуки. Поэтому в рендер громкую музыку не
записываем: либо тишина, либо тихая подложка без вокала (не перебивает хук),
начинается с первого кадра. Склейки шотов - по битам: подгони `from` шотов под
сетку трека. Музыку даёт ученик (свой файл или лицензионный источник), в
проект не складывай.

## Шаг 6. Рендер

Перед рендером покажи ученику превью: `npx remotion studio` - таймлайн и плеер
в браузере, правки вносятся в таймлайн. Потом рендер.

Из папки проекта (`cd output/video/reel`, если после шага 4 ты в корне):

```sh
npx remotion render Reel out/reel.mp4 --codec h264
```

Первый рендер долгий (компиляция бандла + скачивание браузера) - это
нормально. Whisper считается на шаге 4, при рендере он не запускается.

## Шаг 7. Проверка перед сдачей

- Кадры на каждом шве: нет чёрных вставок и дёрганья.
- Субтитры: не вылезают за экран, синхронны речи, читаются в муте.
- Хук читается за 2 секунды в муте.
- Файл открывается в плеере ученика, вертикаль 1080x1920, faststart.
- Громкость музыки не глушит речь (речь громче музыки).

Правки после первого рендера - только через таймлайн (шаги 2-4), потом
новый рендер. Не правь компоненты, пока таймлайном решается.
