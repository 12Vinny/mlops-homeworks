# Threat Model & Risk Register: LexiGuard AI

Цей документ описує поверхню атак нашого Legal-tech продукту та приоритизує загрози.

## Attack surface narrative
Система LexiGuard AI обробляє конфіденційні M&A контракти через Web-застосунок (Client Application). Це формує першу лінію атаки: користувачі можуть завантажувати спеціально змінені файли (Adversarial inputs) або вводити зловмисні промпти для маніпуляції логікою LLM. Друга лінія — це внутрішній RAG-компонент (Pinecone Vector DB), який зберігає корпоративні політики. Оскільки архітектура є multi-tenant, збій ізоляції призведе до розкриття секретів одного enterprise-клієнта іншому. Третя лінія — це AWS EKS інфраструктура (Model inference) та CI/CD-пайплайни (GitLab), де відсутність жорсткого Role-Based Access Control (RBAC) створює ризики крадіжки ваг моделей або виснаження дорогих GPU-ресурсів через DDoS.

## Threat Table

| ID | Category | Threat | Attack surface | Business impact | Likelihood | Priority | Owner | Mitigation | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **T-001** | LLM-specific | **Direct prompt injection** | Web API / File Input | $120K (втрата клієнта через генерацію хибного драфту) | High | **P1** | Head of AI Safety | Впровадження вхідного Guardrails-фільтра та строгий System Prompt. | In progress |
| **T-002** | LLM-specific | **High-stakes hallucinations** | Inference API | $100K+ (судовий позов за юридичну помилку моделі) | Medium | **P1** | ML Lead | LLM-as-a-judge оцінка відповідей + RAG grounding. | Implemented |
| **T-003** | Infra/Product | **PII leakage через RAG** | Pinecone Vector DB | €10M+ (штраф GDPR за витік даних між клієнтами) | Low | **P1** | DevOps Lead | Жорстка ізоляція клієнтів на рівні Namespaces у Pinecone. | Implemented |
| **T-004** | Classic ML | **Adversarial examples** | ML Pipeline (Intent) | $50K (операційні збитки юристів через хибне сортування) | Medium | **P2** | ML Engineer | Robustness-тестування датасету перед кожним релізом. | Not started |
| **T-005** | LLM-specific | **Sensitive data leakage** | Inference API | $80K (репутаційна шкода від генерації тренувальних даних) | Low | **P2** | ML Lead | Сувора PII sanitization перед fine-tuning циклом. | Implemented |
| **T-006** | Infra/Product | **Insecure CI/CD access** | GitLab CI / IAM | $200K+ (крадіжка моделей або несанкціонований cryptomining) | Medium | **P2** | MLOps Lead | OIDC авторизація та Least Privilege IAM roles для пайплайнів. | In progress |
| **T-007** | Infra/Product | **Model exhaustion (DoS)** | API Gateway | $15-30K (рахунок від AWS за виснаження GPU-інстансів) | High | **P1** | Backend Lead | Rate-limiting на API Gateway + billing alerts. | Accepted risk |
| **T-008** | Classic ML | **Data poisoning** | Training Data | $30K (падіння метрик класифікації та витрати на retraining) | Medium | **P2** | Data Engineer | Валідація даних користувацького фідбеку перед навчанням. | In progress |
| **T-009** | LLM-specific | **Jailbreaking / policy bypass** | Inference API | $20K (PR-криза від токсичної генерації тексту) | Medium | **P2** | Head of AI Safety | Використання Llama-Guard на етапі виводу (Output filtering). | In progress |
| **T-010** | Infra/Product | **Insufficient audit logging** | MLflow / EKS | Регуляторний блок роботи в EU через невідповідність AI Act | Low | **P2** | Head of Compliance | Інтеграція MLflow з AWS CloudTrail, строге версіонування. | Implemented |
| **T-011** | Classic ML | **Model bias (Legal bias)** | ML Pipeline | $40K (відтік клієнтів з певних європейських юрисдикцій) | High | **P2** | AI PM | Моніторинг Data Drift у розрізі геолокації в Grafana. | Not started |
| **T-012** | Infra/Product | **Model supply chain attack** | GitLab CI | $150K+ (backdoor у базовій LLM моделі з HuggingFace) | Low | **P3** | MLOps Lead | Звірка SHA256 хешів base-моделей перед fine-tuning. | Accepted risk |
| **T-013** | LLM-specific | **Indirect prompt injection** | RAG Document Sync | $70K (маніпуляція виводом через завантажений шкідливий PDF) | Medium | **P2** | Backend Lead | Санітизація та перевірка тексту перед записом у векторну базу. | In progress |
| **T-014** | Classic ML | **Model extraction via API** | API Gateway | Втрата інтелектуальної власності конкурентам | Low | **P3** | Backend Lead | Тротлінг за IP/Token та моніторинг аномальних патернів запитів. | Accepted risk |
| **T-015** | Infra/Product | **S3 bucket public access** | AWS S3 (Artifacts) | €10M+ (GDPR breach тренувальних даних та контрактів) | Low | **P1** | DevOps Lead | S3 Block Public Access + автоматичне сканування Macie. | Implemented |