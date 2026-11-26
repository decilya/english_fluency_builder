# ТЕХНИЧЕСКОЕ ЗАДАНИЕ И АРХИТЕКТУРНЫЙ МАНИФЕСТ v2.0
**Проект:** English Fluency Builder  
**Статус:** APPROVED BASELINE  
**Стек:** Kotlin, Android SDK, Jetpack Compose, Room, Hilt, Coroutines/Flow, FSRS-6  

## 1. НАУЧНЫЙ БАЗИС И МЕТОДОЛОГИЯ ОБУЧЕНИЯ

### 1.1. Фундаментальные принципы (Evidence-Based)
Приложение не является "еще одной Anki". Оно строится на синтезе трех подтвержденных исследований:
1) **Input Processing Theory (VanPatten):** Создание ментальной репрезентации происходит *только* через обработку meaning-bearing input. Output не создает новое знание, он консолидирует существующее.
2) **Output Hypothesis (Swain):** Comprehensible input необходим, но недостаточен для развития грамматической точности и беглости. Pushed output выполняет уникальные когнитивные функции: noticing gap, syntactic processing, hypothesis testing.
3) **Task-Based Language Teaching (TBLT):** Эффективность определяется качеством дизайна задач, а не фактом их использования. Задачи должны быть outcome-oriented с meaningful negotiation, а не form-focused drills.

### 1.2. Ключевое архитектурное следствие: Session-Type Balance
Метрика "70% output" применяется **дифференцированно**, а не глобально:
1) **Introduction Session:** 80% Input / 20% Guided Output. Цель — создание ментальной репрезентации.
2) **Practice Session:** 30% Input / 70% Output. Цель — консолидация и автоматизация.
3) **Transfer Session:** 100% Output в новом контексте. Цель — проверка истинного владения.

SRS управляет переходом пользователя между типами сессий для каждой конкретной фразы.

### 1.3. Таксономия учебных задач (4-Type Progression)
Вместо generic "TaskScenario" используется строго определенная таксономия с нарастающей когнитивной нагрузкой:

| Тип задачи | Описание | Когнитивная нагрузка | Роль в SRS |
| :--- | :--- | :--- | :--- |
| **Recognition** | Выбор правильной фразы из 3 вариантов в диалоге | Low | Введение / Раннее повторение |
| **Reconstruction** | Сборка фразы из блоков после прослушивания | Medium | Консолидация формы |
| **Production** | Произнесение фразы в заданном контексте | High | Автоматизация |
| **Transfer** | Использование фразы в *новом*, непредвиденном сценарии | Highest | Критерий "выучено" |

**Правило:** Фраза считается освоенной ("ProficiencyLevel.AUTOMATIC") только после успешного прохождения Transfer Task.

### 1.4. Алгоритм интервального повторения
1) **Базовый алгоритм:** FSRS-6 (Free Spaced Repetition Scheduler). Превосходит SM-2 по retention prediction accuracy, использует ML-моделирование memory state.
2) **Context Variation Layer:** Надстройка *поверх* FSRS (не модификация самого алгоритма). При оценке "Again" из-за непонимания контекста система меняет *пример использования*, а не только интервал.
3) **Card Format:** Только sentence/phrase cards с обязательным полем "context". Изолированные слова запрещены.
4) **Decay Policy:** Фразы, не повторенные за N месяцев и имеющие низкий frequency score, архивируются (не удаляются). Активный набор ограничен для предотвращения cognitive overload.

---

## 2. ИНЖЕНЕРНЫЕ ПРИНЦИПЫ (NON-NEGOTIABLE)

### 2.1. SOLID + TDD как аксиома
1) **TDD:** Red-Green-Refactor. Тест пишется ДО продакшн-кода. Domain Layer — 100% branch coverage.
2) **SRP:** Каждая класс имеет одну причину для изменения. Педагогическая логика != Лингвистическая модель != Инфраструктура.
3) **OCP:** Расширение через абстракции. Новые типы задач/генераторов добавляются без модификации существующего кода.
4) **LSP:** Подтипы полностью заменяемы. Контракты портов не нарушаются реализациями.
5) **ISP:** Порты минимальны. Клиенты не зависят от неиспользуемых методов.
6) **DIP:** Все зависимости на абстракциях. Конкретные реализации инжектируются через Hilt.

