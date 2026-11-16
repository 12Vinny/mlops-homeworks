# Threat Model & Risk Register: LexiGuard AI

## 1. Attack Surface Overview

Система LexiGuard AI є B2B платформою, інтегрованою в юридичні процеси корпорацій. Її Attack Surface (поверхня атаки) включає кілька критичних меж:
1.  **Client Application (Web/API):** Точка входу, куди користувачі завантажують NDA та M&A контракти. Це вектор для введення зловмисних даних (Adversarial inputs, Prompt Injections), спрямованих на маніпуляцію логікою LLM.
2.  **RAG & Vector DB (Pinecone):** Сховище корпоративних політик. Якщо механізм ізоляції tenant-ів (Namespaces) дасть збій, виникає критичний ризик міжклієнтського витоку даних (Cross-tenant PII leakage).
3.  **Classic ML Pipeline (Intent Classification):** Вразливий до Data Poisoning під час retraining-циклів, якщо розмічені клієнтами дані потрапляють у навчальну вибірку без валідації.
4.  **Inference & Infrastructure (AWS EKS):** Хостинг LLM та API. Вразливий до DoS-атак (Model exhaustion), що вичерпують дорогі GPU-ресурси, та несанкціонованого доступу CI/CD пайплайнів (GitLab) до Kubernetes-секретів.

Ураховуючи, що система працює з європейськими клієнтами, будь-який витік даних чи неконтрольована генерація тексту підпадає під дію GDPR та нагляду EU AI Act (як high-risk system у юридичній сфері).

## 2. Threat Register

*Пріоритет (Priority) обчислюється як Impact × Likelihood.*

| ID | Category | Threat description | Attack surface | Business impact ($) | Likelihood (L/M/H) | Priority | Owner | Mitigation | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **T-001** | LLM (Prompt injection) | **Direct Prompt Injection:** Користувач додає прихований текст у контракт, щоб змусити LLM згенерувати драфт, вигідний іншій стороні. | API / File Upload | $100K+ (судові позови клієнтів), втрата довіри | High | **P1** | Head of AI Safety | Впровадження вхідного LLM-фільтра (Guardrails) та суворий System Prompt. | In progress |
| **T-002** | LLM (Hallucination) | **High-stakes Hallucination:** LLM придумує неіснуючий юридичний прецедент або ігнорує ключову політику (як в інциденті Q3). | Inference API | $120K ARR (втрата enterprise-контракту) | Medium | **P1** | ML Lead | Перевірка фактів через RAG (grounding), RAG-sync alerting, HITL (Manual review). | Implemented |
| **T-003** | Infra (Data Leakage) | **Cross-tenant PII leakage:** RAG повертає фрагмент контракту клієнта А у відповіді на запит клієнта Б. | Pinecone (RAG) | €10M+ (штраф GDPR), знищення компанії | Low | **P1** | DevOps Lead | Жорстка ізоляція на рівні Pinecone Namespaces, шифрування tenant-ключів. | Implemented |
| **T-004** | Classic ML | **Adversarial Examples:** Зміна форматування контракту змушує ML-модель класифікувати M&A як "безпечний NDA". | ML Pipeline | $50K (операційні збитки юристів) | Medium | **P2** | ML Engineer | Robustness тестування перед релізом (Release Checklist), adversarial training. | Not started |
| **T-005** | Infra (Security) | **Overprivileged CI/CD:** GitLab пайплайн має адмінські права в AWS і може бути скомпрометований. | GitLab CI / AWS IAM | $200K+ (крадіжка моделей/даних, cryptomining) | Medium | **P2** | MLOps Lead | OIDC авторизація, Least privilege IAM policies для пайплайнів. | In progress |
| **T-006** | LLM (Data Leakage) | **Sensitive data leakage у відповідях:** LLM відтворює конфіденційну інформацію, яку "завчила" під час fine-tuning. | Inference API | $50K+ (репутаційна шкода) | Low | **P2** | ML Lead | Анонімізація тренувальних даних (PII sanitization) перед fine-tuning. | Implemented |
| **T-007** | Infra (Resource) | **Model Exhaustion (DoS):** Зловмисник надсилає тисячі запитів для виснаження GPU-інстансів (Financial DoS). | API Gateway | $10-20K (Cloud bill spike) | High | **P1** | Backend Lead | Rate-limiting на API Gateway, автоматичний алертинг по cloud-бюджету. | Accepted risk (monitoring only) |
| **T-008** | Classic ML | **Data Poisoning:** Клієнти навмисно або