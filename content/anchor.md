PRD: repo-native intention control plane for agentic development

Рабочее название: Anchor

К 2026 рынок уже довольно явно двинулся в сторону repo-native specs и harness engineering: Kiro ведёт работу через requirements-first, design-first и bugfix specs; Augment строит living specs и оркестрацию агентов вокруг них; Anthropic отдельно описывает long-running harnesses с progress-файлом рядом с git history; OpenAI прямо формулирует harness engineering как отдельный слой инженерной работы. Pi при этом специально оставлен минимальным terminal harness’ом без plan mode и subagents в core, но с project-local TypeScript extensions, lifecycle events, tool interception и custom UI. Это делает правильным не “ещё одну IDE” и не “ещё одну тикетницу”, а repo-native control plane поверх harness’ов, с Pi как первым адаптером.  ￼

1. Продуктовый тезис

Anchor делает intention первичным артефактом репозитория.

Один и тот же артефакт проходит жизненный цикл:
	•	сначала это свободно написанный task intent;
	•	потом это scope и guide для агента;
	•	после реализации это либо архивированный work record, либо promoted contract intent;
	•	дальше этот contract intent становится guard’ом для будущих сессий, агентов и ручных изменений.

Ключевая ставка: продукт должен быть не системой документации, а системой управления drift’ом.

Не “напиши красивую спецификацию”.
А “свяжи claim → code → witness так, чтобы будущий агент не сломал это молча”.

2. Что именно продукт решает

Продукт решает четыре проблемы.
	1.	Намерение живёт в чатах и исчезает между сессиями.
	2.	Изменения в коде не имеют явной связи с тем, какой контракт они реализуют или меняют.
	3.	Агент может делать легитимную работу в задаче B и незаметно повредить поведение из задачи A.
	4.	У команды нет repo-native слоя, который связывает:
	•	зачем это существует,
	•	где это реализовано,
	•	чем это подтверждается,
	•	что должно произойти, если это затронули.

3. Что это за продукт

Anchor — это не один файл и не один плагин.

Это три слоя:
	1.	Канонический engine
	•	CLI/library, живущая рядом с Git
	•	умеет индексировать код, строить trace graph, анализировать diff, запускать checks, предлагать reconcile patches
	2.	Repo-native artifacts
	•	task intents
	•	contract intents
	•	trace sidecars
	•	ссылки на ADR
	•	witnesses
	3.	Harness adapters
	•	первый адаптер: Pi extension
	•	дальше: git hooks, CI, другие harnesses

Это важно архитектурно. Даже Codex уже выделяет harness в отдельный App Server, который используется из разных поверхностей, а не только из одной TUI. Это сильный сигнал, что правильная граница здесь — engine + adapters, а не “вся логика живёт внутри одного harness plugin”.  ￼

4. Главные продуктовые принципы

4.1 Repo first

Source of truth — репозиторий.

Не Slack, не issue tracker, не chat transcript.

Это совпадает с тем, как современные spec-driven инструменты рекомендуют хранить specs: версионировать их прямо в проекте рядом с кодом.  ￼

4.2 Human file stays human

Человеческий файл не должен превращаться в свалку машинного мусора.

Я бы не держал полный Code Map, длинную историю событий и шумные auto-generated блоки прямо внутри markdown. Это быстро убьёт UX ревью.

Мой сильный вкус здесь такой:
	•	markdown-файл — для claims, boundaries, rationale, acceptance signals;
	•	sidecar trace file — для anchors, fingerprints, witness bindings, confidence, stale flags, decision log.

4.3 Claims, not prose blobs

Свободный текст хорош на входе.
Но каноническая форма должна быть claim-based.

Каждый intention должен нормализоваться в 2–7 коротких claims с ID.

Не так:

“Добавить JWT-auth с нормальным handling, refresh, edge cases и вообще всё должно быть ок.”

