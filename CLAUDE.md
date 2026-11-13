# Wordstat Semantic Analyzer

## Цель проекта
Пайплайн для анализа и очистки семантического ядра из Яндекс.Вордстат. Обрабатывает до 20К+ запросов, классифицирует на релевантные/нерелевантные для заданной ниши.

## Конфигурация

Перед запуском создай `config.yaml`:

```yaml
# Обязательные параметры
niche:
  name: "название ниши"
  description: "детальное описание ниши, целевой аудитории, продуктов/услуг"
  relevant_examples:
    - "пример релевантного запроса 1"
    - "пример релевантного запроса 2"
  irrelevant_examples:
    - "пример нерелевантного запроса 1"
    - "пример нерелевантного запроса 2"

# Опциональные параметры
input:
  file: "data/input.csv"
  query_column: "query"           # колонка с запросами
  frequency_column: "frequency"   # колонка с частотой (опционально)

processing:
  min_frequency: 10               # минимальная частота (0 = не фильтровать)
  min_words: 1                    # минимум слов в запросе
  batch_size: 300                 # размер батча для LLM
  
embeddings:
  provider: "local"               # "local" (rubert-tiny2) или "openai"
  openai_model: "text-embedding-3-small"  # если provider: openai

clustering:
  min_cluster_size: 10
  min_samples: 3

output:
  dir: "data/output"
```

## Архитектура

```
wordstat-analyzer/
├── CLAUDE.md
├── config.yaml
├── requirements.txt
├── src/
│   ├── __init__.py
│   ├── pipeline.py          # Главный оркестратор
│   ├── prefilter.py         # Этап 1: детерминированная очистка
│   ├── embeddings.py        # Этап 2: векторизация
│   ├── clustering.py        # Этап 3: кластеризация
│   ├── classifier.py        # Этап 4: LLM-классификация
│   ├── revision.py          # Этап 5: ревизия спорных
│   └── utils/
│       ├── __init__.py
│       ├── logging.py       # Трекинг всех решений
│       └── stopwords.py     # Русские стоп-слова + кастомные
├── data/
│   ├── input.csv            # Входной файл
│   └── output/
│       ├── relevant.csv     # Релевантные запросы
│       ├── irrelevant.csv   # Отфильтрованный мусор
│       ├── uncertain.csv    # Требуют ручной проверки
│       └── processing.log   # Лог всех решений
└── run.py                   # Точка входа
```

## Этапы пайплайна

### Этап 1: Prefilter (prefilter.py)
Детерминированная очистка без LLM:
- Удаление стоп-слов (порно, казино, "скачать бесплатно", etc.)
- Regex-фильтры (телефоны, email, мусорные паттерны)
- Дедупликация с нормализацией (ё→е, lowercase, лишние пробелы)
- Фильтр по минимальной частоте
- Фильтр по минимальной длине

Логировать: сколько отфильтровано на каждом подэтапе.

### Этап 2: Embeddings (embeddings.py)
Векторизация запросов:
- Local: `cointegrated/rubert-tiny2` через sentence-transformers
- OpenAI: `text-embedding-3-small` через API

Кэшировать embeddings в `data/cache/embeddings.npy` для повторных запусков.

### Этап 3: Clustering (clustering.py)
Группировка семантически близких:
- HDBSCAN для автоматического определения кластеров
- Выделение центроидов и топ-10 примеров для каждого кластера
- Пометка "шума" (outliers) — пойдут отдельным батчем

Сохранять: `data/cache/clusters.json` с mapping query → cluster_id

### Этап 4: Classifier (classifier.py)
LLM-классификация батчами:

```python
CLASSIFICATION_PROMPT = """
Ты классифицируешь поисковые запросы для ниши: {niche_name}

Описание ниши: {niche_description}

Примеры РЕЛЕВАНТНЫХ запросов для этой ниши:
{relevant_examples}

Примеры НЕРЕЛЕВАНТНЫХ запросов:
{irrelevant_examples}

---

Классифицируй каждый запрос:
- "relevant" — относится к нише, потенциальный клиент мог искать
- "irrelevant" — не относится к нише, мусор
- "uncertain" — невозможно определить без дополнительного контекста

Запросы для классификации:
{queries}

Ответ строго в JSON (без markdown):
[
  {{"query": "...", "category": "relevant|irrelevant|uncertain", "reason": "короткое пояснение"}}
]
"""
```

Параметры LLM:
- model: claude-sonnet-4-20250514 (оптимум цена/качество)
- temperature: 0.1
- max_tokens: 4096

Обрабатывать батчами по `batch_size` из конфига. Между батчами — пауза 1 сек (rate limits).

### Этап 5: Revision (revision.py)
Повторный прогон для категории "uncertain":
- Добавить больше контекста ниши
- Показать примеры из уже классифицированных relevant/irrelevant
- Если всё ещё uncertain — оставить для ручной проверки

## Логирование

Каждое решение логировать в SQLite (`data/processing.db`):

```sql
CREATE TABLE decisions (
    id INTEGER PRIMARY KEY,
    query TEXT,
    stage TEXT,              -- 'prefilter', 'classifier', 'revision'
    decision TEXT,           -- 'relevant', 'irrelevant', 'uncertain', 'filtered_stopword', etc.
    reason TEXT,
    cluster_id INTEGER,
    batch_id INTEGER,
    created_at TIMESTAMP
);

CREATE TABLE runs (
    id INTEGER PRIMARY KEY,
    config_hash TEXT,
    input_count INTEGER,
    output_relevant INTEGER,
    output_irrelevant INTEGER,
    output_uncertain INTEGER,
    started_at TIMESTAMP,
    finished_at TIMESTAMP
);
```

## CLI

```bash
# Полный прогон
python run.py --config config.yaml

# Только до кластеризации (без LLM, для проверки)
python run.py --config config.yaml --until clustering

# Продолжить с этапа классификации (если embeddings закэшированы)
python run.py --config config.yaml --from classifier

# Статистика по последнему прогону
python run.py --stats
```

## Зависимости (requirements.txt)

```
anthropic>=0.18.0
sentence-transformers>=2.2.0
hdbscan>=0.8.33
numpy>=1.24.0
pandas>=2.0.0
pyyaml>=6.0
tqdm>=4.65.0
```

## Важные принципы

1. **Fail fast** — если батч не распарсился, логировать и продолжить, не падать
2. **Idempotent** — повторный запуск не дублирует данные
3. **Resumable** — можно продолжить с любого этапа
4. **Observable** — всё логируется, любое решение можно отследить

## Пример запуска

```bash
# 1. Создать конфиг
cp config.example.yaml config.yaml
# отредактировать под свою нишу

# 2. Положить данные
cp ~/wordstat_export.csv data/input.csv

# 3. Запустить
python run.py --config config.yaml

# 4. Результаты в data/output/
```
