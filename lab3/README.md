# Основы обработки данных с помощью R и Dplyr

author: Meh

## Цель работы

1.  Развить практические навыки использования языка программирования R для обработки данных
2.  Закрепить знания базовых типов данных языка R
3.  Развить практические навыки использования функций обработки данных пакета dplyr – функции `select()`, `filter()`, `mutate()`, `arrange()`, `group_by()`

## Исходные данные

1.  Программное обеспечение Windows 10 Pro
2.  Visual Studio Code с установленными плагинами для работы с языком R
3.  Интерпретатор языка R 4.5.1

## План

Проанализировать встроенные в пакет `nycflights13` наборы данных с помощью языка R и ответить на вопросы

## Шаги:

1.  Загрузка библиотек

```{r}
    library(nycflights13)
    library(dplyr)
```
```
Присоединяю пакет: ‘dplyr’

Следующие объекты скрыты от ‘package:stats’:

    filter, lag

Следующие объекты скрыты от ‘package:base’:

    intersect, setdiff, setequal, union
    ```

Сколько встроенных в пакет `nycflights13` датафреймов?

```{r}
    nrow(data(package = "nycflights13")$results)
```
```
[1] 5
```

Сколько строк в каждом датафрейме?

```{r}
    sapply(list(flights, airlines, airports, planes, weather), nrow)
```
```
[1] 336776     16   1458   3322  26115
```

Сколько столбцов в каждом датафрейме?

```{r}
sapply(list(flights, airlines, airports, planes, weather), ncol)
```
```
[1] 19  2  8  9 15
```


Как просмотреть примерный вид датафрейма?

```{r}
glimpse(flights)
```
```
Rows: 336,776
Columns: 19
$ year           <int> 2013, 2013, 2013, 2013, 2013, 2013, 2013, 2013, 2013, 2013, 2013, 2013, …
$ month          <int> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, …
$ day            <int> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, …
$ dep_time       <int> 517, 533, 542, 544, 554, 554, 555, 557, 557, 558, 558, 558, 558, 558, 55…
$ sched_dep_time <int> 515, 529, 540, 545, 600, 558, 600, 600, 600, 600, 600, 600, 600, 600, 60…
$ dep_delay      <dbl> 2, 4, 2, -1, -6, -4, -5, -3, -3, -2, -2, -2, -2, -2, -1, 0, -1, 0, 0, 1,…
$ arr_time       <int> 830, 850, 923, 1004, 812, 740, 913, 709, 838, 753, 849, 853, 924, 923, 9…
$ sched_arr_time <int> 819, 830, 850, 1022, 837, 728, 854, 723, 846, 745, 851, 856, 917, 937, 9…
$ arr_delay      <dbl> 11, 20, 33, -18, -25, 12, 19, -14, -8, 8, -2, -3, 7, -14, 31, -4, -8, -7…
$ carrier        <chr> "UA", "UA", "AA", "B6", "DL", "UA", "B6", "EV", "B6", "AA", "B6", "B6", …
$ flight         <int> 1545, 1714, 1141, 725, 461, 1696, 507, 5708, 79, 301, 49, 71, 194, 1124,…
$ tailnum        <chr> "N14228", "N24211", "N619AA", "N804JB", "N668DN", "N39463", "N516JB", "N…
$ origin         <chr> "EWR", "LGA", "JFK", "JFK", "LGA", "EWR", "EWR", "LGA", "JFK", "LGA", "J…
$ dest           <chr> "IAH", "IAH", "MIA", "BQN", "ATL", "ORD", "FLL", "IAD", "MCO", "ORD", "P…
$ air_time       <dbl> 227, 227, 160, 183, 116, 150, 158, 53, 140, 138, 149, 158, 345, 361, 257…
$ distance       <dbl> 1400, 1416, 1089, 1576, 762, 719, 1065, 229, 944, 733, 1028, 1005, 2475,…
$ hour           <dbl> 5, 5, 5, 5, 6, 5, 6, 6, 6, 6, 6, 6, 6, 6, 6, 5, 6, 6, 6, 6, 6, 6, 6, 6, …
$ minute         <dbl> 15, 29, 40, 45, 0, 58, 0, 0, 0, 0, 0, 0, 0, 0, 0, 59, 0, 0, 0, 0, 10, 5,…
$ time_hour      <dttm> 2013-01-01 05:00:00, 2013-01-01 05:00:00, 2013-01-01 05:00:00, 2013-01-…
```


