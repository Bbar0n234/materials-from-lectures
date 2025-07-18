---
layout: notes
title: "Алгоритмы ANN"
lecture: "vector-databases-ann"
order: 2
description: "Алгоритмы приближенного поиска ближайших соседей: HNSW, IVF, FAISS, Annoy и критерии выбора"
estimated_reading: "20 мин"
---

# Алгоритмы ANN

## Алгоритм HNSW (Hierarchical Navigable Small World)

### 1. Назначение и основная идея
* **Что это?** HNSW — это алгоритм для **аппроксимации поиска ближайших соседей (Approximate Nearest Neighbor, ANN)**.
* **Зачем нужен в RAG?** В RAG-системах он необходим, чтобы не перебирать все доступные документы (эмбеддинги) для поиска релевантного контекста. Прямой перебор имеет **линейную сложность `O(N)`**, что очень медленно для больших баз знаний.
* **Преимущество:** HNSW предлагает решение с **приближенно логарифмической сложностью `O(log N)`**, при этом обеспечивая высокую **полноту (recall)**, то есть находит действительно близких соседей.

### 2. Структура: Иерархический граф
* **Основа:** Алгоритм строит графовую структуру, состоящую из нескольких иерархических слоев.
* **Нижний слой (Layer 0):** Самый плотный, содержит **все** эмбеддинги из базы знаний.
* **Верхние слои:** Прогрессивно более **разреженные**. Чем выше слой, тем меньше на нем узлов (эмбеддингов), но связи между ними более "дальние", как скоростные шоссе.

### 3. Построение графа
1. **Построение иерархии:**
* При добавлении нового эмбеддинга мы сначала помещаем его на самый нижний слой (Layer 0).
* Далее для каждого следующего слоя мы "подбрасываем монетку": с определенной вероятностью эмбеддинг попадает на слой выше. Если "монетка" выпадает неудачно, то на этот и все последующие слои эмбеддинг уже не добавляется.
2. **Установка связей:**
* На каждом слое, где присутствует узел, для него строятся прямые связи (ребра) с его ближайшими соседями на **этом же слое**.
* Количество связей для каждого узла ограничено гиперпараметром `M`.

### 4. Процесс поиска ближайшего соседа
Поиск — это итеративный спуск по иерархии.
1. **Старт:** Начинаем поиск на самом верхнем, разреженном слое. Находим узел, ближайший к нашему входному запросу. Этот узел становится точкой входа для следующего уровня.
2. **Спуск и жадный поиск:**
* Опускаемся на слой ниже, используя найденный узел как отправную точку.
* Запускаем **жадный алгоритм**:
* Оцениваем всех соседей, с которыми у текущего узла есть прямые рёбра.
* Если находится сосед с **минимальным расстоянием** до запроса, которое меньше, чем у текущего узла, мы **переходим к нему**. Это становится новой отправной точкой, и итерация поиска повторяется.
* Если же ни один из прямых соседей не оказался ближе к цели, значит, мы **нашли локальный минимум**. Поиск на этом слое завершен.
3. **Финал:** Процесс повторяется, пока мы не дойдем до самого нижнего слоя (Layer 0). Узел, найденный на последнем шаге, и будет результатом приблизительного поиска ближайшего соседа.

### 5. Добавление нового эмбеддинга в граф
1. **Определение уровня:** Сначала для нового эмбеддинга случайным образом определяется максимальный слой, на котором он будет присутствовать ("подбрасывание монетки").
2. **Поиск места:** Начиная с верхнего слоя, алгоритм ищет наиболее подходящее "место" для вставки, спускаясь вниз и находя на каждом уровне ближайших кандидатов в соседи.
3. **Установка и оптимизация связей:**
* Новый узел связывается с `M` ближайшими соседями на каждом из своих уровней.
* **Важная оптимизация:** Если у одного из этих соседей уже есть максимальное количество связей, он может **"отрубить" самую дальнюю связь**, чтобы установить новую, более близкую, с нашим добавляемым эмбеддингом. Это позволяет графу постоянно самосовершенствоваться.

---

## Алгоритм IVF и Product Quantization

### Алгоритм IVF (Inverted File Index)

**1. Основная цель:**
*   Это алгоритм для **аппроксимации поиска ближайших соседей** в RAG-системах.
*   Главная задача — избежать медленного полного перебора всех эмбеддингов (`brute-force`).

