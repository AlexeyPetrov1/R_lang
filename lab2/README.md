
# Основы обработки данных с помощью R и Dplyr
Meh

## Цель работы

1.  Развить практические навыки использования языка программирования R
    для обработки данных
2.  Закрепить знания базовых типов данных языка R
3.  Развить практические навыки использования функций обработки данных
    пакета dplyr – функции `select()`, `filter()`, `mutate()`,
    `arrange()`, `group_by()`

## Исходные данные

1.  Программное обеспечение Windows 10 Pro
2.  Visual Studio Code с установленными плагинами для работы с языком R
3.  Интерпретатор языка R 4.5.1

## План

Проанализировать встроенный в пакет `dplyr` набор данных `starwars` с
помощью языка R и ответить на вопросы

## Шаги:

1.  Загрузка библиотеки

    ``` r
    library(dplyr)
    ```

        Warning: пакет 'dplyr' был собран под R версии 4.4.2


        Присоединяю пакет: 'dplyr'

        Следующие объекты скрыты от 'package:stats':

            filter, lag

        Следующие объекты скрыты от 'package:base':

            intersect, setdiff, setequal, union

2.  Сколько строк в датафрейме?

    ``` r
    starwars %>% ncol()
    ```

        [1] 14

3.  Как просмотреть примерный вид датафрейма?

    ``` r
    starwars %>% glimpse()
    ```

        Rows: 87
        Columns: 14
        $ name       <chr> "Luke Skywalker", "C-3PO", "R2-D2", "Darth Vader", "Leia Or…
        $ height     <int> 172, 167, 96, 202, 150, 178, 165, 97, 183, 182, 188, 180, 2…
        $ mass       <dbl> 77.0, 75.0, 32.0, 136.0, 49.0, 120.0, 75.0, 32.0, 84.0, 77.…
        $ hair_color <chr> "blond", NA, NA, "none", "brown", "brown, grey", "brown", N…
        $ skin_color <chr> "fair", "gold", "white, blue", "white", "light", "light", "…
        $ eye_color  <chr> "blue", "yellow", "red", "yellow", "brown", "blue", "blue",…
        $ birth_year <dbl> 19.0, 112.0, 33.0, 41.9, 19.0, 52.0, 47.0, NA, 24.0, 57.0, …
        $ sex        <chr> "male", "none", "none", "male", "female", "male", "female",…
        $ gender     <chr> "masculine", "masculine", "masculine", "masculine", "femini…
        $ homeworld  <chr> "Tatooine", "Tatooine", "Naboo", "Tatooine", "Alderaan", "T…
        $ species    <chr> "Human", "Droid", "Droid", "Human", "Human", "Human", "Huma…
        $ films      <list> <"A New Hope", "The Empire Strikes Back", "Return of the J…
        $ vehicles   <list> <"Snowspeeder", "Imperial Speeder Bike">, <>, <>, <>, "Imp…
        $ starships  <list> <"X-wing", "Imperial shuttle">, <>, <>, "TIE Advanced x1",…

4.  Сколько уникальных рас персонажей (species) представлено в данных?

    ``` r
    starwars %>% pull(species) %>% unique() %>% length()
    ```

        [1] 38

5.  Найти самого высокого персонажа

    ``` r
    starwars %>%
      filter(!is.na(height)) %>%
      arrange(desc(height)) %>%
      slice(1)
    ```

        # A tibble: 1 × 14
          name      height  mass hair_color skin_color eye_color birth_year sex   gender
          <chr>      <int> <dbl> <chr>      <chr>      <chr>          <dbl> <chr> <chr> 
        1 Yarael P…    264    NA none       white      yellow            NA male  mascu…
        # ℹ 5 more variables: homeworld <chr>, species <chr>, films <list>,
        #   vehicles <list>, starships <list>

6.  Найти всех персонажей ниже 170

    ``` r
    starwars %>% filter(height < 170)
    ```

        # A tibble: 22 × 14
           name     height  mass hair_color skin_color eye_color birth_year sex   gender
           <chr>     <int> <dbl> <chr>      <chr>      <chr>          <dbl> <chr> <chr> 
         1 C-3PO       167    75 <NA>       gold       yellow           112 none  mascu…
         2 R2-D2        96    32 <NA>       white, bl… red               33 none  mascu…
         3 Leia Or…    150    49 brown      light      brown             19 fema… femin…
         4 Beru Wh…    165    75 brown      light      blue              47 fema… femin…
         5 R5-D4        97    32 <NA>       white, red red               NA none  mascu…
         6 Yoda         66    17 white      green      brown            896 male  mascu…
         7 Mon Mot…    150    NA auburn     fair       blue              48 fema… femin…
         8 Wicket …     88    20 brown      brown      brown              8 male  mascu…
         9 Nien Nu…    160    68 none       grey       black             NA male  mascu…
        10 Watto       137    NA black      blue, grey yellow            NA male  mascu…
        # ℹ 12 more rows
        # ℹ 5 more variables: homeworld <chr>, species <chr>, films <list>,
        #   vehicles <list>, starships <list>

