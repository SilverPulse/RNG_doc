# Диаграммы модуля RNG

> Ответственный: Шаимов Богдан | Проверено по коду: RNG_brenсh2d2f164

## 1. Текущая структура (AS-IS)

Кратко: показано внутреннее устройство модуля `RNG.py` (v1.2.0) в ветке `RNG_brench`. Фасад `RandomGenerator` управляет созданием потоков и в зависимости от параметра `engine` делегирует генерацию одному из трёх классов: `RngStream` (`MRG32k3a`), `PhiloxStream` (`Philox4x64`) или `DataStream` (чтение `.json`/`.csv`).

### 1.1. Схема компонентов и потоков данных модуля (AS-IS)

```mermaid
flowchart LR
    subgraph Входы["Входные параметры"]
        PARAMS["Параметры инициализации\n(seed, base_stream_idx, run_idx, engine)"]
        FILES[("Файлы датасетов\n.json / .csv")]
    end

    subgraph RNG_MOD["Модуль RNG (RNG.py)"]
        RG["RandomGenerator\n(фасад распределений,\nсловарь потоков _streams)"]
        MRG["RngStream\n(ядро MRG32k3a)"]
        PHI["PhiloxStream\n(ядро Philox4x64)"]
        DS["DataStream\n(буфер deque из файла)"]

        RG -->|"engine == 'MRG32k3a'\nrand_u01()"| MRG
        RG -->|"engine == 'Philox'\nrand_u01()"| PHI
        RG -->|"engine == 'File'\nget_next_value()"| DS
    end

    subgraph Выходы["Выходные данные"]
        OUT["Случайные величины / данные из файла\nuniform(), normal(), exponential(), get_integer()"]
    end

    PARAMS -->|"__init__()"| RG
    FILES -->|"_load_data()"| DS
    RG --> OUT
```

### 1.2. UML-диаграмма классов модуля RNG.py (AS-IS)

```mermaid
classDiagram
    class RandomGenerator {
        +int seed
        +int base_stream_idx
        +int run_idx
        +str engine
        +str filepath
        -dict _streams
        -dict _normal_cache
        +__init__(seed, base_stream_idx, run_idx, engine, filepath)
        -_get_stream(stream_offset) RngStream|PhiloxStream|DataStream
        +uniform(min_val, max_val, stream_offset) float
        +normal(mean, variance, bound, stream_offset) float
        +exponential(mean, bound, stream_offset) float
        +get_integer(min_val, max_val, stream_offset) int
    }

    class RngStream {
        +float m1$
        +float m2$
        +float norm$
        -bool _cache_built$
        -list _cache_A1$
        -list _cache_A2$
        +int seed
        +int stream_idx
        +int run_idx
        +list~float~ state
        +__init__(seed, stream_idx, run_idx)
        +_build_cache()$ void
        +rand_u01() float
        -_mult_mod_m(a, s, c, m)$ float
        -_mat_vec_mod_m(A, s, m)$ list~float~
        -_mat_mat_mod_m(A, B, m)$ list~list~float~~
        -_power_of_two_matrix(n)$ tuple
        -_advance_nth_by(nth, by, state) void
    }

    class PhiloxStream {
        +int PHILOX_M4x64_0$
        +int PHILOX_M4x64_1$
        +int PHILOX_W64_0$
        +int PHILOX_W64_1$
        +int seed
        +int stream_idx
        +int run_idx
        +list~int~ key
        +list~int~ counter
        -list~int~ _buffer
        +__init__(seed, stream_idx, run_idx)
        -_mulhilo(a, b) tuple~int, int~
        -_philox_round() void
        +rand_u01() float
    }

    class DataStream {
        +str filepath
        +bool loop
        -deque _buffer
        -list _initial_data
        +__init__(filepath, loop)
        -_load_data() void
        -_load_from_json() void
        -_load_from_csv() void
        +get_next_value() float
    }

    RandomGenerator "1" *-- "0..*" RngStream : создаёт в _streams
    RandomGenerator "1" *-- "0..*" PhiloxStream : создаёт в _streams
    RandomGenerator "1" *-- "0..*" DataStream : создаёт в _streams
```

