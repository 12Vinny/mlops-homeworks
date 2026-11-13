# Title: Контейнеризація Image Classification моделі (Fat та Slim Docker образи)

## Background
Команда успішно підготувала pretrained-модель класифікації зображень з torchvision.models (артефакт model/model.pt) та скрипт app/inference.py. Наступний крок — запакувати це рішення у Docker для подальшої передачі DevOps-команді та розгортання. Нам потрібен базовий образ для перевірки та оптимізований образ для подальшого використання у staging/production.

## Scope
* Створення базового Docker-образу (ml-infer-fat:1.0) на базі python:3.13.
* Створення оптимізованого Docker-образу (ml-infer-slim:1.0) на базі python:3.13-slim із використанням multi-stage build.
* Створення та налаштування файлу .dockerignore для виключення зайвих файлів з build context.
* Фіксація всіх необхідних залежностей у requirements.txt.
* Проведення перевірки: забезпечення однакового результату inference для обох образів.
* Створення README.md із покроковими командами для збірки образів та запуску inference.
* Підготовка короткого звіту report.md із порівнянням розміру та часу роботи обох образів.

## Out of Scope
* Model retraining (додаткове навчання чи донавчання моделі).
* Покращення accuracy або інших ML-метрик моделі.
* Налаштування production Kubernetes кластера чи маніфестів.
* Налаштування full monitoring (Prometheus, Grafana тощо) для моделі.

## Expected Deliverables
1. Dockerfile.fat
2. Dockerfile.slim
3. .dockerignore
4. requirements.txt (із зафіксованими версіями)
5. README.md (інструкція)
6. report.md (порівняння образів)
7. Вихідний код та артефакт моделі (app/inference.py, model/model.pt).

## Technical Constraints
* Python Version: Строго 3.13 для обох образів.
* Base Images: python:3.13 (для fat), python:3.13-slim (для slim).
* Slim Build: Обов'язкове використання підходу multi-stage build для slim-образу.
* Inference Output: Скрипт має повертати top-3 prediction.

## Acceptance Criteria Summary
Завдання буде прийнято після успішної збірки обох образів без помилок, їх коректного запуску через docker run та отримання ідентичних top-3 результатів класифікації на тестовому зображенні. Детальний перелік критеріїв знаходиться у файлі acceptance_criteria.md.