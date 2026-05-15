# Iris ML API — Лабораторна робота №2

![CI](https://github.com/69MadLad69/mlops-lab2/actions/workflows/ci.yml/badge.svg)

## Опис проекту

Навчальний MLOps-проект, що реалізує наскрізний конвеєр: від тренування ML-моделі до її розгортання як публічного REST API.

Модель — логістична регресія, навчена на класичному датасеті **Iris** (4 ознаки квітки → 3 класи: *setosa*, *versicolor*, *virginica*). Сервіс приймає JSON з параметрами квітки та повертає передбачений клас разом із ймовірністю.

Проект демонструє ключові практики MLOps:

- автоматизоване тренування моделі (`ml/train.py`);
- REST API для real-time інференсу (`app/main.py`);
- unit-тести коду моделі та ендпоінтів (`tests/`);
- контейнеризація через Docker;
- безперервна інтеграція через GitHub Actions;
- хмарне розгортання на Render.

## Стек технологій

| Категорія         | Технологія                |
|-------------------|---------------------------|
| Мова              | Python 3.11               |
| ML                | scikit-learn, joblib      |
| Веб-фреймворк     | FastAPI + Uvicorn         |
| Валідація         | Pydantic v2               |
| Тестування        | pytest, httpx (TestClient)|
| Контейнеризація   | Docker                    |
| CI/CD             | GitHub Actions            |
| Хостинг           | Render (Docker)           |

## Структура репозиторію

```
ml-api-lab2/
├── .github/
│   └── workflows/
│       └── ci.yml           # GitHub Actions workflow
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI застосунок
│   └── schemas.py           # Pydantic-моделі вхід/вихід
├── ml/
│   ├── __init__.py
│   └── train.py             # Скрипт тренування
├── tests/
│   ├── __init__.py
│   ├── test_api.py          # Тести API
│   └── test_model.py        # Тести моделі
├── requirements.txt         # Залежності Python
├── Dockerfile               # Інструкції для Docker-образу
├── .dockerignore
├── .gitignore
└── README.md
```

Файл `model.joblib` генерується автоматично при запуску `python -m ml.train` і знаходиться у `.gitignore` — у репозиторії його немає.

## Як запустити локально

### 1. Клонування та віртуальне оточення

```bash
git clone https://github.com/69MadLad69/mlops-lab2.git
cd mlops-lab2

python -m venv .venv #py -3.11 -m venv .venv
# Linux / macOS:
source .venv/bin/activate
# Windows:
# .venv\Scripts\activate
```

### 2. Встановлення залежностей

```bash
python.exe -m pip install --upgrade pip
pip install -r requirements.txt
```

### 3. Тренування моделі

```bash
python -m ml.train
```

В результаті у корені проекту з'явиться файл `model.joblib`, а в консолі буде виведено accuracy на тестовій вибірці (очікувано ~0.97).

### 4. Запуск API

```bash
uvicorn app.main:app --reload
```

Сервіс буде доступний за адресою <http://localhost:8000>:

**Швидке посилання на Swagger UI:**<http://localhost:8000/docs>

- `GET /` — статус сервісу;
- `GET /health` — health check;
- `POST /predict` — інференс;
- `GET /docs` — інтерактивна документація Swagger UI.

## Запуск через Docker

### Збірка образу

```bash
docker build -t ml-api:lab2 .
```

Тренування моделі виконується **під час збірки образу**, тож запускати `ml/train.py` окремо не потрібно — артефакт `model.joblib` вже всередині образу.

### Запуск контейнера

```bash
docker run --rm -p 8000:8000 ml-api:lab2
```

Перевірка:

```bash
curl http://localhost:8000/health
# {"status":"healthy","model_loaded":true}
```

## Як запустити тести

```bash
pytest -q
```

Очікуваний результат — усі тести зелені:

```
......                                                            [100%]
6 passed in X.XXs
```

Тести покривають:

- `test_model.py` — створення файлу моделі, accuracy > 0.8, валідність класів передбачення;
- `test_api.py` — статус-ендпоінти, коректний відгук `/predict` для типового зразка setosa, обробка валідаційних помилок (HTTP 422).

## Як працює API

### `POST /predict`

**Запит:**

```json
{
  "sepal_length": 5.1,
  "sepal_width": 3.5,
  "petal_length": 1.4,
  "petal_width": 0.2
}
```

**Відповідь:**

```json
{
  "class_id": 0,
  "class_name": "setosa",
  "probability": 0.9712
}
```

### Приклад через `curl`

```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"sepal_length": 5.1, "sepal_width": 3.5, "petal_length": 1.4, "petal_width": 0.2}'
```

### Помилки валідації

Якщо передати некоректний тип або значення поза межами `[0, 10]`, FastAPI поверне `422 Unprocessable Entity` з описом проблеми. Перевірки описані у `app/schemas.py` через Pydantic-поля з `Field(..., ge=0, le=10)`.

## CI/CD

При кожному `push` або `pull_request` до `main` запускається workflow `.github/workflows/ci.yml`, що:

1. встановлює Python 3.11 та кешує pip-залежності;
2. ставить пакети з `requirements.txt`;
3. тренує модель командою `python -m ml.train`;
4. запускає `pytest -q`;
5. збирає Docker-образ (job `docker-build`, залежить від успішних тестів).

Статус останнього білду відображається у badge на початку README.

## Посилання на деплой

**Публічний сервіс на Render:** <https://mlops-lab2-ed3g.onrender.com>

Швидка перевірка живого сервісу:

```bash
curl https://mlops-lab2-ed3g.onrender.com/health

curl -X POST https://mlops-lab2-ed3g.onrender.com/predict \
  -H "Content-Type: application/json" \
  -d '{"sepal_length": 6.3, "sepal_width": 2.9, "petal_length": 5.6, "petal_width": 1.8}'
```

**Швидке посилання на Swagger UI:**<https://mlops-lab2-ed3g.onrender.com/docs>

## Автор
Студент групи ТР-51мп Авраменко Олег


Виконано в рамках курсу MLOps, лабораторна робота №2 «CI/CD та ML API».