## 2. Типовой сценарий (AS-IS)

Сценарий: запрос случайной величины через `RandomGenerator.normal()` при двух последовательных обращениях к одному `stream_offset` (демонстрация ленивого создания потока и работы кэша `_normal_cache`).

```mermaid
sequenceDiagram
    autonumber
    participant Caller as Вызывающий код
    participant RG as RandomGenerator
    participant Stream as RngStream / PhiloxStream

    Note over Caller,RG: Вызов №1 (поток для stream_offset=1 ещё не создан)
    Caller->>RG: normal(mean=0.0, variance=1.0, bound=3.0, stream_offset=1)
    RG->>RG: _get_stream(stream_offset=1)
    alt Поток отсутствует в _streams
        RG->>Stream: __init__(seed, base_stream_idx + 1, run_idx)
        Stream-->>RG: экземпляр потока
    end
    loop Пока не выполнено 0 < w <= 1.0 и |x - mean| <= bound
        RG->>Stream: rand_u01()
        Stream-->>RG: u1
        RG->>Stream: rand_u01()
        Stream-->>RG: u2
        RG->>RG: Расчёт (x1, x2) по методу Бокса — Мюллера
    end
    RG->>RG: Сохранить (y, v2) в _normal_cache[1]
    RG-->>Caller: Возврат x1 (float)

    Note over Caller,RG: Вызов №2 (используется кэш _normal_cache[1])
    Caller->>RG: normal(mean=0.0, variance=1.0, bound=3.0, stream_offset=1)
    RG->>RG: _get_stream(stream_offset=1)
    RG->>RG: Извлечь (y, v2) из _normal_cache.pop(1) и вычислить x2
    alt |x2 - mean| <= bound
        RG-->>Caller: Возврат x2 (float)
    else |x2 - mean| > bound
        RG->>Stream: Генерация новой пары через rand_u01()
        RG-->>Caller: Возврат нового значения (float)
    end
```

## 3. Целевая структура (TO-BE)

> Не согласовано, см. `docs/features.md` и `docs/spec_FEAT-RNG-001.md`.
> Внутри самого модуля `RNG.py` архитектура классов (`RandomGenerator`, `RngStream`, `PhiloxStream`, `DataStream`) сохраняется. Предлагаемая схема фабричной инициализации `RandomGenerator` в словаре `rng_instances` на стороне `SimulationManager` показана ниже для согласования с владельцем модуля `SIMULATION_MANAGER`.

```mermaid
flowchart TB
    START(["SimulationManager.start_simulation()"]) --> CHECK{"Проверка доменов:\nUE_MODULE, TRAFFIC_MODEL,\nMOBILITY_MODEL, CHANNEL_MODEL"}
    CHECK --> COND{"Передан dataset_path\nв SimulationConfig?"}
    COND -->|"Да"| FILE_RNG["RandomGenerator(\nengine='File', filepath=dataset_path)"]
    COND -->|"Нет"| MATH_RNG["RandomGenerator(\nengine=sim_config.rng_engine)"]
    FILE_RNG --> SAVE["Сохранение в словарь\nself.rng_instances[module_name]"]
    MATH_RNG --> SAVE
    SAVE --> TTI(["Использование в цикле run_tti()"])
```

## 4. Ссылки на исходники

- `RNG.py::RngStream` — математическое ядро `MRG32k3a` (`_build_cache`, `_advance_nth_by`, `rand_u01`).
- `RNG.py::PhiloxStream` — математическое ядро `Philox4x64` (`_philox_round`, `_mulhilo`, `rand_u01`).
- `RNG.py::DataStream` — чтение датасетов из `.json` и `.csv` (`_load_data`, `_load_from_json`, `_load_from_csv`, `get_next_value`).
- `RNG.py::RandomGenerator` — точка входа для работы с распределениями и потоками (`_get_stream`, `uniform`, `normal`, `exponential`, `get_integer`).
