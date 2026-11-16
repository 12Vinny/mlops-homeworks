# Architectural Decision Records для LexiGuard AI

## ADR-001: Model Hosting Strategy (LLM & Classic ML)

*   **Status:** Accepted
*   **Date:** 2026-11-14
*   **Decision maker:** Senior AI PM (в консультації з CTO та Head of Compliance)
*   **Stakeholders consulted:** CTO, MLOps Lead, CFO, Head of Compliance, DPO

### Context
LexiGuard AI обробляє ~1M висококонфіденційних контрактів щомісяця (M&A, NDA) для 800 B2B клієнтів у EU та US. Відповідно до GDPR та EU AI Act, ми маємо жорсткі вимоги щодо конфіденційності даних (Data Privacy). Використання публічних API для LLM створює ризики передачі клієнтських даних третім сторонам, що є неприпустимим для enterprise-юристів.

### Decision
Ми обираємо **Self-hosted гібридну архітектуру**:
1. Класична ML-модель (Intent classification) хоститься на нашому власному AWS EKS кластері через TorchServe (CPU-ноди).
2. LLM-компонент (fine-tuned модель рівня Llama 3 70B) хоститься на виділених GPU-інстансах (або AWS Inferentia) у нашому приватному VPC через vLLM / TGI.

### Alternatives considered
*   **Alternative 1: Commercial API (OpenAI / Anthropic)**
    *   *Pros:* Нульовий operational overhead, оплата лише за токени, найвища якість baseline.
    *   *Cons:* Критичні блокери по compliance, тому що не можемо гарантувати ізоляцію M&A контрактів, vendor lock-in.
    *   *Estimated cost:* ~$15K/міс.
    *   *Why not chosen:* Не проходить аудит безпеки від клієнтів (enterprise-сегмент відмовиться від продукту).
*   **Alternative 2: Managed Hosting (Amazon Bedrock з Provisioned Throughput)**
    *   *Pros:* Дані не покидають AWS, хороша безпека, менший DevOps overhead.
    *   *Cons:* Дуже висока базова вартість Provisioned Throughput для fine-tuned моделей; менше контролю над оптимізацією інференсу.
    *   *Estimated cost:* ~$50-60K/міс.
    *   *Why not chosen:* Знижує нашу unit-економіку та обмежує гнучкість у тюнінгу.

### Consequences
*   **Positive:**
    *   Повний контроль над даними (Data Sovereignty), що розблоковує enterprise-продажі (уникнення штрафів GDPR).
    *   Фіксована вартість інференсу при великих обсягах (економія на масштабі).
*   **Negative:**
    *   Високі інфраструктурні витрати (compute costs): ~$35-40K/міс на GPU/Inferentia.
    *   Потрібен +1 FTE MLOps Engineer для підтримки GPU-інфраструктури та LLM-serving (~$120K/рік).
*   **Risks:**
    *   Дефіцит GPU інстансів в обраному AWS-регіоні (eu-central-1). Fallback-план: налаштувати крос-регіональний кластер із допустимою затримкою (latency +60ms).

### Success metrics
*   Compliance: Успішне проходження SOC2 Type II та ISO 27001 аудитів.
*   Cost: Blended cost per prediction залишається < $0.03.

---

## ADR-002: RAG Storage & Tenant Isolation Strategy

*   **Status:** Accepted
*   **Date:** 2026-11-14
*   **Decision maker:** Senior AI PM
*   **Stakeholders consulted:** MLOps Lead, Head of Security

### Context
Наш RAG-компонент оперує ~2.5M векторів (внутрішні політики та контракти клієнтів). Інцидент минулого місяця показав, що несвоєчасне оновлення векторної бази (GDPR-політики) може коштувати нам контракту на $120K ARR. Більше того, витік даних клієнта А в контекст клієнта Б (cross-tenant leakage) знищить репутацію LexiGuard AI.

### Decision
Ми використовуємо **Pinecone Serverless** як векторну базу даних із суворою ізоляцією: **окремий логічний Namespace для кожного клієнта** + Metadata filtering (Role-Based Access Control на рівні документа). 