А так:
	•	C1. Все /api/v2/* endpoints требуют валидный JWT, кроме явно перечисленных исключений.
	•	C2. Expired token даёт 401, а не 500.
	•	C3. Refresh flow остаётся отдельным и не ломается.

Это делает guard понятным и точным: система предупреждает не “ты задел intention”, а “ты, вероятно, меняешь claim C2”.

4.4 Одна активная единица работы на сессию

По умолчанию у сессии один active task intent.

Можно иметь много пассивно загруженных contract intents как guard rails, но один task intent — это default.

Если человек хочет параллельную работу, пусть форкает сессию, а не смешивает два потока в одном контексте.

4.5 Passive first, blocking only when needed

Система не должна быть modal на каждый write.

Правильный UX:
	•	при первом касании чужого контракта — мягкий сигнал;
	•	при накоплении изменений — consolidated review;
	•	blocking — только на high-risk контрактах, witness failures или явном behavioral drift без reconcile.

4.6 Tree-sitter owns the hot path

Всё, что можно решить локально и детерминистски, надо решать локально и детерминистски.

LLM не должен первым видеть весь diff и гадать о смысле.
Сначала symbolic slicing, потом уже semantic review.

4.7 Promotion, not accumulation

Не каждый task intent должен жить вечно.

Иначе папка .intent/ через месяц станет кладбищем.

Правильная модель:
	•	task intent — рабочая единица;
	•	contract intent — долговечный guard;
	•	после завершения task intent либо архивируется, либо промоутится в contract intent.

5. Основные артефакты

5.1 ADR

ADR отвечает на вопрос “почему система устроена так”.

Он фиксирует архитектурное решение, ограничение, trade-off.

ADR не должен быть hot-path guard’ом на каждый edit.

5.2 Task intent

Task intent отвечает на вопрос “что мы сейчас хотим изменить”.

Он может стартовать как одно предложение.
Он может быть кривым, неполным, разговорным.
Это нормально.

Его задача:
	•	запустить clarify/plan flow,
	•	дать агенту scope,
	•	собрать вокруг себя implementation work.

5.3 Contract intent

Contract intent отвечает на вопрос “что должно продолжать быть true”.

Это долговечный артефакт.
Именно он участвует в future guard’ах.

Contract intent может родиться:
	•	напрямую,
	•	из task intent после merge,
	•	как производное от ADR,
	•	как extracted invariant после нескольких похожих изменений.

5.4 Witness

Witness — это наблюдаемое доказательство того, что claim держится.

Witness не равен тесту, хотя тест — частный случай witness.

Witness может быть:
	•	unit/integration/property test;
	•	route inventory;
	•	public API snapshot;
	•	schema diff rule;
	•	dependency rule;
	•	config invariant;
	•	generated query;
	•	smoke scenario.

Коротко:
	•	ADR — почему;
	•	intent — что должно быть true;
	•	witness — чем это проверяется;
	•	code — где это воплощено.

5.5 Trace

Trace — это machine-managed graph:
	•	claim → anchor
	•	claim → witness
	•	claim → ADR
	•	symbol → intent
	•	diff → impacted claims

Именно trace делает систему практичной.

6. Граница между intention, ADR и тестом

Это важный продуктовый выбор.

6.1 Intention не должен заменять ADR

Если решение уровня “мы используем event sourcing” или “все auth flows идут через gateway”, это ADR.

Если следствие этого решения — “все /api/v2/* endpoints обязаны проходить через middleware X”, это contract intent.

6.2 Intention не должен заменять тест

Тест проверяет исполнимое поведение.
Intention фиксирует контракт и границы.

Тест может сказать:
	•	при expired token возвращается 401.

Intention говорит:
	•	весь класс endpoints обязан быть защищён;
	•	refresh flow не должен затрагиваться;
	•	исключения перечислены явно.

То есть тест — evidence.
Intention — claim.

6.3 Bugfix и refactor требуют особой формы

Здесь я бы прямо заимствовал хорошую мысль из Kiro bugfix flow: bugfix должен явно содержать текущее поведение, ожидаемое поведение и то, что обязано остаться неизменным; Kiro ещё и строит property-based tests вокруг fix + preservation. Это очень сильный шаблон, и его стоит адаптировать в intent-файлы.  ￼

Поэтому у Anchor должны быть intent kinds:
	•	feature
	•	bugfix
	•	invariant
	•	refactor
	•	migration
	•	ADR-implementation

У каждого — свой template и свои guard rules.

7. UX: как это должно ощущаться

7.1 Создание работы

Лучший UX здесь не “заполни форму из 18 полей”.

Лучший UX такой:
	1.	Разработчик пишет файл:
	•	.intent/tasks/add-rate-limit.md
	•	в свободной форме
	2.	На следующем запуске агент говорит:
	•	нашёл draft intent;
	•	могу взять его в работу;
	•	мне нужно 1–3 уточнения, если они правда нужны
	3.	После уточнений система нормализует файл:
	•	выделяет claims
	•	определяет kind
	•	предлагает boundaries
	•	предлагает candidate witnesses
	•	создаёт trace sidecar
	4.	Сессия привязывается к этому task intent.

Никакого отдельного “еперь открой special planner UI”.
Файл сам является точкой входа.

7.2 Работа внутри intention

Когда task intent активирован, агент получает в контекст только то, что нужно:
	•	summary
	•	claims
	•	non-goals
	•	linked ADR excerpts
	•	текущие anchors
	•	relevant witnesses
	•	открытые вопросы

Не весь markdown, не весь trace graph, а компактный working brief.

В Pi это красиво ложится на before_agent_start: extension может inject’ить persistent context и подкладывать turn-scoped system prompt с active task и relevant contracts. Pi для этого уже даёт нужные lifecycle events и user UI primitives.  ￼

7.3 Во время edits

Во время работы система не должна постоянно орать.

Она делает следующее:
	•	на write / edit / file-modifying bash diff режется на символы;
	•	touched symbols маппятся на claims и contracts;
	•	если всё в рамках active task — ничего не происходит, кроме тихого статуса;
	•	если задет внешний contract intent — появляется non-blocking signal.

Например в footer:

TASK-011 active | touching CON-004:C2 | review pending

Или тост:

Изменение задело contract CON-004, claim C2. Пока не блокирую.

Это лучший UX, чем modal на каждую запись.

7.4 Перед коммитом или в конце агентской сессии

Здесь нужен единый review screen.

Не серия попапов, а одна сводка:

Intent impact review

Current task
- TASK-011 rate limiting on /token

Impacted contracts
- CON-004 JWT boundary
  - touched claims: C2
  - junior review: likely_refactor
  - witnesses: unchanged

- CON-009 refresh flow
  - touched claims: C1, C3
  - junior review: behavioral_change
  - witnesses: 1 changed, 1 failing

Actions
[continue as refactor]
[attach to current task]
[create follow-up task]
[reconcile CON-009]
[abort]

Вот это уже очень хороший UX.

7.5 Если код меняют без intention

Это отдельный и очень важный flow.

Если в репо появляется “материальный” diff, который:
	•	меняет primary anchors,
	•	меняет witnesses,
	•	задевает durable contracts,
	•	и при этом не привязан к active task intent,

система должна спросить не “хочешь ли ты создать тикет”, а:
	•	привязать к существующему task intent;
	•	создать новый task intent из diff;
	•	подтвердить, что это чистый refactor;
	•	отменить change.

Это убирает класс дрейфа “код изменился, а намерения нет”.

7.6 Если intention устарел относительно кода

Нужен обратный flow.

Команды:
	•	intent status
	•	intent doctor
	•	CI report

Они показывают:
	•	stale contracts
	•	orphaned anchors
	•	claims without witnesses
	•	overlapping contracts
	•	task intents, которые давно закончены, но не архивированы
	•	ADRs, после изменения которых связанные contracts нужно review’ить

Это сторона “intent to code”, а не “code to intent”.

8. Предлагаемая модель файлов

8.1 Layout

docs/
  adr/
    ADR-003-auth-boundary.md

.intent/
  tasks/
    TASK-011-rate-limit-token.md
  contracts/
    CON-004-jwt-boundary.md
    CON-009-refresh-flow.md
  traces/
    TASK-011.json
    CON-004.json
    CON-009.json
  cache/                # gitignored
  reports/              # optional, gitignored or generated in CI

8.2 Human-readable intention file

---
id: CON-004
kind: invariant
status: active
promoted_from: TASK-002
adr: [ADR-003]
risk: high
---

# JWT boundary for /api/v2

## Outcome
All /api/v2 endpoints require a valid JWT except explicitly listed exclusions.

## Claims
- C1. Every /api/v2/* route is protected by gateway JWT middleware unless listed in Exclusions.
- C2. Expired or invalid JWT returns 401 and never 500.
- C3. Refresh-token flow is handled separately and is not broken by gateway validation changes.

## Exclusions
- /api/v2/auth/token
- /api/v2/auth/refresh

## Must remain unchanged
- P1. Existing refresh-token semantics remain intact.
- P2. Public profile endpoints outside /api/v2 are unaffected.

## Acceptance signals
- W1. Route inventory contains no unguarded /api/v2 route beyond exclusions.
- W2. Integration auth.jwt.required passes.
- W3. Refresh-flow smoke test passes.

## Notes
- Derived from ADR-003.

Мой вкус: section Claims — обязательная, Acceptance signals — почти обязательная, giant History — не нужна. История уже в git.

8.3 Machine sidecar

{
  "id": "CON-004",
  "kind": "invariant",
  "claims": {
    "C1": {
      "anchors": [
        { "type": "symbol", "id": "ts://src/gateway/jwt.ts#validateJWT", "fingerprint": "..." },
        { "type": "route", "id": "ts://src/api/v2/routes.ts#/api/v2/*" }
      ],
      "witnesses": [
        { "type": "test", "id": "tests/integration/auth.jwt.required.test.ts" },
        { "type": "inventory", "id": "route_inventory:/api/v2/*" }
      ]
    }
  },
  "secondary_anchors": [],
  "related_adr": ["ADR-003"],
  "stale": false,
  "confidence": 0.91,
  "decision_log": []
}

Именно здесь живут:
	•	fingerprints
	•	symbol IDs
	•	confidence
	•	stale flags
	•	review decisions
	•	reconcile metadata

Не в markdown.

9. Техническая архитектура проверки

9.1 Layer 1: local symbolic engine

Это hot path.

Инструменты:
	•	Git diff
	•	Tree-sitter
	•	ripgrep
	•	test discovery
	•	language-specific extractors

Роль Tree-sitter:
	•	найти enclosing symbols для каждого changed hunk;
	•	извлечь routes, exports, components, schemas, tests;
	•	построить normalized AST fingerprint;
	•	отличать whitespace/comment churn;
	•	переживать rename/move лучше, чем line-based mapping;
	•	давать compact structural summary для reviewer model.

Tree-sitter обязателен.
Без него UX быстро деградирует в file-level noise.

9.2 Layer 2: semantic expansion

Здесь уже можно подключать LSP или language-specific analyzers.

Но не в hot path.

Роль LSP:
	•	references
	•	implementations
	•	call hierarchy
	•	cross-file impact
	•	deep expansion of secondary anchors

Моя сильная ставка: LSP нужен для map --deep, doctor, PR review и reconcile quality.
Не нужен как обязательный blocking dependency на каждый edit.

9.3 Layer 3: junior reviewer model

Здесь живёт “младшая LLM”.

Её задача узкая:
	•	классифицировать impacted claim:
	•	aligned
	•	likely_refactor
	•	behavioral_change
	•	uncertain
	•	дать короткий rationale
	•	предложить action

Она получает не весь repo, а только:
	•	impacted claims
	•	diff slices
	•	structural summaries из Tree-sitter
	•	witness delta
	•	current task summary
	•	linked ADR excerpt, если есть

Сейчас это уже хорошо совпадает с тем, как сами harnesses и model providers советуют использовать subagents: узко, явно и на focused tasks. Codex спавнит subagents только по явному запросу, OpenAI отдельно позиционирует smaller models вроде GPT‑5.4 mini как подходящие для lighter subagent work, а Augment прямо рекомендует делать subagents специализированными и не превращать их в open-ended multi-step generalists.  ￼

Именно здесь “дешёвый ревьюер” очень уместен.

9.4 Layer 4: reconcile writer

Сильная модель нужна не для каждого check.

Она нужна для двух вещей:
	•	сформулировать хорошие clarification questions, когда draft intent туманный;
	•	подготовить reconcile patch для human-readable intent/contract file.

И важный продуктовый принцип:

reconcile — это предложенный patch, а не silent rewrite.

Система не должна переписывать историческую правду за тебя.

10. Когда и где происходят проверки

10.1 Intent → code

Проверяем в четыре момента:
	1.	При активации task intent
	•	генерируем claims, boundaries, candidate anchors
	2.	Перед агентским проходом
	•	inject scope в context
	3.	После крупного implementation step
	•	обновляем trace и candidate witnesses
	4.	При закрытии task
	•	архивируем или промоутим в contract

10.2 Code → intent

Проверяем в пять моментов:
	1.	На write/edit/bash с file changes
	•	лёгкий локальный impact preview
	2.	На agent_end
	•	consolidated review
	3.	На pre-commit
	•	authoritative local gate
	4.	В CI / PR
	•	branch-level contract impact report
	5.	На branch switch / session resume
	•	stale intent scan

10.3 Intent ↔ ADR

Проверяем в два момента:
	1.	Когда task/contract repeatedly widens scope
	•	предложить extraction в ADR
	2.	Когда ADR меняется
	•	все linked contracts получают context_stale

ADR не должен обновляться автоматически.
Только suggested patch.

11. Относительно Pi

Pi — очень хороший первый UX surface для этого продукта не потому, что “легче”, а потому что он даёт ровно те крючки, которые нужны: before_agent_start, tool_call, tool_result, agent_end, session_start, плюс ctx.ui для confirm/select/input/notify/status/widget. Pi также намеренно не вшивает plan mode и subagents в core, так что intention-first layer органично живёт как extension, а не как форк harness’а.  ￼

Конкретно я бы использовал так:
	•	session_start
	•	показать active task, stale contracts, unresolved clarifications
	•	before_agent_start
	•	inject current task + linked contracts + ADR excerpts
	•	tool_call
	•	intercept write, edit, file-modifying bash
	•	запустить fast impact analysis
	•	при high-risk вернуть block/confirm
	•	tool_result
	•	аннотировать результат impacted intents
	•	agent_end
	•	показать consolidated review
	•	предложить reconcile / archive / promote
	•	custom commands
	•	/intent new
	•	/intent work
	•	/intent status
	•	/intent reconcile
	•	/intent doctor

12. Как это живёт относительно ADR

Мой вкус здесь очень конкретный.

ADR и intentions должны жить рядом, но не сливаться.

Хорошая модель:
	•	ADR — редкий, тяжёлый, архитектурный
	•	contract intent — более частый, поведенческий, guardable
	•	task intent — рабочий, временный

Связи:
	•	один ADR может породить несколько contract intents;
	•	один task intent может ссылаться на ADR и при этом трогать несколько contracts;
	•	если task intent после merge вводит новый долгоживущий контракт, он промоутится в contract intent;
	•	если несколько contracts делят одну и ту же rationale, система предлагает вынести rationale в ADR.

То есть ADR — parent rationale.
Intent — executable contract layer.

13. PR / review UX

Лучший PR UX не должен показывать только diff markdown-файлов.

Он должен показывать intent impact summary:
	•	какие task intents появились;
	•	какие contracts затронуты;
	•	какие claims изменены;
	•	какие witnesses изменились;
	•	что помечено как refactor;
	•	что требует reconcile;
	•	что стало stale.

Идеальный CI report:

Intent report

New tasks
- TASK-011 rate limiting on /token

Changed contracts
- CON-009 refresh flow
  - claims changed: C1, C3
  - reconcile patch present

Uncertain impacts
- CON-004 JWT boundary
  - claim C2 touched
  - reviewer verdict: uncertain

Stale artifacts
- none

Блокирующее правило должно быть узким:

CI block только если:
	•	changed durable contract без recorded decision;
	•	witness failure у durable contract;
	•	missing intent for significant behavioral change.

Не блокировать за всё подряд.

14. Метрики успеха

Продукт жив, если он даёт signal, а не просто красивые файлы.

Основные метрики:
	•	useful warning rate
	•	сколько раз разработчик реально сказал “хорошо, что меня остановило”
	•	false positive rate
	•	отдельно для soft warnings и blocking gates
	•	unowned change rate
	•	доля существенных diffs без task intent
	•	durable contract health
	•	доля contract intents с хотя бы одним witness
	•	доля stale contracts
	•	latency
	•	median local symbolic check
	•	median junior review check
	•	reconcile quality
	•	сколько reconcile patches принимаются почти без правок
	•	archive discipline
	•	доля task intents, которые либо архивируются, либо промоутятся, а не висят бесконечно

15. Что продукт не должен делать

Очень важно удержать границы.

Продукт не должен:
	•	превращаться в новую Jira;
	•	требовать вручную поддерживать code map;
	•	хранить giant auto-generated sections в markdown;
	•	модалить пользователя на каждый edit;
	•	silently переписывать intent/ADR файлы;
	•	заставлять весь workflow проходить через жёсткий spec-first ceremony;
	•	обещать perfect semantic understanding of arbitrary legacy code.

16. Моя самая сильная рекомендация по форме v1

Если смотреть не как на research paper, а как на продукт, я бы делал v1 не как “универсальную систему intentions”, а как три чётких сущности:
	•	task intent
	•	contract intent
	•	trace

И один главный UX loop:
	1.	набросал task intent;
	2.	агент уточнил и взял в работу;
	3.	изменения в коде маппятся на claims/contracts;
	4.	если задет чужой контракт — появляется review;
	5.	если контракт реально изменён — предлагается reconcile patch;
	6.	после merge task либо архивируется, либо становится contract.

