# Gym-Boy 3000

Retro-terminal gym workout logger for Android.

[🇬🇧 English](#english) · [🇨🇳 中文](#中文) · [🇷🇺 Русский](#русский)

## English

Personal Android app for logging gym workouts. The style is the Pip-Boy from Fallout: a phosphor-green terminal, a monospaced font, panel frames, CRT scanlines.

### Download

Signed APK: [Releases](https://github.com/sneeb123/gym-boy-3000/releases/latest) — download the APK, allow installing from unknown sources (the app is free, MIT-licensed; releases are published in this repository, the source code is developed in a private one).

### Concept

The workout program is composed **not** on the phone: it is generated externally (a neural network / script / chat-bot) as a JSON file and delivered to the phone. The main channel is automatic sync with a private GitHub repository (see «Cloud» below); a program file can also be opened from a messenger or file manager («Open with Gym-Boy 3000») — it joins the plan the same way. During a workout you fill in only two fields: **actual reps** and **pulse**. A finished workout is uploaded to the cloud with one tap — you decide the moment — and is analyzed externally: the next program is composed from it. A closed loop.

### Workout screen

- **Header**: date, workout number, **START** button. On press it turns into a stopwatch; pressing again records the workout time.
- **Paging through workouts**: swipe or the `‹ 02/06 ›` arrows switch between records chronologically — the archive (oldest first), the active workout, then the queued programs marked `// PLAN` (the past is on the left). The app opens on the nearest unfinished workout; START can be pressed on any planned page — the rest keep waiting.
- **Entry pad**: tapping ACTUAL or PULSE opens a compact numeric pad instead of the system keyboard, so the sets table stays fully visible. The planned value is prefilled as a draft, «NEXT» confirms it and walks set by set, and the pad closes itself after the last set.
- **Upload arrow**: a finished workout waits for the ↑ button in the pager bar — the arrow appears only on its page (the workout must be started and stopped, and the cloud must be linked), archive pages have none. Pressing it uploads the result to the cloud and turns the page into an archive entry. Nothing is uploaded without an explicit tap.
- **Pulse hints**: the pulse of the last workout that had the same exercise (matched set by set) shows up as dim amber in the current set's PULSE cell — to watch how the body adapts to the load.
- **Language**: the button in the footer cycles English → 中文 → Русский (English is the default; the choice is remembered on the phone).
- **Exercise tabs** with a sets table:

| # | PLATES | TOTAL | PLAN | ACTUAL | PULSE |
|---|--------|-------|------|--------|-------|
| 1 | 40     | 60    | 8    | 8      | 125   |
| 2 | 50     | 70    | 8    | 7      | 138   |

Only ACTUAL and PULSE are editable. «Plates» is the weight of the discs on the bar, «Total» = plates + bar (the bar weight is set per exercise, 20 kg by default).

### Data formats

#### Program (input, `gymboy-program/1`)

The only required fields are `exercises` (at least one) and each exercise's `name`; everything else can be omitted. Defaults: `bar_weight_kg` — 20 kg; `total_weight_kg` is computed as plates + bar when omitted; `set_number` is assigned automatically and should not be supplied. `workout_number` and `date` identify the workout (the plan queue is sorted by them); `notes` and `planned_reps` are optional. The `schema` value is not validated — keep `gymboy-program/1` for clarity. Unknown keys are ignored.

```json
{
  "schema": "gymboy-program/1",
  "workout_number": 42,
  "date": "2026-10-23",
  "notes": "Leg focus, 3 min rest between sets",
  "exercises": [
    {
      "name": "Squat",
      "bar_weight_kg": 20.0,
      "sets": [
        { "plates_weight_kg": 40.0, "planned_reps": 8 },
        { "plates_weight_kg": 50.0, "planned_reps": 8 },
        { "plates_weight_kg": 60.0, "planned_reps": 8 },
        { "plates_weight_kg": 60.0, "planned_reps": 6 }
      ]
    }
  ]
}
```

#### Result (output, `gymboy-result/1`)

The same plus timing and actual values:

```json
{
  "schema": "gymboy-result/1",
  "workout_number": 42,
  "date": "2026-10-23",
  "started_at": "2026-10-23T18:03:11.240Z",
  "finished_at": "2026-10-23T19:07:45.110Z",
  "duration_seconds": 3873,
  "exercises": [
    {
      "name": "Squat",
      "bar_weight_kg": 20.0,
      "sets": [
        {
          "set_number": 1,
          "plates_weight_kg": 40.0,
          "total_weight_kg": 60.0,
          "planned_reps": 8,
          "actual_reps": 8,
          "pulse": 125
        }
      ]
    }
  ]
}
```

### Cloud (GitHub)

A closed loop without manual file shuffling: a private GitHub repository acts as a mailbox between the PC (which composes programs) and the phone.

```
programs/pending/    ← new programs (put there by the PC/neural network)
programs/imported/   ← accepted programs (moved by the app itself)
results/             ← results archive (uploaded by the app)
```

Behavior:

- The exchange runs at app start, every time you return to the app from the background (at most once every 3 minutes), right after a workout upload, and on demand — tap the «⇅ LINK» badge in the header.
- Every new program in `pending` joins the plan queue — none is lost; the taken files are moved to `imported`.
- A workout in progress is never touched by an exchange: new programs wait in the plan queue for their turn.
- Results sync both ways: local edits upload, external edits (web editor) download; on conflict the phone's version wins.
- The reverse path works too: results from the cloud that are missing on the phone are downloaded into the local archive — after reinstalling the app or on a new phone, the history comes back by itself at the first exchange.
- Privacy: the app checks whether the repository is private (the `private` flag) and warns if it is not — the badge shows amber «LINK: PUBLIC», the comm panel shows a warning. The exchange is not blocked: the decision is the user's.

#### Setup — quick sign-in (the main path)

The **«Sign in with GitHub»** button in the comm terminal: the app shows a one-time code (it is copied to the clipboard automatically), you enter it at `github.com/login/device`, and on the same page GitHub offers to install the public **Gym-Boy 3000** app — grant it access to your private `gymboy-data` repository. That is the only repository the app can see (repository contents read/write); the session token is stored only on the phone and is renewed silently in the background.

If the repository does not exist yet, create a private one named `gymboy-data`: the app's panel has direct buttons — **App installs** (opens the installation page), **Create repo** (opens github.com/new) and **Recheck**. No registration on the GitHub developer-settings side is needed: the GitHub App is baked into the build.

A manual fine-grained PAT setup remains as a fallback (see below).

#### Manual setup (PAT, if the repository already exists)

1. Create a private repository (e.g. `gymboy-cloud`) with the folders `programs/pending`, `programs/imported`, `results` (at least with `.gitkeep`).
2. Create a fine-grained token: GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token; Repository access — only this repository; Permissions → Contents: Read and write.
3. In the app: tap «⇅ LINK» → «MANUAL SETUP (PAT)» → enter owner, repository and token → «Connect». The token is stored only on the phone (SharedPreferences).

#### Web editor (online)

Tabular editing of cloud files without git and JSON:
**https://sneeb123.github.io/gym-boy-3000-editor/** — a static page from the public repository [gym-boy-3000-editor](https://github.com/sneeb123/gym-boy-3000-editor). There is no server: everything runs in the browser; sign-in is a fine-grained PAT for your data repository (Contents: Read and write); the token is stored in the browser's localStorage and is sent only to api.github.com. Source — the [gym-boy-3000-editor](https://github.com/sneeb123/gym-boy-3000-editor) repository. The phone picks up the edits at the next sync.

## 中文

个人 Android 健身房训练记录应用。风格仿照《辐射》（Fallout）中的 Pip-Boy（哔哔小子）：磷光绿终端、等宽字体、面板边框、CRT 扫描线。

### 下载

签名 APK：[Releases](https://github.com/sneeb123/gym-boy-3000/releases/latest) — 下载 APK，允许"安装未知来源应用"即可（免费、MIT 许可；版本发布在本仓库，源码在私有仓库中开发）。

### 概念

训练计划**不是**在手机上编写的：它由外部（神经网络 / 脚本 / 聊天机器人）生成为 JSON 文件并送到手机。主要通道是与私有 GitHub 仓库的自动同步（见下方「云端」）；也可以从即时通讯应用或文件管理器直接打开计划文件（“打开方式”→ Gym-Boy 3000），它会同样加入计划。训练时只需填写两个字段：**实际次数**和**心率**。完成的训练一键上传到云端——时机由你决定——外部据此进行分析并编写下一个计划。闭环。

### 训练屏幕

- **页眉**：日期、训练编号、**开始**按钮。按下后变成秒表，再次按下记录训练用时。
- **记录翻页**：滑动或顶部的 `‹ 02/06 ›` 箭头按时间顺序切换记录——存档（旧→新）、当前训练、然后是标有 `// 计划` 的排队训练（过去在左边）。应用打开最近一条未完成的训练；任何计划页都可以按「开始」——其余继续排队。
- **数字面板**：点击「实际」或「心率」会打开紧凑的数字面板而不是系统键盘，表格因此保持完整可见。计划值会作为草稿预填，「下一步」逐组确认，最后一组后面板自动关闭。
- **上传箭头**：完成的训练等待翻页栏中的 ↑ 按钮——箭头只出现在它的页面上（训练必须已开始并已停止，且云已连接），存档页没有。按下后结果上传到云端，页面转为存档。没有你的点击，什么都不会上传。
- **心率提示**：上一次含同一动作的训练的心率（逐组对应）以暗琥珀色显示在当前的「心率」格中——观察身体对负荷的适应。
- **语言**：页脚按钮在 English → 中文 → Русский 之间循环切换（默认英语；选择保存在手机上）。
- **动作标签页**，每组一行的表格：

| # | 杠铃片 | 总计 | 计划 | 实际 | 心率 |
|---|--------|------|------|------|------|
| 1 | 40     | 60   | 8    | 8    | 125  |
| 2 | 50     | 70   | 8    | 7    | 138  |

只有「实际」和「心率」可编辑。「杠铃片」是杠铃上配重片的重量，「总计」= 杠铃片 + 杆重（杆重在动作级别设置，默认 20 公斤）。

### 数据格式

#### 计划（输入，`gymboy-program/1`）

计划中只有 `exercises`（至少一个）和每个练习的 `name` 是必填的；其余字段都可以省略。默认值：`bar_weight_kg` — 20 公斤；省略 `total_weight_kg` 时按杠铃片 + 杆重计算；`set_number` 自动分配，无需提供。`workout_number` 和 `date` 标识训练（计划队列按它们排序）；`notes` 和 `planned_reps` 可选。`schema` 的值不做校验——为清晰起见保留 `gymboy-program/1`。未知键会被忽略。

```json
{
  "schema": "gymboy-program/1",
  "workout_number": 42,
  "date": "2026-10-23",
  "notes": "腿部重点，组间休息 3 分钟",
  "exercises": [
    {
      "name": "深蹲",
      "bar_weight_kg": 20.0,
      "sets": [
        { "plates_weight_kg": 40.0, "planned_reps": 8 },
        { "plates_weight_kg": 50.0, "planned_reps": 8 },
        { "plates_weight_kg": 60.0, "planned_reps": 8 },
        { "plates_weight_kg": 60.0, "planned_reps": 6 }
      ]
    }
  ]
}
```

#### 结果（输出，`gymboy-result/1`）

同样内容再加上时间和实际值：

```json
{
  "schema": "gymboy-result/1",
  "workout_number": 42,
  "date": "2026-10-23",
  "started_at": "2026-10-23T18:03:11.240Z",
  "finished_at": "2026-10-23T19:07:45.110Z",
  "duration_seconds": 3873,
  "exercises": [
    {
      "name": "深蹲",
      "bar_weight_kg": 20.0,
      "sets": [
        {
          "set_number": 1,
          "plates_weight_kg": 40.0,
          "total_weight_kg": 60.0,
          "planned_reps": 8,
          "actual_reps": 8,
          "pulse": 125
        }
      ]
    }
  ]
}
```

### 云端（GitHub）

无需手动搬运文件的闭环：一个私有 GitHub 仓库充当电脑（编写计划的一方）与手机之间的邮箱。

```
programs/pending/    ← 新计划（由电脑/神经网络放入）
programs/imported/   ← 已接受的计划（应用自动移入）
results/             ← 结果存档（应用上传）
```

行为：

- 交换在应用启动时、每次从后台返回应用时（最多每 3 分钟一次）、训练上传后立即，以及手动触发——点击页眉的「⇅ 通讯」指示器。
- `pending` 中的每个新计划都会加入计划队列——不会丢失；取走的文件移入 `imported`。
- 进行中的训练不会被交换触碰：新计划在队列中等待自己的顺序。
- 结果双向同步：本地修改上传，外部修改（网页编辑器）下载；冲突时以手机版本为准。
- 反方向同样可用：云端有而手机没有的结果会下载到本地存档——重装应用或换新手机后，历史记录会在第一次交换时自动恢复。
- 隐私：应用会检查仓库是否为私有（`private` 标志），若不是则发出提醒——指示器显示琥珀色「通讯：公开」，通讯面板显示警告。此时交换不会被阻止：决定权在用户。

#### 设置——快速登录（主要方式）

通讯终端中的**「通过 GitHub 登录」**按钮：应用显示一次性代码（已自动复制到剪贴板），你在 `github.com/login/device` 输入它，同一页面上 GitHub 会提议安装公开的 **Gym-Boy 3000** 应用——授权它访问你的私有仓库 `gymboy-data`。这是应用唯一能看到的仓库（仓库内容读写权限）；会话令牌只保存在手机上，并在后台静默续期。

如果仓库还不存在——创建名为 `gymboy-data` 的私有仓库：应用面板有直达按钮：「应用安装页」（打开安装页面）、「创建仓库」（打开 github.com/new）和「重新检查」。无需在 GitHub 开发者设置中注册任何东西：GitHub 应用已编译进安装包。

手动 fine-grained PAT 设置保留为备用（见下文）。

#### 手动设置（PAT，若仓库已存在）

1. 创建私有仓库（例如 `gymboy-cloud`），包含 `programs/pending`、`programs/imported`、`results` 文件夹（至少有 `.gitkeep`）。
2. 创建细粒度令牌：GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token；Repository access——仅此仓库；Permissions → Contents：Read and write。
3. 在应用中：点击「⇅ 通讯」→「手动设置 (PAT)」→ 填写所有者、仓库和令牌 →「连接」。令牌只保存在手机上（SharedPreferences）。

#### 网页编辑器（在线）

无需 git 和 JSON，以表格方式编辑云端文件：
**https://sneeb123.github.io/gym-boy-3000-editor/** ——来自公开仓库 [gym-boy-3000-editor](https://github.com/sneeb123/gym-boy-3000-editor) 的静态页面。没有服务器：一切都在浏览器中运行；登录使用针对自己数据仓库的细粒度 PAT（Contents：Read and write）；令牌保存在浏览器 localStorage 中，且只发送到 api.github.com。源码——[gym-boy-3000-editor](https://github.com/sneeb123/gym-boy-3000-editor) 仓库。手机会在下一次同步时获取修改。

## Русский

Персональное Android-приложение для записи тренировок в спортзале. Стиль — Пип-Бой из Fallout:
фосфорно-зелёный терминал, моноширинный шрифт, рамки-панели, развёртка ЭЛТ.

### Установка

Подписанный APK: [Releases](https://github.com/sneeb123/gym-boy-3000/releases/latest) —
скачать APK, разрешить установку из неизвестных источников
(приложение бесплатное, лицензия MIT; релизы публикуются в этом репозитории,
исходники разрабатываются в приватном).

### Концепция

Программу тренировки составляет **не** телефон: снаружи (нейросеть/скрипт/чат-бот) генерируется
JSON-файл программы и попадает на телефон. Основной канал — автосинхронизация с приватным
репозиторием GitHub (раздел «Облако» ниже); файл программы можно и просто открыть из мессенджера
или проводника («Открыть с помощью Gym-Boy 3000») — она так же встанет в план. На тренировке заполняются только два поля: **фактические повторы** и **пульс**.
Завершённая тренировка уходит в облако одним нажатием — момент решаете вы, — анализируется
снаружи, и по ней составляется следующая программа. Замкнутый цикл.

### Экран тренировки

- **Шапка**: дата, номер тренировки, кнопка **СТАРТ**. По нажатию превращается в секундомер,
  повторное нажатие фиксирует время тренировки.
- **Листание тренировок**: свайп или стрелки `‹ 02/06 ›` вверху экрана листают записи
  хронологически — архив (старые → новые), начатая тренировка, затем будущие программы
  плана (помечены `// ПЛАН`); прошлое слева. При запуске открывается ближайшая
  невыполненная тренировка; ▶ СТАРТ доступен на любой странице плана — остальные
  ждут своей очереди.
- **Своя панель ввода**: тап по ФАКТ или ПУЛЬС открывает компактную цифровую панель
  вместо системной клавиатуры — таблица подходов остаётся видимой целиком. Плановое
  значение подставляется черновиком, «ДАЛЕЕ» подтверждает и ведёт по подходам,
  после последнего панель закрывается сама.
- **Стрелка передачи**: завершённая тренировка ждёт кнопку ↑ в ряду листания — стрелка
  видна только на её странице (должны стоять метки запуска и остановки, облако должно
  быть подключено), у архивных записей её нет; нажатие заливает результат в облако
  и переводит страницу в архив.
- **Подсказки пульса**: пульс последней тренировки с тем же упражнением (по подходам)
  виден тусклым янтарём в ячейке ПУЛЬС текущего подхода — видно, как тело адаптируется
  к нагрузке.
- **Язык**: кнопка в футере переключает English → 中文 → Русский по кругу (по умолчанию
  английский; выбор запоминается на телефоне).
- **Вкладки упражнений** с таблицей подходов:

| № | БЛИНЫ | ВСЕГО | ПЛАН | ФАКТ | ПУЛЬС |
|---|-------|-------|------|------|-------|
| 1 | 40    | 60    | 8    | 8    | 125   |
| 2 | 50    | 70    | 8    | 7    | 138   |

Редактируются только ФАКТ и ПУЛЬС. «Блины» — вес дисков на штанге, «Всего» = блины + гриф
(вес грифа задаётся на уровне упражнения, по умолчанию 20 кг).

### Форматы данных

#### Программа (вход, `gymboy-program/1`)

Обязательны только `exercises` (хотя бы одно) и у каждого упражнения — `name`; остальное
можно опускать. Значения по умолчанию: `bar_weight_kg` — 20 кг; `total_weight_kg`, если
опущен, считается как блины + гриф; `set_number` проставляется сам, указывать его не нужно.
`workout_number` и `date` идентифицируют тренировку (по ним сортируется план); `notes`
и `planned_reps` необязательны. Значение `schema` не проверяется — оставьте
`gymboy-program/1` для ясности. Неизвестные ключи игнорируются.

```json
{
  "schema": "gymboy-program/1",
  "workout_number": 42,
  "date": "2026-10-23",
  "notes": "Акцент на ноги, отдых между подходами 3 мин",
  "exercises": [
    {
      "name": "Приседания",
      "bar_weight_kg": 20.0,
      "sets": [
        { "plates_weight_kg": 40.0, "planned_reps": 8 },
        { "plates_weight_kg": 50.0, "planned_reps": 8 },
        { "plates_weight_kg": 60.0, "planned_reps": 8 },
        { "plates_weight_kg": 60.0, "planned_reps": 6 }
      ]
    }
  ]
}
```

#### Результат (выход, `gymboy-result/1`)

То же самое плюс время и фактические значения:

```json
{
  "schema": "gymboy-result/1",
  "workout_number": 42,
  "date": "2026-10-23",
  "started_at": "2026-10-23T18:03:11.240Z",
  "finished_at": "2026-10-23T19:07:45.110Z",
  "duration_seconds": 3873,
  "exercises": [
    {
      "name": "Приседания",
      "bar_weight_kg": 20.0,
      "sets": [
        {
          "set_number": 1,
          "plates_weight_kg": 40.0,
          "total_weight_kg": 60.0,
          "planned_reps": 8,
          "actual_reps": 8,
          "pulse": 125
        }
      ]
    }
  ]
}
```

### Облако (GitHub)

Замкнутый цикл без ручного перекидывания файлов: приватный репозиторий GitHub работает
как почтовый ящик между ПК (кто составляет программы) и телефоном.

```
programs/pending/    ← новые программы (кладёт ПК/нейросеть)
programs/imported/   ← принятые приложением (переносит само)
results/             ← архив результатов (заливает приложение)
```

Поведение:

- Обмен запускается при старте приложения, при каждом возврате из фона (не чаще раза
  в 3 минуты), сразу после передачи тренировки и вручную — клик по индикатору
  «⇅ СВЯЗЬ» в шапке.
- Каждая новая программа из `pending` встаёт в план очередью — ни одна не теряется;
  забранные файлы уезжают в `imported`.
- Начатую тренировку обмен не трогает: новые программы ждут в плане своей очереди.
- Результаты синхронизируются в обе стороны: локальные правки заливаются,
  внешние (веб-редактор) скачиваются, при конфликте побеждает версия телефона.
- Обратный путь тоже работает: результаты из облака, которых нет на телефоне,
  скачиваются в локальный архив — после переустановки приложения или на новом
  телефоне история возвращается сама при первом обмене.
- Приватность: приложение проверяет, приватный ли репозиторий (флаг `private`),
  и уведомляет, если нет, — бейдж показывает янтарное «СВЯЗЬ: ПУБЛИЧНО», в панели
  связи — предупреждение. Обмен при этом не блокируется: решение за пользователем.

#### Настройка — быстрый вход (основной путь)

Кнопка **«Войти через GitHub»** в терминале связи: приложение показывает одноразовый код
(он сразу скопирован в буфер обмена), вы вводите его на `github.com/login/device`, и на той
же странице GitHub предложит установить публичное приложение **Gym-Boy 3000** — дайте ему
доступ к приватному репозиторию `gymboy-data`. Это единственный репозиторий, который
приложение видит (права только на содержимое); токен сессии хранится только на телефоне
и продлевается в фоне сам.

Если репозитория ещё нет — создайте приватный с именем `gymboy-data`: в панели приложения
есть кнопки — «Установки» (страница установки приложения), «Создать репо»
(github.com/new) и «Проверить». Регистрировать что-либо в настройках разработчика GitHub
не нужно: GitHub App вшит в сборку.

Ручная настройка fine-grained PAT осталась резервом (см. ниже).

#### Ручная настройка (PAT, если репозиторий уже есть)

1. Создать приватный репозиторий (например, `gymboy-cloud`) с папками
   `programs/pending`, `programs/imported`, `results` (хотя бы с `.gitkeep`).
2. Создать fine-grained токен: GitHub → Settings → Developer settings →
   Personal access tokens → Fine-grained tokens → Generate new token;
   Repository access — только этот репозиторий; Permissions → Contents:
   Read and write.
3. В приложении: клик по «⇅ СВЯЗЬ» → «РУЧНАЯ НАСТРОЙКА (PAT)» → вписать владельца,
   репозиторий и токен → «Подключить». Токен хранится только на телефоне (SharedPreferences).

#### Веб-редактор (онлайн)

Табличная правка файлов облака без git и JSON:
**https://sneeb123.github.io/gym-boy-3000-editor/** — статическая страница из
публичного репозитория [gym-boy-3000-editor](https://github.com/sneeb123/gym-boy-3000-editor).
Сервера нет: всё выполняется в браузере, вход — fine-grained PAT на свой репозиторий
данных (Contents: Read and write), токен хранится в localStorage браузера и уходит
только на api.github.com. Исходник — репозиторий [gym-boy-3000-editor](https://github.com/sneeb123/gym-boy-3000-editor). Правки
подхватываются телефоном при следующем синке.
