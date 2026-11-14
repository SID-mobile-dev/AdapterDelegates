# AGENTS.md — инструкции для AI-агентов и инженеров

Этот файл читают и AI-агенты (OpenCode), и люди. Правила ниже обязательны для **любых** изменений в этом репозитории. Если правило не выполнено — изменения не готовы.

## 1. Контекст: что это за проект

**AdapterDelegates** — Android-библиотека (Java/Kotlin), которая позволяет строить RecyclerView-адаптеры композицией переиспользуемых компонентов вместо наследования («Favor composition over inheritance»).

**Идея**: для каждого типа элемента списка (view type) пишется свой `AdapterDelegate`, который умеет создать и привязать (`bind`) ViewHolder только для своего типа. Адаптер собирается как композиция делегатов:

```kotlin
val adapter = ListDelegationAdapter<List<Animal>>(
    catAdapterDelegate(...),
    dogAdapterDelegate(),
    snakeAdapterDelegate()
)
```

### Ключевые термины

| Термин | Значение |
|---|---|
| `AdapterDelegate` | контракт: `isForViewType()` — «мой ли это элемент?», `onCreateViewHolder()`, `onBindViewHolder()` |
| `AdapterDelegatesManager` | реестр делегатов: маппит viewType → делегат, вызывает нужный делегат |
| Delegation adapter | готовый адаптер поверх менеджера: `ListDelegationAdapter`, `AsyncListDifferDelegationAdapter`, `PagedListDelegationAdapter` |
| Fallback-delegate | делегат для «неизвестного» типа элемента (страховка вместо краша) |
| Kotlin DSL | синтаксический сахар `adapterDelegate { ... }` / `adapterDelegateLayoutContainer { ... }` / `adapterDelegateViewBinding { ... }` |
| `AbsListItemAdapterDelegate` | базовый класс делегатa с типизацией item'а (для Java-потребителей) |

### Основные сценарии использования

1. **Список из разнородных элементов** (Cat/Dog/Snake/Advertisement…) — по одному делегату на тип;
2. **Неизвестный тип элемента** — fallback-делегат рендерит заглушку;
3. **Обновления по DiffUtil/AsyncListDiffer** — `AsyncListDifferDelegationAdapter` + payload'ы в `bind { payloads -> }`;
4. **Пагинация** — `PagedListDelegationAdapter` (Paging 2);
5. **Kotlin DSL** — объявление делегата в одном лямбда-блоке (три варианта биндинга: `findViewById`, LayoutContainer/synthetics, ViewBinding).

### Как запустить и проверить

- **Тесты требуют JDK 11** (как в CI, `.github/workflows/`). На JDK 17 юнит-тесты с моками падают: `mockito-inline:2.21.0` не умеет инструментировать классы на Java 17. Локально:
  ```bash
  export JAVA_HOME=/Users/igorsergeev/Library/Java/JavaVirtualMachines/jbr_dcevm-11.0.16/Contents/Home
  ./gradlew test
  ```
  (путь к JDK машинно-зависимый; в CI используется `setup-java` с версией 11)
- Полная сборка: `./gradlew build`.
- Сборка сэмпл-приложения **не входит** в сборку проекта (см. раздел 2).

### Стек технологий

| Слой | Технология | Версия / ограничение |
|---|---|---|
| Язык (ядро) | Java | source/target compatibility **1.8** (`library`, `paging`) |
| Язык (DSL) | Kotlin | **1.5.31** (`kotlin-dsl*`) |
| Платформа | Android SDK | compileSdk 31, targetSdk 31, minSdk **14** |
| UI-компонент | AndroidX RecyclerView | 1.2.1 |
| Поддержка | AndroidX annotation / core / appcompat | 1.3.0 / 1.7.0 / 1.3.1 |
| Пагинация | AndroidX Paging 2 | paging-runtime **2.0.0** (именно Paging 2, не 3!) |
| Биндинги (в DSL) | Kotlin Android Extensions (synthetics, LayoutContainer), ViewBinding/DataBinding | `kotlin-android-extensions` (deprecated, но используется) |
| Тесты | JUnit 4.12, Mockito (mockito-inline 2.21.0) | тесты с моками — только под JDK 11 |
| Сборка | Gradle (wrapper) 7.0.2, Android Gradle Plugin 7.0.3 | |
| Публикация | com.vanniktech:gradle-maven-publish-plugin 0.18.0 | артефакты `com.hannesdorfmann:adapterdelegates4*`, версия в `gradle.properties` (`VERSION_NAME`) |

Не «модернизируй» стек по своей инициативе (Kotlin 2.x, Paging 3, ViewBinding вместо synthetics и т.п.) — это отдельные задачи с согласованием, а не побочный эффект других изменений.

