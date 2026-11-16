# Architectural Decision Records для LexiGuard AI

## ADR-001: Model Hosting Strategy (LLM & Classic ML)

Status: Accepted
Date: 2026-11-14
Decision maker: Senior AI PM
Stakeholders consulted: CTO, MLOps Lead, CFO, Head of Compliance, DPO

## Context
LexiGuard AI обробляє ~1M висококонфіденційних контрактів щомісяця (M&A, NDA) для 800 B2B клієнтів у EU та US. Відповідно до GDPR та EU AI Act, ми маємо жорсткі вимоги щодо конфіденційності даних (Data Privacy). Використання комерційних API (OpenAI/Anthropic) для LLM створює ризик передачі даних клієнтів третім сторонам, що блокує продажі в enterprise-сегменті. 

## Decision
Ми обираємо Self-hosted гібридну архітектуру. Класична ML-модель (Intent classification) хоститиметься на власному AWS EKS кластері через TorchServe на CPU-нодах. LLM-компонент (fine-tuned модель рівня Llama 3) хоститиметься на виділених GPU-інстансах у нашому приватному VPC.

## Alternatives considered
*   **Alternative 1: Commercial API (OpenAI / Anthropic):** 
    *   *Trade-off:* Економить ~$25K/міс на інфраструктурі, але створює критичний блокер для compliance (неможливо гарантувати ізоляцію M&A даних), що відлякує enterprise-клієнтів.
*   **Alternative 2: Managed Hosting (Amazon Bedrock Provisioned Throughput):** 
    *   *Trade-off:* Дані не покидають AWS (вирішує compliance), проте базова вартість provisioned throughput для нашої кастомної моделі складатиме ~$50-60K/міс, що економічно невигідно.
*   **Alternative 3: Serverless Inference (SageMaker Serverless):**
    *   *Trade-off:* Хороший вибір для нерегулярного навантаження, але при нашому стабільному обсязі (1M+ запитів) генеруватиме величезні рахунки за мілісекунди роботи та матиме проблему "cold starts" (висока затримка для першого запиту).

## Consequences
Positive:
- Повний контроль над даними (Data Sovereignty), що розблоковує enterprise-продажі та гарантує GDPR compliance.
- Фіксована вартість інференсу при подальшому масштабуванні обсягів.
Negative:
- Високі витрати на compute: ~$35-40K/міс на AWS GPU.
- Потреба у +1 FTE MLOps Engineer для обслуговування LLM-serving.
Risks:
- Дефіцит GPU в обраному AWS-регіоні (eu-central-1). Fallback: налаштування резервних нод у us-east-1 із затримкою +60ms.

## Related ADRs
- ADR-003
- ADR-004

---

## ADR-002: RAG Storage & Tenant Isolation Strategy

Status: Accepted
Date: 2026-11-14
Decision maker: Senior AI PM
Stakeholders consulted: MLOps Lead, Head of Security, Backend Lead

## Context
RAG-компонент LexiGuard AI оперує ~2.5M векторів (внутрішні політики клієнтів). Минулого місяця виник інцидент, коли LLM використала застарілу політику клієнта. Окрім того, будь-який витік контексту між клієнтом А та клієнтом Б (cross-tenant leakage) загрожує компанії штрафами та репутаційним знищенням.

## Decision
Ми використовуємо Pinecone Serverless як векторну базу даних. Для забезпечення суворої безпеки кожен клієнт отримуватиме окремий логічний Namespace, а запити додатково обмежуватимуться через Metadata filtering (RBAC).

## Alternatives considered
*   **Alternative 1: Self-hosted pgvector (PostgreSQL):** 
    *   *Trade-off:* Дозволяє тримати вектори разом з основними даними (дешево), проте дуже складно горизонтально масштабувати пошук на 2.5M+ high-dim векторів без втрати швидкості.
*   **Alternative 2: Milvus (Self-hosted на EKS):** 
    *   *Trade-off:* Повний контроль та висока продуктивність, але додає величезний operational overhead на команду, яка мусить підтримувати ще один stateful кластер.
*   **Alternative 3: Managed AWS OpenSearch:**
    *   *Trade-off:* Сильна інтеграція в екосистему AWS, проте набагато складніше налаштувати сувору логічну ізоляцію (Namespaces) на рівні окремих документів для 800 клієнтів, порівняно з Pinecone.

