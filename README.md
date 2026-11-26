# Техническое задание и архитектурный манифест v2.0
**Проект:** English Fluency Builder (Android pet project)  
**Стек:** Kotlin, Android SDK, Jetpack Compose, Room, Hilt, Coroutines/Flow, FSRS-6  

## 1. Научный базис и методология

Приложение не является очередным клоном Anki. Оно опирается на три подтвержденных исследованиями принципа, адаптированных под мобильный формат:

1. **Осмысленный входной материал (Input Processing):** Усвоение происходит только через понимание смысла в контексте. Простое заучивание без контекста не создает устойчивых связей.
2. **Активное воспроизведение (Output Hypothesis):** Понимания недостаточно для беглости. Пользователь должен формулировать фразы сам, чтобы заметить пробелы в знаниях и проверить свои гипотезы о языке.
3. **Обучение через задачи (TBLT):** Эффективность зависит от качества задачи. Упражнения должны быть ориентированы на результат (например, "ответь на вопрос в диалоге"), а не на механическую подстановку слов.

### Баланс типов сессий
Правило "70% активного воспроизведения" применяется гибко, в зависимости от этапа изучения фразы:
- **Введение:** 80% восприятие / 20% направляемое воспроизведение. Цель: понять смысл.
- **Практика:** 30% восприятие / 70% активное воспроизведение. Цель: автоматизация.
- **Перенос:** 100% активное воспроизведение в новом контексте. Цель: проверка реального владения.

Алгоритм интервального повторения (SRS) управляет переходом фразы между этими этапами.

### Типы учебных задач
Вместо абстрактных "карточек" используется прогрессия из четырех типов задач с нарастающей сложностью:
1. **Распознавание:** Выбор правильной фразы из вариантов в диалоге. (Низкая нагрузка, этап введения).
2. **Сборка:** Составление фразы из блоков после прослушивания. (Средняя нагрузка, закрепление формы).
3. **Воспроизведение:** Самостоятельное произнесение или ввод фразы в заданном контексте. (Высокая нагрузка, автоматизация).
4. **Перенос:** Использование фразы в новом, непредвиденном сценарии. (Критерий полного усвоения).

Фраза считается выученной только после успешного прохождения задачи типа "Перенос".

---

## 2. Инженерные принципы (адаптировано для пет-проекта)

Чтобы пет-проект не превратился в неподдерживаемый монолит, соблюдаются строгие, но разумные правила:

1. **TDD в домене:** Тесты пишутся до кода для бизнес-логики. Это компас: если тест написать сложно, архитектура переусложнена.
2. **Чистая архитектура:** Строгое разделение слоев. Domain-слой не знает об Android, Room или Hilt.
3. **Модульность вместо раздутого DDD:** Вместо тяжелых "ограниченных контекстов" (Bounded Contexts) используются два четких логических модуля в домене: Лингвистический и Педагогический. Это предотвращает смешивание данных о языке и данных о прогрессе пользователя.
4. **Интерфейсы (порты) минимальны:** Инфраструктура (TTS, STT, FSRS) подключается через узкие интерфейсы, что позволяет легко писать фейки для тестов.

---

## 3. Архитектура домена (два логических модуля)

### 3.1. Лингвистический модуль
Отвечает за язык как систему. Меняется только при обновлении контента или правил языка.

```kotlin
@JvmInline value class PhraseCanonical(val value: String)
@JvmInline value class ContextId(val value: UUID)

enum class Register { FORMAL, INFORMAL, NEUTRAL }
enum class FrequencyBand { TOP_100, TOP_500, TOP_1000, RARE }

data class LinguisticUnit(
    val id: UnitId,
    val canonical: PhraseCanonical,
    val frequencyBand: FrequencyBand,
    val register: Register
)

data class UsageContext(
    val id: ContextId,
    val scenarioDescription: String,
    val exampleUtterance: PhraseCanonical,
    val sourceReference: String // например, "Friends S01E03" или "COCA"
)
```

### 3.2. Педагогический модуль
Отвечает за процесс обучения и прогресс пользователя.

```kotlin
enum class TaskType { RECOGNITION, RECONSTRUCTION, PRODUCTION, TRANSFER }
enum class ProficiencyLevel { NONE, PASSIVE, CONTROLLED, AUTOMATIC }

@JvmInline value class AcceptanceThreshold(val value: Float) {
    init { require(value in 0.0f..1.0f) { "Порог должен быть от 0 до 1" } }
}

data class PhraseProficiency(
    val unitId: UnitId,
    val level: ProficiencyLevel,
    val lastReviewAt: Instant
)

// Интерфейс политики оценки: определяет, ЧТО считать правильным, а не КАК это считать
interface PronunciationAssessmentPolicy {
    fun assess(
        spoken: RawTranscript, 
        expected: PhraseCanonical,
        threshold: AcceptanceThreshold
    ): AssessmentVerdict
}

data class AssessmentVerdict(
    val isAccepted: Boolean,
    val feedbackType: FeedbackType // PERFECT, CLOSE, RETRY, LISTEN_AGAIN
)
```

