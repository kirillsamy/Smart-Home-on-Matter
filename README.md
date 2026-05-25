# Smart-Home-on-Matter

Курсовая работа по дисциплине «Программирование»  
Язык: **C** | Протокол: **Matter 1.3**

-----/*

- Умный дом — протокол Matter
- Курсовая работа, 2 курс
- 
- Программа:
- 1. Принимает показания датчиков вручную (с клавиатуры)
- 1. Проверяет нормы и выводит предупреждения
- 1. Сохраняет все данные в файл log.txt
- 1. Показывает итоговый отчёт
   */

#include <stdio.h>
#include <string.h>
#include <time.h>

/* ===== КОНСТАНТЫ ===== */
#define MAX_ROOMS    10          /* максимум комнат                */
#define LOG_FILE     “log.txt”   /* файл для сохранения данных     */
#define TEMP_MAX     26.0f       /* норма температуры, °C          */
#define HUMIDITY_MAX 70.0f       /* норма влажности, %             */
#define CO2_MAX      800         /* норма CO2, ppm                 */

/* ===== СТРУКТУРА ДАТЧИКА ===== */
typedef struct {
int   node_id;       /* номер устройства в сети Matter */
char  room[32];      /* название комнаты               */
float temperature;   /* температура, °C                */
float humidity;      /* влажность, %                   */
int   co2;           /* углекислый газ, ppm            */
int   motion;        /* движение: 1=да, 0=нет          */
} Sensor;

/* ===== ФУНКЦИИ ===== */

/* Возвращает текущее время в виде строки “ЧЧ:ММ:СС” */
void get_time(char *buf, int size) {
time_t now = time(NULL);
strftime(buf, size, “%H:%M:%S”, localtime(&now));
}

/* Разделитель для красивого вывода */
void separator(void) {
printf(”================================================\n”);
}

/* Вводит данные одного датчика с клавиатуры */
void input_sensor(Sensor *s, int number) {
int motion_input;

```
separator();
printf("  Датчик #%d\n", number);
separator();

s->node_id = number;

printf("Название комнаты: ");
scanf("%31s", s->room);

printf("Температура (C):   ");
scanf("%f", &s->temperature);

printf("Влажность (%%):     ");
scanf("%f", &s->humidity);

printf("Уровень CO2 (ppm): ");
scanf("%d", &s->co2);

printf("Есть движение? (1=да, 0=нет): ");
scanf("%d", &motion_input);
s->motion = motion_input;

printf("\n");
```

}

/* Выводит данные одного датчика на экран с проверкой норм */
void print_sensor(const Sensor *s) {
printf(”  [Node 0x%04X] %s\n”, s->node_id, s->room);

```
printf("    Температура  : %.1f C", s->temperature);
if (s->temperature > TEMP_MAX)
    printf("  <<< ВЫШЕ НОРМЫ (до %.0f C)", TEMP_MAX);
printf("\n");

printf("    Влажность    : %.1f %%", s->humidity);
if (s->humidity > HUMIDITY_MAX)
    printf("  <<< ВЫШЕ НОРМЫ (до %.0f %%)", HUMIDITY_MAX);
printf("\n");

printf("    CO2          : %d ppm", s->co2);
if (s->co2 > CO2_MAX)
    printf("  <<< ВЫШЕ НОРМЫ (до %d ppm)", CO2_MAX);
printf("\n");

printf("    Движение     : %s\n\n", s->motion ? "ОБНАРУЖЕНО" : "нет");
```

}

/* Сохраняет данные датчика в файл */
void save_to_file(FILE *f, const Sensor *s, const char *time_str) {
fprintf(f, “[%s] Комната: %s (Node 0x%04X)\n”, time_str, s->room, s->node_id);
fprintf(f, “  Температура : %.1f C%s\n”,
s->temperature,
s->temperature > TEMP_MAX ? “ [ПРЕВЫШЕНА НОРМА]” : “”);
fprintf(f, “  Влажность   : %.1f %%%s\n”,
s->humidity,
s->humidity > HUMIDITY_MAX ? “ [ПРЕВЫШЕНА НОРМА]” : “”);
fprintf(f, “  CO2         : %d ppm%s\n”,
s->co2,
s->co2 > CO2_MAX ? “ [ПРЕВЫШЕНА НОРМА]” : “”);
fprintf(f, “  Движение    : %s\n\n”, s->motion ? “да” : “нет”);
}

/* Считает среднюю температуру по всем датчикам */
float average_temp(const Sensor sensors[], int count) {
float sum = 0.0f;
int i;
for (i = 0; i < count; i++)
sum += sensors[i].temperature;
return sum / count;
}

/* Считает количество тревог (превышений норм) */
int count_alerts(const Sensor sensors[], int count) {
int alerts = 0, i;
for (i = 0; i < count; i++) {
if (sensors[i].temperature > TEMP_MAX)  alerts++;
if (sensors[i].humidity    > HUMIDITY_MAX) alerts++;
if (sensors[i].co2        > CO2_MAX)    alerts++;
}
return alerts;
}

/* ===== ГЛАВНАЯ ФУНКЦИЯ ===== */
int main(void) {
Sensor sensors[MAX_ROOMS];
int    count = 0;
char   choice;
char   time_str[32];
int    i;

```
/* Открываем файл лога (добавляем в конец) */
FILE *log = fopen(LOG_FILE, "a");
if (log == NULL) {
    printf("Ошибка: не удалось открыть файл %s\n", LOG_FILE);
    return 1;
}

get_time(time_str, sizeof(time_str));
fprintf(log, "======= Новый сеанс: %s =======\n\n", time_str);

/* Заголовок */
separator();
printf("   УМНЫЙ ДОМ — протокол Matter 1.3\n");
printf("   Ввод показаний датчиков\n");
separator();
printf("\n");

/* Цикл ввода датчиков */
do {
    if (count >= MAX_ROOMS) {
        printf("Достигнут максимум датчиков (%d).\n", MAX_ROOMS);
        break;
    }

    input_sensor(&sensors[count], count + 1);

    get_time(time_str, sizeof(time_str));
    save_to_file(log, &sensors[count], time_str);

    count++;

    printf("Добавить ещё один датчик? (y/n): ");
    scanf(" %c", &choice);
    printf("\n");

} while (choice == 'y' || choice == 'Y');

/* Итоговый отчёт */
separator();
printf("   ОТЧЁТ ПО ВСЕМ ДАТЧИКАМ\n");
separator();
printf("\n");

for (i = 0; i < count; i++)
    print_sensor(&sensors[i]);

separator();
printf("  Всего датчиков      : %d\n", count);
printf("  Средняя температура : %.1f C\n", average_temp(sensors, count));
printf("  Количество тревог   : %d\n", count_alerts(sensors, count));
printf("  Данные сохранены в  : %s\n", LOG_FILE);
separator();

fprintf(log, "======= Конец сеанса =======\n\n");
fclose(log);

return 0;
```

}

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