**2. Принцип работы: Построение индекса и поиск**
*   **Шаг 1: Индексация (заранее)**
    *   Мы изначально берём **весь наш набор данных** (все эмбеддинги) и кластеризуем его, как правило, с помощью алгоритма **k-средних (k-means)**.
    *   В алгоритм подается параметр `k` — желаемое число кластеров. В результате мы получаем `k` **центроидов**, которые являются "центрами" для каждой группы векторов.
*   **Шаг 2: Поиск (при поступлении запроса)**
    *   Когда мы хотим найти ближайшего соседа для нашего входного запроса, мы сначала считаем расстояние (например, косинусное) между **вектором запроса и всеми центроидами**.
    *   Затем для нескольких (`nprobe`) ближайших к запросу центроидов мы **перебираем уже все эмбеддинги, принадлежащие только этим кластерам**.

**3. Баланс "Скорость vs. Качество"**
*   Количество проверяемых ближайших кластеров (`nprobe`) — это настраиваемый параметр.
    *   **Уменьшаем параметр:** Берём меньше кластеров. Поиск работает **быстрее**, но качество (точность) может быть **ниже**.
    *   **Повышаем параметр:** Берём больше кластеров. Поиск становится **дольше**, но и качество **лучше**, так как мы проверяем больше потенциальных кандидатов.

### Оптимизация: Product Quantization (PQ)

**1. Как работает кодирование вектора:**
*   **Разделение:** Каждый эмбеддинг в наборе данных делится на фиксированное количество **равных отрезков** (подвекторов), например, на 8.
*   **Отдельная кластеризация:** Для каждого из этих отрезков (точнее, для пространства, которое они занимают) проводится **своя** кластеризация.
*   **Кодирование:** Вектор кодируется таким образом, что вместо его фактического отрезка мы записываем **просто индекс ближайшего центроида** из соответствующего "словаря".
    *   *Практическая деталь:* Обычно для кластеризации отрезков берут `k=256` центроидов. Это позволяет закодировать индекс каждого центроида **одним байтом**. Таким образом, если вектор был разделен на 8 частей, его сжатая версия будет занимать всего **8 байт**.

**2. Как работает поиск с PQ:**
*   **Шаг 1: Подготовка запроса**
    *   Входной поисковый запрос **также делится на 8 частей**. Сам запрос остается несжатым (в float).
    *   Для **каждой части** запроса мы вычисляем расстояние до **каждого из 256 центроидов** в соответствующем словаре.
    *   В результате мы получаем **8 табличек**, где для каждого индекса центроида записано соответствующее расстояние.
*   **Шаг 2: Оценка расстояния до векторов в базе**
    *   Чтобы найти расстояние до любого сжатого вектора из нашей базы данных, мы делаем следующее:
        1.  Берём его сжатый код (например, 8 байт, где каждый байт — это индекс центроида).
        2.  Для **каждого индекса** в коде мы смотрим в **соответствующую ему табличку** и берем оттуда заранее вычисленное расстояние.
        3.  Все эти 8 найденных расстояний **складываем**.
*   **Результат:**
    *   **Чем меньше итоговая сумма расстояний, тем ближе входной запрос к соответствующему эмбеддингу из набора данных.**

---

## FAISS — роль и позиционирование

### 1. Определение и назначение
*   **FAISS (Facebook AI Similarity Search)** — это высокопроизводительная библиотека с открытым исходным кодом, разработанная Meta AI.
*   **Основная функция:** Реализация алгоритмов для быстрого поиска по сходству (Similarity Search) и кластеризации в плотных векторах.
*   **Ключевая характеристика:** FAISS не является базой данных, а представляет собой **вычислительное ядро (computational core) / поисковый движок (search engine)**.

### 2. Технологическая основа
*   **Реализация:** Написана на C++ для максимальной производительности с официальными обертками (bindings) для Python.
*   **Поддерживаемые алгоритмы:**
    *   **Точный поиск (Exact Search):** `IndexFlatL2` (полный перебор, используется как эталон).
    *   **Приблизительный поиск ближайших соседей (Approximate Nearest Neighbor, ANN):**
        *   **IVF (Inverted File):** `IndexIVFFlat`, `IndexIVFPQ`. Метод основан на разделении пространства векторов на кластеры (ячейки Вороного) и поиске только внутри ближайших кластеров.
        *   **HNSW (Hierarchical Navigable Small World):** `IndexHNSW`. Метод основан на построении многослойного графа для быстрой навигации к ближайшим соседям.
        *   **Квантование (Quantization):** `Product Quantization (PQ)`. Метод сжатия векторов для уменьшения потребления памяти и ускорения вычисления расстояний за счет потери точности.
