# Incident Runbook: LexiGuard AI

Цей документ визначає стандартизовані процедури реагування на критичні інциденти в production-середовищі LexiGuard AI.

---

## Scenario 1: Model Quality Degradation (Classic ML Drift)

*   **Severity:** P2
*   **Expected frequency:** 1-2 рази на рік (часто через зміни у шаблонах контрактів клієнтів)
*   **MTTR target:** 2 години до mitigation, 2 дні до full retraining & recovery

### 1. Detection
**Тригери:**
*   Grafana alert: `Intent_Classification_Confidence_Drop` (середня впевненість моделі впала нижче 80% за останні 4 години).
*   Manual report: Команда Customer Success фіксує скарги клієнтів на те, що M&A контракти позначаються як NDA.

### 2. Initial response (first 15 minutes)
- [ ] 0-2 хв: MLOps Engineer on-call підтверджує alert у Grafana (виключає false positive через сплеск нетипового трафіку).
- [ ] 2-5 хв: Створити incident channel `#inc-YYYY-MM-DD-ml-drift`.
- [ ] 5-10 хв: Запросити в канал: ML Lead, AI PM, Head of Support.
- [ ] 10-15 хв: AI PM публікує Initial assessment (вплив на клієнтів) у каналі.

### 3. Diagnosis
**Дашборди для перевірки:**
*   Grafana: `TorchServe_Prediction_Distribution` (чи змінився розподіл класів).
*   Evidently AI / MLflow: Перевірка Data Drift (чи змінилися довжина, формат або словник вхідних контрактів).
**Ключові питання:**
*   Чи деградація спостерігається у всіх клієнтів, чи лише в одного (специфічний кастомний шаблон)?
*   Чи були релізи нових версій моделі за останні 48 годин?

### 4. Mitigation
1.  **Fallback to Rules:** Якщо помилки критичні, тимчасово активувати rule-based евристику (через Feature Flag у бекенді), яка надсилає всі непевні контракти (confidence < 85%) на ручне рев'ю юристів клієнта.
2.  **Rollback:** Якщо деградація спричинена вчорашнім Canary-релізом, зробити `git revert` у GitOps репозиторії для повернення до попередньої стабільної версії моделі (v.X.Y). Approver: MLOps Lead.

### 5. Communication plan
*   **Internal:** Оновлення в Slack `#inc-...` кожні 60 хв. Notification для CPO.
*   **External:** Не потребує публічної заяви (status page). Customer Success team попереджає зачеплених клієнтів індивідуально про "можливі затримки автоматичної класифікації".

### 6. Recovery
Інцидент вважається закритим, коли метрики confidence стабілізуються > 85% протягом 12 годин після відкату або застосування fallback-логіки.

### 7. Post-mortem checklist
- [ ] Blameless post-mortem заплановано протягом 5 днів.
- [ ] Root cause identified (напр., новий тип M&A укладених за французьким правом).
- [ ] Update Threat Model: T-012 (Model Drift).
- [ ] Створити Jira-тікет на збір нових даних та Retraining моделі.

### 8. Preventive measures
*   Налаштувати більш гранулярний моніторинг Data Drift у розрізі окремих клієнтів (tenant-level drift detection).

---

## Scenario 2: High-stakes LLM Hallucination (GDPR Policy Ignore)

*   **Severity:** P1
*   **Expected frequency:** < 1 разу на рік
*   **MTTR target:** 30 хвилин до mitigation, 4 години до root cause analysis

### 1. Detection
**Тригери:**
*   External report: Клієнт екскалює проблему через Account Manager'а — LLM згенерувала юридичний драфт, який порушує їхню нову Data Privacy політику.
*   Grafana alert: `RAG_Context_Relevance_Low` (LLM генерує відповідь, спираючись на низькорелевантні вектори з Pinecone).

### 2. Initial response (first 15 minutes)
- [ ] 0-2 хв: AI PM отримує ескалацію, підтверджує критичність (юридичний ризик).
- [ ] 2-5 хв: Створити incident channel `#inc-YYYY-MM-DD-llm-hallucination`.
- [ ] 5-10 хв: Запросити в канал: Backend Lead, ML Lead, Head of Compliance, DPO, CPO.
- [ ] 10-15 хв: Активувати режим "Кризової комунікації".