7.  Подсчитать ИМТ (индекс массы тела) для всех персонажей. ИМТ
    подсчитать по формуле

    ``` r
    mutate(starwars, BMI=mass/(height^2)) %>% 
        select(name, height, mass, BMI)
    ```

        # A tibble: 87 × 4
           name               height  mass     BMI
           <chr>               <int> <dbl>   <dbl>
         1 Luke Skywalker        172    77 0.00260
         2 C-3PO                 167    75 0.00269
         3 R2-D2                  96    32 0.00347
         4 Darth Vader           202   136 0.00333
         5 Leia Organa           150    49 0.00218
         6 Owen Lars             178   120 0.00379
         7 Beru Whitesun Lars    165    75 0.00275
         8 R5-D4                  97    32 0.00340
         9 Biggs Darklighter     183    84 0.00251
        10 Obi-Wan Kenobi        182    77 0.00232
        # ℹ 77 more rows

8.  Найти 10 самых “вытянутых” персонажей. “Вытянутость” оценить по
    отношению массы (mass) к росту (height) персонажей

    ``` r
    starwars %>%
      filter(!is.na(mass), !is.na(height), height > 0) %>%
      mutate(stretchiness = mass / height) %>%
      arrange(desc(stretchiness)) %>%
      head(10)
    ```

        # A tibble: 10 × 15
           name     height  mass hair_color skin_color eye_color birth_year sex   gender
           <chr>     <int> <dbl> <chr>      <chr>      <chr>          <dbl> <chr> <chr> 
         1 Jabba D…    175  1358 <NA>       green-tan… orange         600   herm… mascu…
         2 Grievous    216   159 none       brown, wh… green, y…       NA   male  mascu…
         3 IG-88       200   140 none       metal      red             15   none  mascu…
         4 Owen La…    178   120 brown, gr… light      blue            52   male  mascu…
         5 Darth V…    202   136 none       white      yellow          41.9 male  mascu…
         6 Jek Ton…    180   110 brown      fair       blue            NA   <NA>  <NA>  
         7 Bossk       190   113 none       green      red             53   male  mascu…
         8 Tarfful     234   136 brown      brown      blue            NA   male  mascu…
         9 Dexter …    198   102 none       brown      yellow          NA   male  mascu…
        10 Chewbac…    228   112 brown      unknown    blue           200   male  mascu…
        # ℹ 6 more variables: homeworld <chr>, species <chr>, films <list>,
        #   vehicles <list>, starships <list>, stretchiness <dbl>

9.  Найти средний возраст персонажей каждой раcы вселенной Звездных войн

    ``` r
    starwars %>%
        group_by(species) %>%
        summarise(Avg_age = median(100+birth_year, na.rm = TRUE)) 
    ```

        # A tibble: 38 × 2
           species   Avg_age
           <chr>       <dbl>
         1 Aleena         NA
         2 Besalisk       NA
         3 Cerean        192
         4 Chagrian       NA
         5 Clawdite       NA
         6 Droid         133
         7 Dug            NA
         8 Ewok          108
         9 Geonosian      NA
        10 Gungan        152
        # ℹ 28 more rows

10. Найти самый распространенный цвет глаз персонажей вселенной Звездных
    войн

    ``` r
    starwars %>%
      count(eye_color, sort = TRUE) %>%
      slice(1)
    ```

        # A tibble: 1 × 2
          eye_color     n
          <chr>     <int>
        1 brown        21

11. Подсчитать среднюю длину имени в каждой расе

    ``` r
    starwars %>%
      filter(!is.na(species), !is.na(name)) %>%
      mutate(name_length = nchar(name)) %>%
      group_by(species) %>%
      summarise(avg_name_length = mean(name_length))
    ```

        # A tibble: 37 × 2
           species   avg_name_length
           <chr>               <dbl>
         1 Aleena              12   
         2 Besalisk            15   
         3 Cerean              12   
         4 Chagrian            10   
         5 Clawdite            10   
         6 Droid                4.83
         7 Dug                  7   
         8 Ewok                21   
         9 Geonosian           17   
        10 Gungan              11.7 
        # ℹ 27 more rows

## Оценка результата

В результате лабораторной работы мы получили практические навыки
использования языка программирования “R” для обработки данных и знания
базовых типов данных языка R

## Вывод

Итого, мы получили практические навыки использования функций обработки
данных пакета dplyr, а именно функций `select()`, `filter()`,
`mutate()`, `arrange()`, `group_by()`