*   **Аппаратное ускорение:** Одна из ключевых особенностей FAISS — нативная поддержка вычислений на **GPU**, что позволяет достигать кратного прироста производительности в задачах индексации и поиска.

### 3. FAISS в сравнении с векторными базами данных
*   **FAISS — это библиотека, а не сервис.** Он не предоставляет:
    *   Сетевого API (REST/gRPC).
    *   Механизмов персистентного хранения данных и управления ими.
    *   CRUD-операций (удаление/обновление векторов затруднено для многих типов индексов).
    *   Фильтрации по метаданным.
    *   Встроенных инструментов для масштабирования, репликации и администрирования.
*   **Векторные БД — это полноценные системы.** Они используют поисковые движки (собственные или сторонние, как FAISS) и добавляют к ним всю вышеперечисленную функциональность, необходимую для промышленной эксплуатации.

### 4. Использование FAISS в индустрии
*   **Прямое использование:** Применяется в задачах, где требуется максимальный контроль над производительностью и ресурсами, в научных исследованиях и при быстром прототипировании локальных RAG-систем.
*   **Как компонент БД:**
    *   **Milvus:** Активно использует FAISS как один из основных поисковых движков в своем ядре **Knowhere**.
    *   **Большинство других БД (Qdrant, Weaviate, Chroma, Pinecone, pgvector):** **Не используют** FAISS. Они разрабатывают собственные реализации ANN-алгоритмов (преимущественно HNSW) на своих основных языках (Rust, Go, C++) для лучшей интеграции, контроля над производительностью, поддержки динамических данных и реализации специфических функций, таких как фильтруемый поиск.

---

## Выбор ANN-алгоритма для RAG-системы

### Введение: От "Карго-культа" к Инженерному Решению
Выбор ANN-алгоритма — это не выбор "лучшего" инструмента, а выбор набора **компромиссов**, которые оптимально подходят под вашу конкретную задачу. Простое следование тренду (например, "все используют HNSW") может привести к созданию неэффективной, дорогой или медленной системы.

### Два титана: HNSW vs. IVF. Краткая суть
*   **HNSW (Hierarchical Navigable Small World):** Представьте социальную сеть. Чтобы найти человека, вы не спрашиваете всех подряд. Вы спрашиваете своих друзей, они — своих, и так вы быстро добираетесь до нужного человека через "шесть рукопожатий". HNSW строит многоуровневый граф, где поиск — это быстрый спуск по "хайвеям" связей к нужной области. **Девиз: быстрый прицельный поиск.**
*   **IVF (Inverted File):** Представьте огромную библиотеку, разделенную на залы по темам ("Финансы", "История", "Фантастика"). Чтобы найти книгу, вы сначала идете в нужный зал (кластер), а уже потом ищете на его полках. IVF делит все векторы на кластеры и при поиске смотрит только в нескольких самых релевантных. **Девиз: разделяй и властвуй.**

### Ключевые Факторы для Принятия Решения

#### 1. Фильтрация: Иллюзия "Pre-filtering" в HNSW
*   **Настоящий Pre-filtering (подход IVF):**
    1.  **Сначала фильтр, потом поиск:** Система сначала отбирает всех кандидатов, подходящих под метаданные (например, `source = 'legal_docs'`).
    2.  **Поиск по подмножеству:** ANN-поиск запускается только по этому, уже отфильтрованному, небольшому набору данных.
    3.  **Результат:** Быстро и очень точно. Чем строже фильтр, тем быстрее поиск.
*   **Filtered Search (реальность HNSW):**
    1.  **Поиск и фильтр одновременно:** HNSW не может заранее отфильтровать данные, так как это сломает его навигационный граф. Вместо этого он начинает обход графа, и на каждом шаге проверяет узел: "Ты подходишь под фильтр?".
    2.  **Если узел не подходит:** Он отбрасывается, и алгоритм ищет обходной путь.
    3.  **Результат и компромиссы:**
        *   **Падение скорости:** Алгоритм "блуждает" по графу, отбрасывая 99% узлов. Чем строже фильтр, тем **медленнее** поиск.
        *   **Падение точности (Recall):** Самый релевантный для вас вектор может быть "спрятан" за узлом, который не проходит по фильтру. Алгоритм отбросит этот узел-мост и никогда не найдет вашу цель.

#### 2. Динамика Данных: Статический Архив vs. Живой Поток
*   **HNSW (для динамических данных):**
    *   **Добавление/удаление:** Очень эффективно. Добавление нового вектора — это локальная операция по установлению связей с ближайшими соседями. Удаление обычно происходит через пометку узла как "удаленного". Это не требует перестройки всего индекса.