### 3. Diagnosis
**Дашборди/Логи для перевірки:**
*   Loki / CloudWatch: Знайти `trace_id` конкретної сесії клієнта.
*   MLflow Tracing (LangSmith / Phoenix): Проаналізувати, які саме шматки політики RAG витягнув з Pinecone, і який був фінальний prompt, переданий у LLM.
**Ключові питання:**
*   Проблема в LLM (проігнорувала правильний контекст) чи в RAG (не витягнув оновлену політику клієнта з Pinecone)?

### 4. Mitigation
1.  **Kill-switch:** Негайно відключити функцію автогенерації для даного tenant-а (через Feature Flag), перевівши його в "Read-only" режим.
2.  **Global Fallback:** Якщо виявлено, що RAG-пайплайн глобально не синхронізує оновлені документи (Pinecone out-of-sync), відключити GenAI-компонент для всіх клієнтів до з'ясування причин. Approver: CPO + Head of Compliance.

### 5. Communication plan
*   **Internal:** Slack `#inc-...` updates кожні 15 хв.
*   **External (Clients):** Account Manager особисто зв'язується із зачепленим клієнтом, пояснює, що баг ізольовано.
*   **Compliance:** DPO аналізує, чи стався витік даних (якщо ні — повідомляти регулятора в рамках 72h GDPR не потрібно, оскільки це лише галюцинація, а не витік).

### 6. Recovery
Синхронізація RAG бази (реіндексація документів клієнта). Валідація через тестові промпти. Активація GenAI-функції.

### 7. Post-mortem checklist
- [ ] Post-mortem заплановано протягом 3 робочих днів.
- [ ] Root cause identified (напр., job синхронізації з Pinecone впав).
- [ ] Update Threat Model: T-002 (High-stakes Hallucination).

### 8. Preventive measures
*   Впровадити жорсткий alert: якщо job синхронізації корпоративних політик фейлиться — автоматично блокувати LLM-генерацію для цього клієнта.
*   Впровадити LLM-as-a-judge на етапі output, який перевіряє згенерований драфт на суперечності з policy-документами перед показом користувачу.

---

## Scenario 3: Deployment Failure (ArgoCD OutOfSync / Infrastructure)

*   **Severity:** P1
*   **Expected frequency:** 2-3 рази на рік
*   **MTTR target:** 15 хвилин до mitigation

### 1. Detection
**Тригери:**
*   Grafana/Prometheus alert: `ArgoCD_App_OutOfSync_Production`.
*   Alert: `EKS_Node_GPU_Capacity_Reached` під час спроби Canary релізу LLM.

### 2. Initial response (first 15 minutes)
- [ ] 0-2 хв: DevOps on-call бачить, що новий реліз не може заскейлити поди.
- [ ] 2-5 хв: Створити incident channel `#inc-YYYY-MM-DD-deploy-fail`.
- [ ] 5-10 хв: Запросити: MLOps Lead, VP Engineering.

### 3. Diagnosis
**Дашборди для перевірки:**
*   ArgoCD UI: Перегляд статусів (Missing, Degraded).
*   AWS EC2 Console: Перевірка лімітів квот на інстанси `p4d.24xlarge` (GPU) у нашому регіоні.

### 4. Mitigation
1.  **Abort Release:** Натиснути `Abort` для Canary Rollout в ArgoCD UI або через CLI, зупинивши переведення трафіку (залишити 100% на старій стабільній версії).
2.  **Git Revert:** Якщо ArgoCD перебуває у стані Degraded, зробити `git revert` останнього коміту в інфраструктурному репозиторії, щоб повернути бажаний стан (Desired State) до попереднього.

### 5. Communication plan
*   **Internal:** Повідомити VP Engineering.
*   **External:** Не впливає на клієнтів (zero downtime завдяки Canary та Abort), зовнішня комунікація не потрібна.

### 6. Recovery
Кластер повертається у стан `Synced` та `Healthy` в ArgoCD. Трафік 100% обслуговується старою моделлю.

### 7. Post-mortem checklist
- [ ] Root cause identified.
- [ ] Update Release Checklist: Додати пункт про обов'язкову перевірку AWS GPU Limits перед релізом важких моделей.

### 8. Preventive measures
*   Розширити AWS Service Quotas для GPU інстансів.
*   Впровадити multi-AZ (Availability Zone) або cross-region fallback для GPU-нод.