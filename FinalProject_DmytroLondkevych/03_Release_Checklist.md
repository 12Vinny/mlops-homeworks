# Release Checklist: LexiGuard AI (Hybrid AI Product)

Цей чекліст є обов'язковим для заповнення перед кожним розгортанням нової версії моделі (LLM або класичної) у production.

## 1. Model Quality Gates
- [ ] [Model Quality] Класична ML: F1-score для Intent Classification на holdout-датасеті не нижче 94% (production baseline: 93.5%).
    * Owner: ML Lead
    * Evidence: MLflow Evaluation Run link
    * Blocker if failed: Yes
- [ ] [Model Quality] LLM: Hallucination rate (вимірюваний через LLM-as-a-judge) < 1.5%.
    * Owner: ML Lead
    * Evidence: LLM Evals Report у Confluence
    * Blocker if failed: Yes
- [ ] [Model Quality] LLM: Refusal accuracy (безпечна відмова на зловмисні промпти) > 99%.
    * Owner: Head of AI Safety
    * Evidence: Red-teaming test results (GitLab CI artifact)
    * Blocker if failed: Yes
- [ ] [Model Quality] RAG: Retrieval Recall@5 > 95% для корпоративних політик.
    * Owner: ML Engineer
    * Evidence: MLflow Retrieval Metrics
    * Blocker if failed: Yes
- [ ] [Model Quality] Перевірка дрифту (Model Drift): Розподіл передбачень класичної моделі на тестових даних збігається з production-розподілом (P-value > 0.05).
    * Owner: MLOps Engineer
    * Evidence: Evidently AI Drift Report
    * Blocker if failed: No (потребує manual review)

## 2. Data Readiness
- [ ] [Data Readiness] PII sanitization підтверджена: тестові та тренувальні датасети очищені від реальних імен та фінансових даних клієнтів.
    * Owner: Data Engineer / DPO
    * Evidence: Data Sanitization Pipeline success log
    * Blocker if failed: Yes
- [ ] [Data Readiness] Data schema валідована для вхідних контрактів.
    * Owner: Data Engineer
    * Evidence: Great Expectations Data Docs
    * Blocker if failed: Yes
- [ ] [Data Readiness] Навчальні дані зафіксовані у DVC з відповідним тегом версії.
    * Owner: Data Engineer
    * Evidence: DVC commit hash
    * Blocker if failed: Yes
- [ ] [Data Readiness] Векторна база даних (Pinecone) оновлена актуальними політиками для всіх tenant-ів.
    * Owner: Backend Lead
    * Evidence: RAG Sync Job Success Log
    * Blocker if failed: Yes
- [ ] [Data Readiness] Розподіл train/prod даних порівняно, суттєвого data drift не виявлено.
    * Owner: MLOps Engineer
    * Evidence: MLflow Data Profiling
    * Blocker if failed: No

## 3. Infrastructure Readiness
- [ ] [Infra Readiness] K8s resources (GPU/CPU limits & requests) заплановані та затверджені для нового інференсу.
    * Owner: MLOps Lead
    * Evidence: Terraform Plan / K8s Manifest PR
    * Blocker if failed: Yes
- [ ] [Infra Readiness] HPA (Horizontal Pod Autoscaler) налаштовано для класичної ML-моделі (TorchServe).
    * Owner: DevOps Engineer
    * Evidence: HPA configuration yaml
    * Blocker if failed: Yes
- [ ] [Infra Readiness] Monitoring dashboards у Grafana оновлені для відстеження нових метрик (якщо додані нові фічі).
    * Owner: DevOps Engineer
    * Evidence: Grafana Dashboard URL
    * Blocker if failed: No
- [ ] [Infra Readiness] Prometheus alerts підключені до відповідного PagerDuty сервісу.
    * Owner: DevOps Engineer
    * Evidence: PagerDuty Service link
    * Blocker if failed: Yes
- [ ] [Infra Readiness] Навантажувальне тестування (Load Testing) пройдено: P99 latency < 2s для LLM та < 200ms для класичної ML при 100 RPS.
    * Owner: QA / MLOps
    * Evidence: K6 Load Test Report
    * Blocker if failed: Yes