## 2. Структура и границы модулей

### Карта репозитория — зачем нужен каждый каталог

```text
AdapterDelegates/
├── library/                  ядро: публичное API библиотеки (Java). Всё, что публикуется в Maven как adapterdelegates4.
│   └── src/{main,test}/java/com/hannesdorfmann/adapterdelegates4/
├── kotlin-dsl/               Kotlin DSL adapterDelegate { } для потребителей на Kotlin (базовый вариант, findViewById)
├── kotlin-dsl-layoutcontainer/  то же через LayoutContainer + synthetics (Kotlin Android Extensions)
├── kotlin-dsl-viewbinding/   то же через ViewBinding/DataBinding
├── paging/                   интеграция с AndroidX Paging 2: PagedListDelegationAdapter (Java)
├── app/                      сэмпл-приложение: примеры использования API (Reptiles/Diff/Pagination/DSL).
│                             ВНЕ сборки (нет в settings.gradle) — только как образец кода
├── gradle/, gradlew*         Gradle wrapper 7.0.2 — сборка идёт только через него
├── build.gradle              общий тулчейн и версии зависимостей (ext-блок) — источник истины по версиям
├── settings.gradle           список собираемых модулей (см. таблицу ниже)
├── gradle.properties         VERSION_NAME / GROUP — версионирование публикуемых артефактов
├── .github/workflows/        CI: сборка на JDK 11, публикация релизов/снапшотов
├── AGENTS.md                 этот файл — инструкции для AI и инженеров
└── REPORT.md                 отчёт по ДЗ (не влияет на сборку)
```

### Модули из `settings.gradle`:

| Модуль | Язык | Ответственность |
|---|---|---|
| `library` | Java | ядро публичного API: `AdapterDelegate`, `AdapterDelegatesManager`, `AbsDelegationAdapter`, `ListDelegationAdapter`, `AsyncListDifferDelegationAdapter`, `AbsListItemAdapterDelegate`, `AbsFallbackAdapterDelegate` |
| `kotlin-dsl` | Kotlin | DSL `adapterDelegate { }` (через `findViewById`) |
| `kotlin-dsl-layoutcontainer` | Kotlin | DSL `adapterDelegateLayoutContainer { }` (Kotlin Android Extensions / synthetics) |
| `kotlin-dsl-viewbinding` | Kotlin | DSL `adapterDelegateViewBinding { }` (ViewBinding/DataBinding) |
| `paging` | Java | `PagedListDelegationAdapter` для Paging 2 |

- `app/` — сэмпл-приложение (примеры `ReptilesActivity`, `DiffActivity`, `PaginationActivity`). **Не входит в `settings.gradle` и не собирается** `./gradlew build`. Используется только как образец использования API.
- Направление зависимостей: `kotlin-dsl*` и `paging` → `library`. Обратных зависимостей быть не должно.

### Что нельзя менять (и почему)

1. **Публичные сигнатуры `AdapterDelegate`, `AdapterDelegatesManager`, `AbsDelegationAdapter` и других классов `library`/`paging`** — это опубликованное API (Maven-артефакты `com.hannesdorfmann:adapterdelegates4*`), его используют внешние потребители. Изменение сигнатуры = breaking change.
2. **Не переписывать Java-код `library` и `paging` на Kotlin** — артефакты публикуются и для Java-потребителей, и для Kotlin; смена языка меняет публичный API (nullability-контракты, synthetic-методы).
3. **Не добавлять зависимости в `library` без крайней необходимости** — ядро должно оставаться лёгким; тестовые зависимости (JUnit, Mockito) объявляются в `ext` корневого `build.gradle` и переиспользуются модулями.
4. **Не менять версионирование и публикацию** (`gradle.properties` с `VERSION_NAME`, `com.vanniktech.maven.publish`) — релизный процесс автоматизирован через GitHub Actions.
5. **Не трогать `.github/workflows/`** — CI гоняет `./gradlew build` на JDK 11, это эталонная проверка.

### Уточняющие вопросы vs. допущения

- **Обязательно спросить** (не делать допущений), если задача затрагивает: публичное API `library`/`paging`, новые зависимости, версионирование, новые модули.
- **Можно делать консервативные допущения** в остальных случаях (внутренняя реализация, тесты, сэмпл `app/`), но каждое допущение нужно явно перечислить в ответе.

## 3. Правила безопасности (обязательны для агента)

