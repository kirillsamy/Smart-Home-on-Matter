# 🏠 Smart Home on Matter

Курсовая работа по дисциплине «Программирование»  
Язык: **C** | Протокол: **Matter 1.3**

-----

## 📋 Описание

Консольная программа на языке C для сбора и обработки показаний датчиков системы умного дома, работающих по протоколу **Matter**.

Программа позволяет:

- Вводить показания датчиков вручную с клавиатуры
- Проверять значения на соответствие допустимым нормам
- Выводить сводный отчёт по всем комнатам
- Сохранять журнал событий в файл `log.txt`
- Вычислять среднюю температуру и количество тревог

-----

## 🌡️ Датчики

Каждый узел сети Matter собирает следующие параметры:

|Параметр   |Единица |Норма     |
|-----------|--------|----------|
|Температура|°C      |до 26 °C  |
|Влажность  |%       |до 70 %   |
|CO2        |ppm     |до 800 ppm|
|Движение   |да / нет|—         |

-----

## 📡 Протокол Matter

**Matter** — открытый стандарт для устройств умного дома от консорциума CSA (Apple, Google, Amazon, Samsung).

- Транспорт: **Thread** и **Wi-Fi** (стек IPv6)
- Адресация: уникальный **Node ID** для каждого устройства
- Безопасность: шифрование **AES-128**, сертификаты X.509

-----

## 🗂️ Структура проекта

```
Smart-Home-on-Matter/
├── smart_home.c   # исходный код программы
├── log.txt        # журнал событий (создаётся при запуске)
└── README.md      # описание проекта
```

-----

## ⚙️ Сборка и запуск

### Linux / macOS

```bash
gcc smart_home.c -o smart_home
./smart_home
```

### Windows

```bash
gcc smart_home.c -o smart_home.exe
smart_home.exe
```

-----

## 💻 Пример работы

```
================================================
   УМНЫЙ ДОМ — протокол Matter 1.3
   Ввод показаний датчиков
================================================

  Датчик #1
Название комнаты: Кухня
Температура (C):   27.5
Влажность (%):     75.0
Уровень CO2 (ppm): 950
Есть движение? (1=да, 0=нет): 1

================================================
   ОТЧЁТ ПО ВСЕМ ДАТЧИКАМ
================================================

  [Node 0x0001] Кухня
    Температура  : 27.5 C  <<< ВЫШЕ НОРМЫ (до 26 C)
    Влажность    : 75.0 %  <<< ВЫШЕ НОРМЫ (до 70 %)
    CO2          : 950 ppm <<< ВЫШЕ НОРМЫ (до 800 ppm)
    Движение     : ОБНАРУЖЕНО

================================================
  Всего датчиков      : 1
  Средняя температура : 27.5 C
  Количество тревог   : 3
  Данные сохранены в  : log.txt
================================================
```