### 2.2. Strict Clean Architecture
Никаких упрощений. Dependency Rule соблюдается абсолютно.
1) **Domain Layer:** Чистый Kotlin. Zero Android dependencies. Два Bounded Contexts (см. п. 3.1).
2) **Application Layer:** Use Cases, orchestrating Domain BCs. Application-specific business rules.
3) **Data Layer:** Repository implementations, Room DAO, mapping DTO <-> Domain.
4) **Infrastructure Layer:** TTS, STT, FSRS adapter, JSON parsing, Android APIs.
5) **Presentation Layer:** Jetpack Compose, ViewModels. Зависит только от Application (Use Cases).

### 2.3. Domain-Driven Design Elements
1) **Value Objects:** Неизменяемые, с валидацией при создании. Защищают инварианты.
2) **Entities:** С идентичностью и жизненным циклом.
3) **Domain Services:** Логика, не принадлежащая одной сущности.
4) **Ubiquitous Language:** Код говорит на языке предметной области.

### 2.4. Testing Strategy
1) **Unit Tests:** JVM only. No Android Framework, no Robolectric. MockK/Fakes для портов.
2) **Pedagogical Acceptance Tests (PAT):** Интеграционные сценарии, валидирующие *методику*, а не код. Пишутся совместно лингвистом и QA. Пример: "Given phrase X learned in context Y, When tested after Z days with Transfer task, Then retention > N%".
3) **Contract Testing:** Для всех Infrastructure портов. Тесты прогоняются против Fake И Real implementation. Расхождение = fail build.
4) **Integration Tests:** Room, Network, critical Infrastructure adapters.

---

## 3. АРХИТЕКТУРА DOMAIN LAYER (TWO BOUNDED CONTEXTS)

Педагогическая методика IS the business rule данного продукта. Она находится в Domain, но выделена в отдельный Bounded Context для соблюдения SRP.

### 3.1. Linguistic Bounded Context
Отвечает за *язык как систему*. Меняется при изменении лингвистических требований.

```kotlin
// Value Objects
@JvmInline value class PhraseCanonical(val value: String)
@JvmInline value class ContextId(val value: UUID)

enum class Register { FORMAL, INFORMAL, NEUTRAL }
enum class FrequencyBand { TOP_100, TOP_500, TOP_1000, RARE }

// Entity
data class LinguisticUnit(
    val id: UnitId,
    val canonical: PhraseCanonical,
    val type: LinguisticType, // COLLOCATION, IDIOM, FRAME, PHRASAL_VERB
    val frequencyBand: FrequencyBand,
    val register: Register
)

// Value Object: Контекст использования (не строка!)
data class UsageContext(
    val id: ContextId,
    val scenarioDescription: String,
    val exampleUtterance: PhraseCanonical,
    val emotionalTone: Tone,
    val sourceReference: String // "Friends S01E03", "COCA corpus"
)
```

### 3.2. Pedagogical Bounded Context
Отвечает за *процесс обучения*. Меняется при изменении методики.

```kotlin
// Value Objects
enum class TaskType { RECOGNITION, RECONSTRUCTION, PRODUCTION, TRANSFER }
enum class ProficiencyLevel { NONE, PASSIVE, CONTROLLED, AUTOMATIC }

@JvmInline value class AcceptanceThreshold(val value: Float) {
    init { require(value in 0.0f..1.0f) { "Threshold must be 0..1" } }
}

// Entity: Состояние владения конкретной фразой
data class PhraseProficiency(
    val unitId: UnitId,
    val level: ProficiencyLevel,
    val taskTypeProgress: Map<TaskType, TaskCompletionState>,
    val lastReviewAt: Instant
)

// Domain Service Interface: Политика оценки произношения
// WHAT оценивается, а не HOW
interface PronunciationAssessmentPolicy {
    fun assess(
        spoken: RawTranscript, 
        expected: PhraseCanonical,
        threshold: AcceptanceThreshold
    ): AssessmentVerdict
}

data class AssessmentVerdict(
    val isAccepted: Boolean,
    val confidenceScore: Float,
    val feedbackType: FeedbackType // PERFECT, CLOSE, RETRY, LISTEN_AGAIN
)

// Domain Service Interface: Генератор последовательности упражнений
interface ExerciseSequenceGenerator {
    fun nextTaskFor(
        proficiency: PhraseProficiency,
        availableContexts: List<UsageContext>
    ): LearningTask
}
```