1. **Никогда не хардкодить секреты**: API-ключи, токены, пароли, URL с credentials, содержимое `local.properties` и ключи подписи. Секреты живут только в переменных окружения (как в CI: `ORG_GRADLE_PROJECT_mavenCentralUsername`, `ORG_GRADLE_PROJECT_signingInMemoryKey` и т.п.).
2. **Не коммитить и не показывать в отчётах/логах**: `local.properties` (путь к Android SDK), содержимое `~/.gradle/gradle.properties`, любые credentials. Если такой файл попал в вывод — заменить значения на `<redacted>`.
3. **Не ослаблять защиту репозитория**: не изменять `.gitignore` с целью «прокинуть» игнорируемые файлы, не добавлять реальные ключи в сэмпл `app/` (только placeholder'ы и `BuildConfig` из env).
4. **Не выполнять деструктивных действий без явного подтверждения**: `git push --force`, удаление веток/тегов, `rm -rf` по широким маскам, публикация артефактов (`./gradlew publish`).
5. **Не подключать новые внешние сервисы и MCP-серверы**, не отправлять код проекта наружу (внешние URL) без явного запроса пользователя.
6. **Не расширять себе права**: не добавлять себя в `permissions`/`sudoers`-конфиги, не отключать проверки (тесты, lint, CI) ради «зелёной» сборки.

## 4. Если нужной информации нет в этом файле

**Не додумывай и не выдумывай факты.** Если ответа нет в `AGENTS.md` / README:

1. **Изучи соответствующий участок кода** (и его тесты) — источники истины в порядке приоритета: код и тесты модуля → README → build-файлы → git history (`git log -p <файл>`).
2. **Ссылайся на источник** в ответе: «в `AdapterDelegatesManager.java:NN` сделано так-то» — а не «я предполагаю, что…».
3. **Не выдумывай API, команды и версии**: если не нашёл, как зовут метод или как запускается проверка, — сначала найди в коде/конфиге, потом пиши.
4. **Проверяй выводы тестами**: любое утверждение о поведении кода можно подтвердить существующим тестом или добавить новый.
5. **Если после изучения кода остаётся неоднозначность** — задай вопрос пользователю (см. политику вопросов в разделе 2), а не выбирай вариант наугад.

## 5. Правила взаимодействия с AI и требования к результату

### Как ставятся задачи

Задача ставится промптом в свободной форме, но всегда трактуется через этот файл. Минимальный смысл задачи: **цель → ограничения → критерии готовности**. Если в промпте чего-то нет — действуй по правилам этого файла (а не по своим общим привычкам) и перечисли свои допущения в ответе.

### Обязательный формат ответа агента

Ответ по каждой задаче состоит из четырёх блоков, строго в этом порядке:

1. **План действий** — до любых правок: что и в каких файлах будет сделано;
2. **Изменения в коде** — конкретный список: файл → что изменено (без полных дампов файлов в ответ);
3. **Пояснение решений** — почему выбран этот вариант, какие сделаны допущения, какие альтернативы отклонены;
4. **Шаги проверки** — команды, которые нужно выполнить, и их фактический результат («запустил `./gradlew test` — 100 тестов зелёные»).

### Стиль кода

- **Следуй стилю модуля**: Java в `library`/`paging` (Java 8, отступы 4 пробела, javadoc на публичных методах — как в существующем коде), Kotlin в `kotlin-dsl*` (официальный Kotlin style).
- **Не переформатируй существующий код** без необходимости — дифф должен содержать только смысловые изменения.
- **Тесты**: JUnit 4; в Java-тестах имена методов `camelCase` (`callAllMethods`), в Kotlin-тестах — backtick-фразы (`init block is called once - bind multiple times`) — сохраняй принятый в модуле стиль.
- Не оставляй закомментированный код и мёртвый код; `TODO` — только с обоснованием.

### Тесты и проверки

- Любое изменение поведения в собираемых модулях покрывается юнит-тестами (JUnit 4 + Mockito; для проверки вызовов используй паттерн `SpyableAdapterDelegate`, как в существующих тестах `library`).
- Исправление бага = **регрессионный тест**, воспроизводивший баг до фикса.
- Перед завершением задачи прогони `./gradlew test` под **JDK 11** (см. раздел 1) и приведи фактический результат в блоке «Шаги проверки».
- Запрещено «чинить» сборку отключением/удалением тестов, уменьшением их строгости или правкой CI.

### Проверочное правило (маркер, что инструкции прочитаны)

**Каждый новый или существенно изменённый файл исходного кода начинается с комментария-преамбулы:**

```java
// AI-TASK: <краткое название задачи>
```

(для Kotlin — тот же комментарий `// AI-TASK: ...` на первой строке файла, до `package`)

Цель правила: по преамбуле сразу видно, что изменения вносил агент, читавший `AGENTS.md`. Если преамбулы нет — изменения считаются несогласованными.