### Alternatives considered
*   **Alternative 1: Self-hosted pgvector (PostgreSQL)**
    *   *Pros:* Зберігання векторів поруч із реляційними даними, економія на ліцензіях.
    *   *Cons:* Важко горизонтально масштабувати при 2.5M+ high-dim векторах; повільний пошук.
*   **Alternative 2: Milvus (Self-hosted на EKS)**
    *   *Pros:* Відкритий код, потужна продуктивність.
    *   *Cons:* Високий operational overhead; команда має підтримувати ще один stateful сервіс у Kubernetes.

### Consequences
*   **Positive:**
    *   Zero operational overhead на підтримку бази (повністю managed).
    *   Апаратна та логічна ізоляція tenant-ів через Namespaces мінімізує ризик витоку даних.
*   **Negative:**
    *   SaaS vendor lock-in.
    *   Зростання вартості при масштабуванні до 10M+ векторів.
*   **Risks:**
    *   Затримка синхронізації між нашою основною базою та Pinecone (саме це призвело до минулого інциденту). Mitigation: впровадження event-driven архітектури для миттєвого оновлення векторів при зміні політик.

---

## ADR-003: Deployment Pattern для AI Компонентів

*   **Status:** Accepted
*   **Date:** 2026-11-14
*   **Decision maker:** Senior AI PM
*   **Stakeholders consulted:** VP Engineering, MLOps Lead

### Context
Помилки в інференсі класичної ML-моделі або галюцинації LLM мають прямий фінансовий вплив. Ми не можемо дозволити собі перемикати 100% трафіку на нову модель миттєво, оскільки rollback займає час, протягом якого генеруються хибні драфти контрактів.

### Decision
Ми впроваджуємо **Canary Deployment (через ArgoCD Rollouts)** як єдиний стандарт розгортання як для класичних ML-моделей, так і для LLM. Трафік перемикається поетапно (наприклад: 5% -> 25% -> 50% -> 100%) з автоматичним аналізом метрик (Prometheus) на кожному кроці.

### Alternatives considered
*   **Alternative 1: Blue-Green Deployment**
    *   *Pros:* Миттєвий і безпечний rollback (перемикання роутера).
    *   *Cons:* Вимагає дублювання інфраструктури. Тримати x2 GPU інстансів для LLM під час релізу економічно недоцільно (cost spike).
*   **Alternative 2: Shadow Deployment**
    *   *Pros:* 0% ризику для клієнта.
    *   *Cons:* Не дає реального бізнес-фідбеку та UX-метрик; подвоює вартість інференсу під час тіньового прогону.

### Consequences
*   **Positive:**
    *   Мінімізація "blast radius" (зони ураження) при деградації моделі до 5-10% трафіку.
    *   Економічно ефективніше за Blue-Green, оскільки трафік балансується між існуючими ресурсами.
*   **Negative:**
    *   Більш тривалий час релізу (може займати години).
    *   Ускладнення логіки метрик (потрібно порівнювати canary-поди з baseline-подами).
*   **Risks:**
    *   Користувач може отримати різні результати в межах однієї сесії, якщо балансувальник не налаштований на "sticky sessions".

---

## ADR-004: Unified Monitoring Stack

*   **Status:** Accepted
*   **Date:** 2026-11-14
*   **Decision maker:** Senior AI PM
*   **Stakeholders consulted:** MLOps Lead

### Context
Нам потрібен єдиний центр управління для моніторингу EKS інфраструктури (CPU, RAM, GPU), класичного ML (Data Drift, Concept Drift) та LLM (Hallucination rate, Toxicity, Latency).

### Decision
Ми використовуємо **Prometheus/Grafana/Loki** для моніторингу інфраструктури та операційних метрик (latency, error rates), а **MLflow (з MLflow LLM Evaluate)** для відстеження ML-специфічних метрик, дрифту та управління реєстром моделей.

### Alternatives considered
*   **Datadog:** Потужний, але вкрай дорогий при великих обсягах логів.
*   **Evidently AI +