-----
📄 Код программы
```
#include <stdio.h>
#include <string.h>
#include <time.h>

#define MAX_ROOMS    10
#define LOG_FILE     "log.txt"
#define TEMP_MAX     26.0f
#define HUMIDITY_MAX 70.0f
#define CO2_MAX      800

typedef struct {
    int   node_id;
    char  room[32];
    float temperature;
    float humidity;
    int   co2;
    int   motion;
} Sensor;

void get_time(char *buf, int size) {
    time_t now = time(NULL);
    strftime(buf, size, "%H:%M:%S", localtime(&now));
}

void separator(void) {
    printf("================================================\n");
}

void input_sensor(Sensor *s, int number) {
    int motion_input;
    s->node_id = number;
    separator();
    printf("  Sensor #%d\n", number);
    separator();
    printf("Room name:         ");
    scanf("%31s", s->room);
    printf("Temperature (C):   ");
    scanf("%f", &s->temperature);
    printf("Humidity (%%):      ");
    scanf("%f", &s->humidity);
    printf("CO2 (ppm):         ");
    scanf("%d", &s->co2);
    printf("Motion (1/0):      ");
    scanf("%d", &motion_input);
    s->motion = motion_input;
    printf("\n");
}

void print_sensor(const Sensor *s) {
    printf("  [Node 0x%04X] %s\n", s->node_id, s->room);
    printf("    Temperature  : %.1f C", s->temperature);
    if (s->temperature > TEMP_MAX) printf("  <<< ABOVE LIMIT");
    printf("\n");
    printf("    Humidity     : %.1f %%", s->humidity);
    if (s->humidity > HUMIDITY_MAX) printf("  <<< ABOVE LIMIT");
    printf("\n");
    printf("    CO2          : %d ppm", s->co2);
    if (s->co2 > CO2_MAX) printf("  <<< ABOVE LIMIT");
    printf("\n");
    printf("    Motion       : %s\n\n", s->motion ? "YES" : "no");
}

void save_to_file(FILE *f, const Sensor *s, const char *t) {
    fprintf(f, "[%s] %s (Node 0x%04X)\n", t, s->room, s->node_id);
    fprintf(f, "  Temperature : %.1f C%s\n", s->temperature,
            s->temperature > TEMP_MAX ? " [LIMIT EXCEEDED]" : "");
    fprintf(f, "  Humidity    : %.1f %%%s\n", s->humidity,
            s->humidity > HUMIDITY_MAX ? " [LIMIT EXCEEDED]" : "");
    fprintf(f, "  CO2         : %d ppm%s\n", s->co2,
            s->co2 > CO2_MAX ? " [LIMIT EXCEEDED]" : "");
    fprintf(f, "  Motion      : %s\n\n", s->motion ? "yes" : "no");
}

float average_temp(const Sensor sensors[], int count) {
    float sum = 0.0f;
    int i;
    for (i = 0; i < count; i++)
        sum += sensors[i].temperature;
    return sum / count;
}

int count_alerts(const Sensor sensors[], int count) {
    int alerts = 0, i;
    for (i = 0; i < count; i++) {
        if (sensors[i].temperature > TEMP_MAX)    alerts++;
        if (sensors[i].humidity    > HUMIDITY_MAX) alerts++;
        if (sensors[i].co2        > CO2_MAX)      alerts++;
    }
    return alerts;
}

int main(void) {
    Sensor sensors[MAX_ROOMS];
    int    count = 0;
    char   choice;
    char   time_str[32];
    int    i;

    FILE *log = fopen(LOG_FILE, "a");
    if (log == NULL) {
        printf("Error: cannot open file %s\n", LOG_FILE);
        return 1;
    }

    get_time(time_str, sizeof(time_str));
    fprintf(log, "=== New session: %s ===\n\n", time_str);

    separator();
    printf("   SMART HOME — Matter 1.3\n");
    separator();
    printf("\n");

    do {
        if (count >= MAX_ROOMS) {
            printf("Maximum sensors reached (%d).\n", MAX_ROOMS);
            break;
        }
        input_sensor(&sensors[count], count + 1);
        get_time(time_str, sizeof(time_str));
        save_to_file(log, &sensors[count], time_str);
        count++;
        printf("Add another? (y/n): ");
        scanf(" %c", &choice);
        printf("\n");
    } while (choice == 'y' || choice == 'Y');

    separator();
    printf("   REPORT\n");
    separator();
    printf("\n");

    for (i = 0; i < count; i++)
        print_sensor(&sensors[i]);

    separator();
    printf("  Sensors             : %d\n", count);
    printf("  Average temperature : %.1f C\n", average_temp(sensors, count));
    printf("  Alerts              : %d\n", count_alerts(sensors, count));
    printf("  Log saved to        : %s\n", LOG_FILE);
    separator();

    fprintf(log, "=== End of session ===\n\n");
    fclose(log);
    return 0;
}

```

## 🧱 Использованные конструкции C

- `struct` / `typedef` — структура данных датчика
- Массив структур `Sensor sensors[10]`
- Передача структуры через указатель `Sensor *s`
- Файловый ввод-вывод: `fopen`, `fprintf`, `fclose`
- Цикл `do...while` для ввода нескольких датчиков
- Директивы `#define` для пороговых значений
- Функции: `time()`, `strftime()`, `scanf()`, `printf()`

-----

## 👤 Автор

**Кирилл** — студент 1 курса  
GitHub: [@kirillsamy](https://github.com/kirillsamy)