## Consequences
Positive:
- Zero operational overhead на підтримку кластера бази (fully managed).
- Гарантована апаратна та логічна ізоляція tenant-ів на рівні Namespaces.
Negative:
- Vendor lock-in на платформу Pinecone.
- Ризик непрогнозованого зростання витрат при масштабуванні до 10M+ векторів.
Risks:
- Затримка синхронізації між нашою основною реляційною базою та Pinecone. 

## Related ADRs
- ADR-001

---

## ADR-003: Deployment Pattern для AI Компонентів

Status: Accepted
Date: 2026-11-14
Decision maker: Senior AI PM
Stakeholders consulted: VP Engineering, MLOps Lead

## Context
Деградація класичної ML-моделі або раптові галюцинації нової версії LLM безпосередньо впливають на бізнес клієнтів. Ми не можемо дозволити собі перемикати 100% трафіку на нову модель миттєво, оскільки rollback потребує часу, протягом якого можуть генеруватися некоректні юридичні драфти.

## Decision
Ми впроваджуємо Canary Deployment (через ArgoCD Rollouts) як єдиний стандарт розгортання для LLM та ML. Трафік перемикатиметься поетапно (5% -> 25% -> 50% -> 100%), з обов'язковою автоматичною паузою для аналізу метрик на кожному етапі.

## Alternatives considered
*   **Alternative 1: Blue-Green Deployment:** 
    *   *Trade-off:* Дозволяє миттєвий rollback, але вимагає тимчасового дублювання інфраструктури (х2 GPU інстансів) на момент деплою, що неприпустимо дорого для LLM.
*   **Alternative 2: Shadow Deployment:** 
    *   *Trade-off:* Нульовий ризик для користувача (трафік дублюється непомітно), але він подвоює вартість інференсу і не дає реальних метрик клієнтської реакції (UX).
*   **Alternative 3: A/B Testing Deployment:**
    *   *Trade-off:* Відмінно підходить для перевірки бізнес-гіпотез, але не має вбудованого автоматичного відкату (rollback) при технічних збоях чи стрибках галюцинацій.

## Consequences
Positive:
- Мінімізація зони ураження (blast radius) до 5% користувачів при помилках нової моделі.
- Економічно ефективно (балансується в межах існуючих compute-ресурсів).
Negative:
- Збільшення тривалості релізу (від кількох хвилин до кількох годин).
Risks:
- Відсутність sticky sessions на балансувальнику може призвести до того, що клієнт отримуватиме відповіді від різних версій моделі у межах однієї сесії.

## Related ADRs
- ADR-001
- ADR-004

---

## ADR-004: Unified Monitoring Stack

Status: Accepted
Date: 2026-11-14
Decision maker: Senior AI PM
Stakeholders consulted: MLOps Lead

## Context
Для підтримки Canary-релізів та швидкого реагування на інциденти нам потрібен єдиний стек моніторингу, який відстежує стан інфраструктури (CPU, RAM, GPU), метрики класифікації (Data Drift) та метрики генерації LLM (Hallucination rate).

## Decision
Ми використовуємо комбінований стек: Prometheus/Grafana/Loki для відстеження операційних метрик (latency, errors) та логів, а MLflow для відстеження ML-метрик, дрифту та як Model Registry. 

## Alternatives considered
*   **Alternative 1: Datadog:** 
    *   *Trade-off:* Потужний "все-в-одному", але дуже дорогий при наших обсягах інференсу (~$15K+/міс лише за retention логів).
*   **Alternative 2: SageMaker Model Registry + CloudWatch:** 
    *   *Trade-off:* Сильна інтеграція з AWS, але блокує нас у закритій екосистемі SageMaker і має слабший функціонал для кастомного LLM evaluation порівняно з MLflow.
*   **Alternative 3: ELK Stack (Elasticsearch, Logstash, Kibana) + MLflow:**
    *   *Trade-off:* Класичний потужний інструмент для логів, але вимагає значно більше ресурсів на адміністрування (управління JVM) порівняно з легшим Grafana Loki.

## Consequences
Positive:
- Оптимізація витрат (відмова від дорогих SaaS-рішень для логів на користь open-source).
- Використання стандарту (MLflow), з яким вже знайома наша команда (zero learning curve).
Negative:
- Вимагає ручних зусиль інженерів на розробку кастомних LLM-дашбордів у Grafana.
Risks:
- Якщо MLflow сервер впаде, ми тимчасово втратимо можливість трекати нові метрики під час training pipeline.

## Related ADRs
- ADR-001
- ADR-003