*   **IVF (для статических данных):**
    *   **Добавление/удаление:** Добавлять векторы легко (просто положить в нужный кластер), но это постепенно "портит" индекс, делая центры кластеров неактуальными.
    *   **Переиндексация:** Чтобы поддерживать качество поиска, IVF-индекс необходимо **периодически полностью перестраивать** (запускать перекластеризацию). Это очень долгая и ресурсоемкая операция.

#### 3. Тип Нагрузки: Одиночные vs. Пакетные Запросы
*   **Одиночный запрос (Low Latency):** Цель — минимальное время ответа для одного пользователя.
    *   **HNSW — король этого режима.** Его навигация по графу идеально заточена под быстрый "спринт" к цели для одного запроса. Идеально для чат-ботов.
*   **Пакетный запрос (High Throughput):** Цель — обработать максимум запросов в секунду в офлайн-режиме.
    *   **IVF — чемпион этого режима.** Он может сгруппировать 1000 запросов, определить, какие кластеры им всем нужны, загрузить каждый нужный кластер в память **один раз** и "прогнать" по нему все релевантные запросы. Это гораздо эффективнее, чем 1000 раз бегать по графу HNSW. Идеально для аналитики.

#### 4. Масштабируемость: Монолитность vs. Шардирование
*   **HNSW — по своей природе монолитен.** Его граф сложно "разрезать" на несколько машин.
    *   **Решение (как в Qdrant):** Архитектурный трюк **"Scatter-Gather"**. Данные делятся на шарды по ID. На каждом шарде строится свой **независимый** HNSW-индекс. Поисковый запрос отправляется **параллельно на все шарды**, а результаты потом собираются и объединяются.
*   **IVF — нативно масштабируемый.**
    *   Его кластеры — это независимые "куски" данных. Их можно легко разложить по разным машинам (шардам). Поиск идет только на тех машинах, где лежат нужные кластеры.

### Итоговая Сводка и Рекомендации

| Критерий | HNSW (Qdrant, Weaviate) | IVF (Milvus, Faiss) |
| :--- | :--- | :--- |
| **Фильтрация** | ⚠️ **Filtered Search** (медленнее, ниже точность) | ✅ **True Pre-filtering** (быстро, точно) |
| **Динамика данных** | ✅ **Отлично** (быстрое добавление/удаление) | ⚠️ **Плохо** (требует дорогой переиндексации) |
| **Режим работы** | ✅ **Low Latency** (идеально для одиночных запросов) | ✅ **High Throughput** (идеально для пакетной обработки) |
| **Память (RAM)** | ⚠️ **Высокое** потребление | ✅ **Низкое** потребление (особенно с квантизацией) |
| **Масштабирование** | ⚠️ **Сложное** (монолит, требует "Scatter-Gather") | ✅ **Простое** (нативно шардируется по кластерам) |

**Когда выбирать HNSW:**
*   Вам нужна **минимальная задержка** для интерактивного RAG (чат-боты).
*   Данные **постоянно обновляются** (добавляются/удаляются).
*   Фильтрация не является основной или очень "жесткой" функцией.
*   Вы готовы платить за большое количество RAM или используете БД с продвинутой квантизацией (как Qdrant).

**Когда выбирать IVF:**
*   Ваша RAG-система работает со **статическими или редко обновляемыми** данными.
*   **Эффективная фильтрация** — критически важная часть вашей системы.
*   Основная нагрузка — **офлайн-обработка больших пакетов** запросов.
*   Вы строите **масштабную распределенную систему** и хотите минимизировать архитектурную сложность.
*   **Бюджет на RAM ограничен.**

---

## Алгоритм Annoy

### 1. Общая концепция и назначение
**Annoy (Approximate Nearest Neighbors Oh Yeah)** — это алгоритм для эффективного **приблизительного поиска ближайших соседей (ANN)**, широко применяемый в RAG-системах и рекомендательных сервисах. Его главная цель — радикально ускорить поиск в многомерном пространстве ценой небольшой и контролируемой потери точности.

### 2. Процесс построения индекса: Лес случайных деревьев
*   **Статичность данных:** Фундаментальное требование Annoy — **статичный и неизменяемый набор данных**. В отличие от HNSW, где можно добавлять элементы "на лету", или IVF, где можно добавлять в существующие кластеры, Annoy требует **полной перестройки индекса** при любом изменении датасета.
*   **Рекурсивное бинарное разбиение:** Основа индексации — итеративное разделение пространства. На каждом шаге для текущего подмножества векторов:
    1.  Строится **случайная гиперплоскость**. Она определяется двумя случайно выбранными точками из подмножества.
    2.  Все векторы в подмножестве распределяются на две группы ("левую" и "правую") в зависимости от их положения относительно этой гиперплоскости.