### 3.3. Разделение ответственности: Assessment Policy vs Algorithm

| Компонент | Слой | Ответственность |
| :--- | :--- | :--- |
| `PronunciationAssessmentPolicy` | Domain (Pedagogical BC) | *Что* считать правильным. Пороги, типы обратной связи. |
| `LevenshteinPronunciationAssessor` | Infrastructure | *Как* сравнивать строки (алгоритм Левенштейна). |
| `MlBasedPronunciationAssessor` | Infrastructure | *Как* сравнивать через ML-модель. |
| `AcceptanceThreshold` | Domain (Value Object) | Конфигурируемый порог. Загружается из remote config для A/B тестов. |

---

## 4. АРХИТЕКТУРА APPLICATION LAYER

Orchestrates взаимодействие двух Bounded Contexts. Содержит application-specific business rules.

```kotlin
// Use Case: Получение следующего упражнения
class GetNextLearningTaskUseCase @Inject constructor(
    private val sequenceGenerator: ExerciseSequenceGenerator,
    private val linguisticRepository: LinguisticRepository,
    private val proficiencyRepository: ProficiencyRepository,
    private val fsrsScheduler: FsrsSchedulerPort
) {
    suspend operator fun invoke(sessionType: SessionType): Result<LearningTask> {
        val dueUnits = proficiencyRepository.getDueForSession(sessionType)
        if (dueUnits.isEmpty()) return Result.success(LearningTask.SessionComplete)
        
        val unit = dueUnits.first()
        val contexts = linguisticRepository.getContextsFor(unit.id)
        val proficiency = proficiencyRepository.getProficiency(unit.id)
        
        return Result.success(
            sequenceGenerator.nextTaskFor(proficiency, contexts)
        )
    }
}

// Use Case: Обработка ответа пользователя
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
        // 1) Оценка (Domain Policy, не алгоритм!)
        val verdict = when (response) {
            is SpokenResponse -> assessmentPolicy.assess(
                response.transcript, response.expectedPhrase, response.threshold
            )
            else -> /* оценка других типов ответов */
        }
        
        // 2) Обновление SRS (FSRS + Context Variation)
        val schedulingDecision = if (!verdict.isAccepted && verdict.feedbackType == FeedbackType.LISTEN_AGAIN) {
            contextVariationService.suggestAlternativeContext(taskId)
        } else {
            fsrsScheduler.schedule(taskId, verdict.toFsrsRating())
        }
        
        // 3) Обновление proficiency
        proficiencyRepository.updateAfterTask(taskId, verdict, schedulingDecision)
        
        return Result.success(ProcessingResult(verdict, schedulingDecision))
    }
}
```

---

## 5. INFRASTRUCTURE LAYER KEY COMPONENTS

### 5.1. Speech Synthesis (TTS)
```kotlin
// Domain Port
interface SpeechSynthesizer {
    suspend fun speak(text: PhraseCanonical, language: Language): Result<Unit>
    suspend fun stop(): Result<Unit>
}

// Infrastructure Implementation
class AndroidSpeechSynthesizer @Inject constructor(
    @ApplicationContext private val context: Context
) : SpeechSynthesizer { /* ... */ }

// Test Double
class FakeSpeechSynthesizer : SpeechSynthesizer {
    val spokenPhrases = mutableListOf<PhraseCanonical>()
    override suspend fun speak(text: PhraseCanonical, language: Language): Result<Unit> {
        spokenPhrases.add(text)
        return Result.success(Unit)
    }
}
```

### 5.2. Speech Recognition (STT)
```kotlin
// Domain Port: Возвращает сырой результат, НЕ оценивает
interface SpeechRecognizer {
    suspend fun recognize(language: Language): Result<RawTranscript>
}

data class RawTranscript(val text: String, val confidence: Float)
```

### 5.3. FSRS Adapter
```kotlin
// Domain Port
interface FsrsSchedulerPort {
    fun schedule(cardId: CardId, rating: FsrsRating): SchedulingDecision
    fun getDueCards(now: Instant): List<CardId>
}

// Infrastructure: Обертка над open-spaced-repetition/fsrs-rs-ffi или Kotlin port
class FsrsSchedulerAdapter @Inject constructor(
    private val fsrsModel: FsrsModel // загружается из assets/config
) : FsrsSchedulerPort { /* ... */ }
```