- [ ] [Infra Readiness] Стратегія резервного копіювання (Backup) для Pinecone RAG валідована.
    * Owner: Backend Lead
    * Evidence: AWS Backup Job status
    * Blocker if failed: Yes

## 4. Security & Compliance
- [ ] [Security] Audit logs (AWS CloudTrail, MLflow Model Registry) увімкнені та зберігаються > 365 днів.
    * Owner: Head of Compliance
    * Evidence: AWS Config Rule compliance
    * Blocker if failed: Yes
- [ ] [Security] Threat Model перевірена: нові вектори атак (якщо є) внесені в реєстр.
    * Owner: AI PM
    * Evidence: Threat Model v.X.X
    * Blocker if failed: Yes
- [ ] [Security] RBAC (Role-Based Access Control) переглянуто: пайплайни мають Least Privilege доступ до EKS/S3.
    * Owner: DevOps Lead
    * Evidence: IAM Policy review sign-off
    * Blocker if failed: Yes
- [ ] [Security] Encryption at rest & in transit (TLS 1.3) активовано для всіх API та баз даних.
    * Owner: Security Engineer
    * Evidence: Security Scan Report
    * Blocker if failed: Yes
- [ ] [Security] EU AI Act документація (Model Card, Data Card) оновлена та збережена.
    * Owner: Head of Compliance / AI PM
    * Evidence: Confluence Model Card link
    * Blocker if failed: Yes
- [ ] [Security] Вхідні Guardrails (фільтри prompt injection) активні та протестовані.
    * Owner: Head of AI Safety
    * Evidence: Security Evals Report
    * Blocker if failed: Yes

## 5. Operational Readiness
- [ ] [Operational] Incident Runbook оновлено актуальними кроками діагностики для нової моделі.
    * Owner: AI PM / MLOps Lead
    * Evidence: Runbook Wiki Link
    * Blocker if failed: Yes
- [ ] [Operational] On-call ротація (Primary/Secondary) призначена на період Canary-релізу.
    * Owner: VP Engineering
    * Evidence: PagerDuty Schedule
    * Blocker if failed: Yes
- [ ] [Operational] Rollback-план протестовано на Staging-середовищі (через ArgoCD git revert).
    * Owner: MLOps Lead
    * Evidence: Staging Rollback Test Run
    * Blocker if failed: Yes
- [ ] [Operational] Communication template для інцидентів (внутрішній та зовнішній) готовий.
    * Owner: AI PM
    * Evidence: Canned Responses in Slack/Zendesk
    * Blocker if failed: No
- [ ] [Operational] Canary Deployment конфігурація (відсотки трафіку 5%->25%->100%) затверджена.
    * Owner: MLOps Lead
    * Evidence: ArgoCD Rollout YAML
    * Blocker if failed: Yes

## 6. Business Readiness
- [ ] [Business] Cost per prediction утримується в межах цільового бюджету (< $0.03/req).
    * Owner: AI PM / CFO
    * Evidence: Cost Estimation Dashboard
    * Blocker if failed: Yes
- [ ] [Business] Stakeholder sign-off: CPO погодив розгортання після review метрик.
    * Owner: CPO
    * Evidence: Approval у Jira тікеті
    * Blocker if failed: Yes
- [ ] [Business] Legal team approval: Юридичний відділ підтвердив відсутність ризиків у нових шаблонах відповідей LLM.
    * Owner: Head of Legal
    * Evidence: Confluence Approval
    * Blocker if failed: Yes
- [ ] [Business] Launch communication (Release Notes) відправлена клієнтам та Sales-команді.
    * Owner: Product Marketing Manager
    * Evidence: Email Campaign Draft
    * Blocker if failed: No
- [ ] [Business] Success metrics baseline зафіксовано для порівняння після 7 днів у production.
    * Owner: AI PM
    * Evidence: Product Analytics Dashboard
    * Blocker if failed: Yes