**Важное разделение:** `PronunciationAssessmentPolicy` живет в домене и задает правила (например, "допускается 1 ошибка в окончании"). Конкретный алгоритм (Левенштейн или ML-модель) живет в инфраструктуре и реализует этот интерфейс.

---

## 4. Слой приложения (Use Cases)

Координирует работу двух модулей домена.

```kotlin
class GetNextLearningTaskUseCase @Inject constructor(
    private val sequenceGenerator: ExerciseSequenceGenerator,
    private val linguisticRepository: LinguisticRepository,
    private val proficiencyRepository: ProficiencyRepository,
class ProcessUserResponseUseCase @Inject constructor(
    private val assessmentPolicy: PronunciationAssessmentPolicy,
    private val proficiencyRepository: ProficiencyRepository,
    private val fsrsScheduler: FsrsSchedulerPort,
    private val contextVariationService: ContextVariationService
) {
    suspend operator fun invoke(
        taskId: TaskId,
        response: UserResponse
    ): Result<ProcessingResult> {
        // 1. Оценка ответа согласно доменной политике
        val verdict = when (response) {
            is SpokenResponse -> assessmentPolicy.assess(
                response.transcript, response.expectedPhrase, response.threshold
            )
            else -> /* логика для других типов ответов */
        }
        
        // 2. Планирование следующего повторения (FSRS + вариация контекста при ошибке)
        val schedulingDecision = if (!verdict.isAccepted && verdict.feedbackType == FeedbackType.LISTEN_AGAIN) {
            contextVariationService.suggestAlternativeContext(taskId)
        } else {
            fsrsScheduler.schedule(taskId, verdict.toFsrsRating())
        }
        
        // 3. Сохранение прогресса
        proficiencyRepository.updateAfterTask(taskId, verdict, schedulingDecision)
        
        return Result.success(ProcessingResult(verdict, schedulingDecision))
 a: TTS, STT, FSRS

```kotlin
// Порт для синтеза речи
interface SpeechSynthesizer {
    suspend fun speak(text: PhraseCanonical, language: Language): Result<Unit>
    suspend fun stop(): Result<Unit>
}

// Порт для распознавания речи (возвращает сырой текст, не оценивает его)
interface SpeechRecognizer {
    suspend fun recognize(language: Language): Result<RawTranscript>
}

data class RawTranscript(val text: String, val confidence: Float)
```

---

## 6. Контентная стратегия

1. **Источники:** Топ-200 коллокаций, топ-50 живых идиом, топ-80 фразовых глаголов (по данным корпусов COCA/BNC).
2. **Структура карточки (обязательные поля):**
   - Целевая фраза (`canonical`).
   - Частотность и регистр использования.
   - Минимум 3 разных `UsageContext` для одной фразы (для вариативности при ошибках).
   - Ссылка на источник контекста.
3. **Запрещено:** Изолированные слова без контекста, устаревшие идиомы, грамматические правила в виде текста, фразы с единственным примером использования.

---

## 7. Этапы разработки (Roadmap для соло-разработчика)

- **Фаза 0: Фундамент (Недели 1-2).** Настройка модулей Gradle, CI (lint, detekt, тесты), базовые value-классы, интеграция FSRS-6.
- **Фаза 1: Ядро домена (Недели 3-5).** Генератор последовательностей задач, политика оценки произношения, Use Cases. Покрытие домена тестами. Создание Fake-реализаций портов.
- **Фаза 2: Данные и инфраструктура (Недели 6-8).** Схема Room, маппинг, адаптеры Android TTS/STT, загрузчик начальных данных из JSON с валидацией.
- **Фаза 3: Интерфейс MVP (Недели 9-12).** Навигация Compose, экраны для 4 типов задач, ежедневный поток занятий.
- **Фаза 4: Валидация (Недели 13-15).** Тестирование на реальных пользователях, настройка порогов распознавания речи, профилирование производительности.

---

## 8. Риски и способы их снижения

| Риск | Влияние | Способ снижения |
| :--- | :--- | | |
| Излишняя сложность (overengineering) | Критическое | TDD как компас. Если тест пишется тяжело, архитектура переусложнена. Фокус на MVP. |
| Плохая работа STT (распознавания) | Высокое | Настраиваемый порог уверенности. Fallback на текстовый ввод. Прогресс не должен блокироваться. |
| Сборка фраз деградирует в угадывание | Высокое | Обязательное произнесение вслух после сборки. Динамические варианты ответов. |
| Выгорание при разработке в одиночку | Критическое | Жесткие рамки по времени для каждой фазы. Никакого разрастания функционала (feature creep) до релиза MVP. |

---

## 9. Критерии готовности (Definition of Done)

Для каждого модуля:
1. Тесты написаны до продакшн-кода (TDD).
2. Покрытие ветвлений (branch coverage) в домене не менее 90%.
3. В модулях Domain нет импортов из `android.*`.
4. Все публичные API документированы через KDoc.
5. Код проходит проверку Detekt и Ktlint без предупреждений.
6. Используются Value Objects вместо примитивов (например, `PhraseCanonical` вместо `String`).
7. Обработка ошибок через тип `Result`.
8. Контрактные тесты для всех портов инфраструктуры проходят успешно.
9. Соблюдено разделение на лингвистический и педагогический модули.