---

## 6. КОНТЕНТНАЯ СТРАТЕГИЯ

### 6.1. Источники и объем
1) **Коллокации:** Top 200 частотных (COCA/BNC corpus)
2) **Идиомы:** Top 50 живых, подтвержденных корпусными данными
3) **Фразовые глаголы:** Top 80 наиболее употребимых
4) **Речевые фреймы:** 50 conversational frames для B1-C1
5) **Контексты:** Минимум 3 различных UsageContext на каждую фразу (для Context Variation Layer)

### 6.2. Структура карточки (Обязательные поля)
Каждая "LinguisticUnit" ОБЯЗАНА иметь:
1) "canonical": Целевая фраза
2) "frequencyBand": Частотность (определяет приоритет и decay)
3) "register": Регистр использования
4) >= 3 "UsageContext" с разными сценариями
5) "sourceReference": Источник каждого контекста

### 6.3. Запрещенный контент
1) Изолированные слова без контекста
2) Устаревшие/книжные идиомы без корпусного подтверждения
3) Грамматические правила в виде текста
4) Фразы с единственным примером использования

---

## 7. ЭТАПЫ РАЗРАБОТКИ (ROADMAP)

### Phase 0: Foundation (Weeks 1-2)
1) Gradle KTS multi-module setup (domain-linguistic, domain-pedagogical, application, data, infrastructure, presentation)
2) CI pipeline: lint, detekt, unit tests, contract tests, build
3) Базовые Value Objects + TDD
4) FSRS-6 integration + adapter + TDD

### Phase 1: Core Domain (Weeks 3-5)
1) "ExerciseSequenceGenerator" + все 4 типа задач + TDD
2) "PronunciationAssessmentPolicy" + TDD (edge cases, threshold behavior)
3) "GetNextLearningTaskUseCase", "ProcessUserResponseUseCase" + TDD
4) Pedagogical Acceptance Tests (совместно с лингвистом)
5) Fake implementations всех портов + Contract Tests

### Phase 2: Data & Infrastructure (Weeks 6-8)
1) Room schema + migrations + integration tests
2) Repository implementations + mapping
3) Android TTS/STT adapters + Contract Tests
4) Seed data loader (JSON -> DB) + validation

### Phase 3: Presentation MVP (Weeks 9-12)
1) Navigation (Compose Destinations)
2) Introduction Session screen (input-first)
3) Practice Session screens (4 task types)
4) Transfer Session screen
5) Daily session flow + progress tracking

### Phase 4: Validation & Polish (Weeks 13-15)
1) PAT execution on real users
2) STT accuracy tuning + threshold A/B testing
3) Performance profiling
4) Beta release

---

## 8. РИСКИ И МИТИГАЦИЯ

| Риск | Вероятность | Влияние | Митигация |
| :--- | :--- | :--- | :--- |
| Overengineering убьет темп | High | Critical | TDD как компас. PAT как validation. Если тест пишется тяжело — архитектура неверна. |
| STT работает плохо | Medium | High | Confidence threshold настраиваемый. Fallback на typing. Не блокировать прогресс. |
| Block Assembly деградирует в pattern matching | High | High | Обязательное произнесение после сборки. Динамические дистракторы. Interleaving с другими типами задач. |
| FSRS не учитывает семантическую связь | Medium | Medium | Context Variation Layer поверх FSRS. Decoupled от базового алгоритма. |
| Pedagogical Acceptance Tests расходятся с реальностью | Medium | High | Регулярная калибровка PAT на beta-тестерах. PAT != unit tests. |
| Burnout solo dev | High | Critical | Fixed scope per phase. No feature creep. MVP first. |

---

## 9. DEFINITION OF DONE (Для каждого модуля)

1) Unit tests written FIRST (TDD Red-Green-Refactor)
2) >= 90% branch coverage для Domain Layer
3) No Android imports в Domain modules
4) All public APIs documented (KDoc)
5) Detekt/Ktlint clean
6) Value Objects used instead of primitives
7) Error handling via Result type
8) Contract Tests pass for all Infrastructure ports
9) Pedagogical Acceptance Tests defined and passing
10) Reviewed against SOLID checklist
11) Two Bounded Contexts respected (no cross-contamination)