*   **Условие остановки:** Этот рекурсивный процесс продолжается до тех пор, пока в каждом конечном подпространстве (листе дерева) количество векторов не станет меньше или равно заранее заданному параметру `k`.
*   **Построение Леса:** Процесс повторяется `T` (`n_trees`) раз на одном и том же исходном наборе данных. Поскольку гиперплоскости строятся случайно, каждое из `T` деревьев получается уникальным. Использование леса повышает вероятность нахождения истинных соседей, так как неудачное разбиение в одном дереве может быть скомпенсировано удачным в другом.

### 3. Структура дерева: Узлы и Листья
Дерево состоит из двух типов элементов:
*   **Узел разделения (Internal Node):** Это "навигационный переключатель". Он **не хранит** никаких векторов данных. Его содержимое — это математическое описание гиперплоскости:
    *   **Нормальный вектор (`n`):** Определяет ориентацию плоскости.
    *   **Смещение (`d`):** Определяет её положение в пространстве.
    Его единственная функция — принять вектор `q` и направить его к левому или правому потомку на основе знака скалярного произведения `n · q - d`.
*   **Листовой узел (Leaf Node):** Это терминальная точка любой ветви. Он хранит список идентификаторов (ID) векторов, попавших в данное конечное подпространство.

### 4. Процесс поиска (Inference): Навигация с приоритетной очередью
Поиск ближайших соседей для вектора запроса `q` — это не "жадный" спуск, а умное исследование с помощью **приоритетной очереди**:
*   **Механизм очереди:** В очередь добавляются узлы для будущего исследования. Приоритет узла — это "цена" пути к нему. **Чем меньше числовое значение приоритета, тем выше приоритет.**
*   **Исследование ветвей:** Когда алгоритм находится в узле разделения, он:
    1.  Определяет "ближнего" потомка (в чьё полупространство попал `q`) и "дальнего".
    2.  Добавляет "ближнего" потомка в очередь, сохраняя текущий приоритет пути.
    3.  Добавляет "дальнего" потомка, но с **увеличенным приоритетом**. Новый приоритет равен `текущий_приоритет + |margin|`, где `|margin|` — это расстояние от `q` до разделяющей гиперплоскости. Это "штраф" за выбор менее очевидного пути.
*   **Динамический путь:** Алгоритм всегда извлекает из очереди узел с наивысшим приоритетом. Это позволяет ему динамически переключаться между разными ветвями и даже разными деревьями, всегда исследуя самый перспективный на данный момент путь.
*   **Завершение поиска:**
    1.  Когда спуск доходит до **листового узла**, все ID из него добавляются в список кандидатов.
    2.  Процесс останавливается, когда общее число рассмотренных кандидатов превышает бюджет `search_k`.
    3.  Из итогового списка кандидатов выбираются топ-N ближайших с помощью точного расчета расстояния.

### Сравнение с HNSW и IVF

| Характеристика | Annoy | HNSW | IVF |
| :--- | :--- | :--- | :--- |
| **Аналогия** | Лес случайных карт | Система скоростных шоссе и городских улиц | Библиотечный каталог или индекс книги |
| **Основная идея** | Рекурсивное бинарное разбиение пространства. | Построение многослойного графа для быстрой навигации. | Кластеризация пространства и поиск только в релевантных кластерах. |
| **Динамичность** | **Нет.** Требует полной перестройки. | **Да.** Легко добавляет новые элементы в граф. | **Частично.** Можно добавлять элементы, но для оптимальной работы нужна периодическая перетренировка кластеров. |

### Асимптотическая сложность поиска
*   **Annoy:** **`O(polylog(N))`** — сложность определяется спуском по `T` деревьям, каждое из которых имеет глубину `O(log N)`. Итоговая сложность поиска — `O(T * log N)` плюс накладные расходы на очередь.
*   **HNSW:** **`O(log N)`** — теоретически самый эффективный из трех. Поиск начинается на самом разреженном верхнем слое графа, что позволяет за логарифмическое число шагов найти нужную область пространства.
*   **IVF:** **`O(K + P * (N/K))`** где `K` — число кластеров, `P` — число исследуемых кластеров. При оптимальном выборе `K ≈ sqrt(N)`, общая сложность стремится к **`O(sqrt(N))`**. 