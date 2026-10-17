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