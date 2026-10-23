# Диаграммы модуля RNG

> Ответственный: Шаимов Богдан | Проверено по коду: RNG_brenсh2d2f164

## 1. Схема компонентов и потоков данных модуля (AS-IS)

Кратко: показано внутреннее устройство модуля `RNG.py` (v1.2.0) в ветке `RNG_brench`. Фасад `RandomGenerator` управляет созданием потоков и в зависимости от параметра `engine` делегирует генерацию одному из трёх классов: `RngStream` (`MRG32k3a`), `PhiloxStream` (`Philox4x64`) или `DataStream` (чтение `.json`/`.csv`).

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

## 3. Целевая структура и UML-диаграмма (TO-BE)

> Отражает план архитектурного рефакторинга по итогам ревью (замена ветвлений на паттерн «Стратегия» и внедрение зависимостей через `Protocol`).

В целевой архитектуре модели получают случайные величины через абстрактный контракт `IRandomSource` (Protocol). Режим чтения файлов (`DataStream`) вынесен из математического генератора. Оркестратор симулятора (`SimulationManager`) управляет резервированием индексов и выступает DI-контейнером.

```mermaid
classDiagram
    class IRandomSource {
        <<Protocol>>
        +uniform(min_val, max_val, stream_offset) float
        +normal(mean, variance, bound, stream_offset) float
        +exponential(mean, bound, stream_offset) float
        +get_integer(min_val, max_val, stream_offset) int
    }

    class SimulationManager {
        +start_simulation()
    }

    class AbstractModel {
        <<Mobility / Channel / Traffic>>
        -IRandomSource rng
        +__init__(rng: IRandomSource)
    }

    class RandomGenerator {
        +int seed
        +int base_stream_idx
        -dict _streams
        -dict _normal_cache
        +__init__(seed, base_stream_idx, run_idx, engine)
    }

    class RngStream {
        <<MRG32k3a Strategy>>
        +rand_u01() float
    }

    class PhiloxStream {
        <<Philox4x64 Strategy>>
        +rand_u01() float
    }

    class DataStream {
        <<Standalone Dataset Provider>>
        +get_next_value() float
    }

    AbstractModel --> IRandomSource : DI (Dependency Injection)
    SimulationManager ..> RandomGenerator : Создает (Политика индексов)
    SimulationManager ..> AbstractModel : Внедряет зависимость (rng)
    RandomGenerator ..|> IRandomSource : Реализует протокол
    DataStream ..|> IRandomSource : Реализует протокол
    RandomGenerator "1" *-- "0..*" RngStream : _streams
    RandomGenerator "1" *-- "0..*" PhiloxStream : _streams
```

## 4. Ссылки на исходники

- `RNG.py::RngStream` — математическое ядро `MRG32k3a` (`_build_cache`, `_advance_nth_by`, `rand_u01`).
- `RNG.py::PhiloxStream` — математическое ядро `Philox4x64` (`_philox_round`, `_mulhilo`, `rand_u01`).
- `RNG.py::DataStream` — чтение датасетов из `.json` и `.csv` (`_load_data`, `_load_from_json`, `_load_from_csv`, `get_next_value`).
- `RNG.py::RandomGenerator` — точка входа для работы с распределениями и потоками (`_get_stream`, `uniform`, `normal`, `exponential`, `get_integer`).