```{r}
head(airports)
```
```
A tibble:6 × 8
faa
<chr>
name
<chr>
lat
<dbl>
lon
<dbl>
alt
<dbl>
tz
<dbl>
dst
<chr>
tzone
<chr>
04G	Lansdowne Airport	41.13047	-80.61958	1044	-5	A	America/New_York
06A	Moton Field Municipal Airport	32.46057	-85.68003	264	-6	A	America/Chicago
06C	Schaumburg Regional	41.98934	-88.10124	801	-6	A	America/Chicago
06N	Randall Airport	41.43191	-74.39156	523	-5	A	America/New_York
09J	Jekyll Island Airport	31.07447	-81.42778	11	-5	A	America/New_York
0A9	Elizabethton Municipal Airport	36.37122	-82.17342	1593	-5	A	America/New_York
```


Сколько компаний-перевозчиков (carrier) учитывают эти наборы данных (представлено в наборах данных)?

```{r}
flights %>%
distinct(carrier) %>%
nrow()
```
```
[1] 16
```


Сколько рейсов принял аэропорт John F Kennedy Intl в мае?

```{r}
flights %>%
filter(dest == "JFK", month == 5) %>%
nrow()
```
```
[1] 0
```


Какой самый северный аэропорт?

```{r}
airports %>%
arrange(desc(lat)) %>%
slice(1) %>%
pull(name)
```
```
[1] "Dillant Hopkins Airport"
```


Какой аэропорт самый высокогорный (находится выше всех над уровнем моря)?

```{r}
airports %>%
arrange(desc(alt)) %>%
slice(1) %>%
pull(name)
```
```
[1] "Telluride"
```


Какие бортовые номера у самых старых самолетов?

```{r}
planes %>%
filter(!is.na(year)) %>%
arrange(year) %>%
slice(1:5) %>%
pull(tailnum)
```
```
[1] "N381AA" "N201AA" "N567AA" "N378AA" "N575AA"
```


Какая средняя температура воздуха была в сентябре в аэропорту John F Kennedy Intl (в градусах Цельсия)?

```{r}
weather %>%
filter(origin == "JFK", month == 9) %>%
summarise(avg_temp_C = mean((temp - 32) * 5/9, na.rm = TRUE)) %>%
pull(avg_temp_C)
```
```
[1] 19.38764
```


Самолеты какой авиакомпании совершили больше всего вылетов в июне?

```{r}
flights %>%
filter(month == 6) %>%
count(carrier, sort = TRUE) %>%
slice(1) %>%
pull(carrier)
```
```
[1] "UA"
```


Самолеты какой авиакомпании задерживались чаще других в 2013 году?

```{r}
flights %>%
filter(!is.na(dep_delay)) %>%
group_by(carrier) %>%
summarise(delay_rate = mean(dep_delay > 0, na.rm = TRUE)) %>%
arrange(desc(delay_rate)) %>%
slice(1) %>%
pull(carrier)
```
```
[1] "WN"
```

## Оценка результата

В результате лабораторной работы мы развили практические навыки использования языка программирования R для обработки данных и закрепили знания базовых типов данных языка R

## Вывод

Таким образом, мы научились использовать функции обработки данных пакета `dplyr` – функций `select()`, `filter()`, `mutate()`, `arrange()`, `group_by()`.
