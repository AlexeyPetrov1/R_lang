Lab 5
=====

Meh

Цель работы
-----------

1.  Получить навыки исследования радиоэлектронной обстановки.
2.  Изучить принципы работы сетей Wi-Fi на канальном и сетевом уровнях (OSI).
3.  Отработать навыки использования языка R для процессинга данных.
4.  Закрепить использование инструментов экосистемы tidyverse.

Исходные данные
---------------

1.  ОС: Windows 10
2.  IDE: RStudio
3.  R Version: 4.5.2

Ход работы
----------

1.  **Подготовка**: Импорт данных `wifi` и базы данных OUI вендоров. Очистка и приведение типов.
2.  **Анализ Access Points (AP)**:
    *   Поиск открытых сетей (OPN).
    *   Определение производителей оборудования.
    *   Поиск WPA3 и SAE.
    *   Анализ длительности сессий и топ самых быстрых точек.
    *   Анализ нагрузки (beacons).
3.  **Анализ Клиентов (Stations)**:
    *   Определение производителей.
    *   Выявление устройств без MAC-рандомизации.
    *   Кластеризация запросов (Probes) и анализ стабильности сигнала.

Шаг 1: Загрузка и подготовка данных
-----------------------------------

Укажем путь к файлу и загрузим необходимые пакеты.

   ``` # Пути к файлам
    path_wifi_data <- "P2_wifi_data.csv"
    path_vendor_json <- "oui_vendor.json"
```
Загрузка базы данных вендоров (OUI).
```
    library(dplyr)
```
 ```   
    Присоединяю пакет: 'dplyr'

    Следующие объекты скрыты от 'package:stats':
    
        filter, lag

    Следующие объекты скрыты от 'package:base':
    
        intersect, setdiff, setequal, union
```
```
    library(jsonlite)
    
    # Читаем JSON и сразу преобразуем в tibble для удобства
    vendor_dict <- fromJSON(path_vendor_json) %>% 
      as_tibble()
```
Подключение библиотек для анализа.
```
    library(tidyverse)
```
```
    ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
    ✔ forcats   1.0.1     ✔ readr     2.1.6
    ✔ ggplot2   4.0.1     ✔ stringr   1.6.0
    ✔ lubridate 1.9.4     ✔ tibble    3.3.0
    ✔ purrr     1.2.0     ✔ tidyr     1.3.1
    ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ✖ dplyr::filter()  masks stats::filter()
    ✖ purrr::flatten() masks jsonlite::flatten()
    ✖ dplyr::lag()     masks stats::lag()
    ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors
```
```
    library(lubridate)
    library(readr)
    library(janitor)
```
 ```   
    Присоединяю пакет: 'janitor'
    
    Следующие объекты скрыты от 'package:stats':
    
        chisq.test, fisher.test
```
```
    library(stringr)
```
Чтение и парсинг основного датасета. Разделим файл на две части: Точки доступа (AP) и Клиенты (Stations).
```
    # Читаем сырые строки, чтобы найти границы таблиц
    raw_lines <- read_lines(path_wifi_data)
    
    # Находим индексы заголовков
    idx_ap_start      <- which(str_starts(raw_lines, "BSSID"))[1]
    idx_station_start <- which(str_starts(raw_lines, "Station MAC"))[1]
    
    # Считываем блок с точками доступа
    access_points <- read_csv(
      path_wifi_data,
      skip = idx_ap_start - 1,
      n_max = idx_station_start - idx_ap_start - 2,
      show_col_types = FALSE
    )
    
    # Считываем блок с клиентами
    clients <- read_csv(
      path_wifi_data,
      skip = idx_station_start - 1,
      show_col_types = FALSE
    )
```
```
    Warning: One or more parsing issues, call `problems()` on your data frame for details,
    e.g.:
      dat <- vroom(...)
      problems(dat)
```
```
    # Нормализуем имена колонок
    access_points <- clean_names(access_points)
    clients       <- clean_names(clients)
    
    # Проверка имен
    names(access_points)
```
```
     [1] "bssid"           "first_time_seen" "last_time_seen"  "channel"        
     [5] "speed"           "privacy"         "cipher"          "authentication" 
     [9] "power"           "number_beacons"  "number_iv"       "lan_ip"         
    [13] "id_length"       "essid"           "key"            
```
```
    names(clients)
```
```
    [1] "station_mac"     "first_time_seen" "last_time_seen"  "power"          
    [5] "number_packets"  "bssid"           "probed_essi_ds" 
```
```
    # Преобразование типов данных для AP
    access_points <- access_points %>%
      rename(
        t_start = first_time_seen,
        t_end   = last_time_seen,
        n_beacons = number_beacons,
        n_iv    = number_iv
      ) %>%
      mutate(
        t_start   = ymd_hms(str_trim(t_start)),
        t_end     = ymd_hms(str_trim(t_end)),
        channel   = as.integer(channel),
        speed     = as.numeric(speed),
        power     = as.numeric(power),
        n_beacons = as.numeric(n_beacons),
        n_iv      = as.numeric(n_iv),
        id_len    = as.integer(id_length),
        lan_ip    = str_squish(lan_ip),
        # Вычисляем длительность сеанса
        duration_s = as.numeric(difftime(t_end, t_start, units = "secs")),
        duration_m = duration_s / 60
      )
    
    glimpse(access_points)
```
```
    Rows: 167
    Columns: 18
    $ bssid          <chr> "BE:F1:71:D5:17:8B", "6E:C7:EC:16:DA:1A", "9A:75:A8:B9:…
    $ t_start        <dttm> 2023-07-28 09:13:03, 2023-07-28 09:13:03, 2023-07-28 0…
    $ t_end          <dttm> 2023-07-28 11:50:50, 2023-07-28 11:55:12, 2023-07-28 1…
    $ channel        <int> 1, 1, 1, 7, 6, 6, 11, 11, 11, 1, 6, 14, 11, 11, 6, 6, 6…
    $ speed          <dbl> 195, 130, 360, 360, 130, 130, 195, 130, 130, 195, 180, …
    $ privacy        <chr> "WPA2", "WPA2", "WPA2", "WPA2", "WPA2", "OPN", "WPA2", …
    $ cipher         <chr> "CCMP", "CCMP", "CCMP", "CCMP", "CCMP", NA, "CCMP", "CC…
    $ authentication <chr> "PSK", "PSK", "PSK", "PSK", "PSK", NA, "PSK", "PSK", "P…
    $ power          <dbl> -30, -30, -68, -37, -57, -63, -27, -38, -38, -66, -42, …
    $ n_beacons      <dbl> 846, 750, 694, 510, 647, 251, 1647, 1251, 704, 617, 139…
    $ n_iv           <dbl> 504, 116, 26, 21, 6, 3430, 80, 11, 0, 0, 86, 0, 0, 0, 9…
    $ lan_ip         <chr> "0. 0. 0. 0", "0. 0. 0. 0", "0. 0. 0. 0", "0. 0. 0. 0",…
    $ id_length      <dbl> 12, 4, 2, 14, 25, 13, 12, 13, 24, 12, 10, 0, 24, 24, 12…
    $ essid          <chr> "C322U13 3965", "Cnet", "KC", "POCO X5 Pro 5G", NA, "MI…
    $ key            <lgl> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA,…
    $ id_len         <int> 12, 4, 2, 14, 25, 13, 12, 13, 24, 12, 10, 0, 24, 24, 12…
    $ duration_s     <dbl> 9467, 9729, 9628, 6658, 4636, 9755, 9461, 8608, 4319, 4…
    $ duration_m     <dbl> 157.78333, 162.15000, 160.46667, 110.96667, 77.26667, 1…
```
Преобразование типов данных для Клиентов.
```
    clients <- clients %>%
      rename(
        mac_addr = station_mac,
        t_start  = first_time_seen,
        t_end    = last_time_seen,
        pkt_cnt  = number_packets,
        probes   = probed_essi_ds
      ) %>%
      mutate(
        t_start = ymd_hms(str_trim(t_start)),
        t_end   = ymd_hms(str_trim(t_end)),
        power   = as.numeric(power),
        pkt_cnt = as.numeric(pkt_cnt),
        duration_s = as.numeric(difftime(t_end, t_start, units = "secs")),
        duration_m = duration_s / 60
      )
    
    glimpse(clients)
```
```
    Rows: 12,081
    Columns: 9
    $ mac_addr   <chr> "CA:66:3B:8F:56:DD", "96:35:2D:3D:85:E6", "5C:3A:45:9E:1A:7…
    $ t_start    <dttm> 2023-07-28 09:13:03, 2023-07-28 09:13:03, 2023-07-28 09:13…
    $ t_end      <dttm> 2023-07-28 10:59:44, 2023-07-28 09:13:03, 2023-07-28 11:51…
    $ power      <dbl> -33, -65, -39, -61, -53, -43, -31, -71, -74, -65, -45, -65,…
    $ pkt_cnt    <dbl> 858, 4, 432, 958, 1, 344, 163, 3, 115, 437, 265, 77, 7, 71,…
    $ bssid      <chr> "BE:F1:71:D5:17:8B", "(not associated)", "BE:F1:71:D6:10:D7…
    $ probes     <chr> "C322U13 3965", "IT2 Wireless", "C322U21 0566", "C322U13 39…
    $ duration_s <dbl> 6401, 0, 9531, 9613, 0, 9781, 9464, 415, 4041, 9718, 6285, …
    $ duration_m <dbl> 106.68333333, 0.00000000, 158.85000000, 160.21666667, 0.000…
```
Очистка справочника вендоров.
```
    raw_manuf <- vendor_dict %>%
      mutate(`_id` = as.character(`_id`)) %>%
      select(oui_id = `_id`, company_name = company)
    
    # Статистика по пропущенным значениям
    raw_manuf %>%
      summarise(
        total    = n(),
        na_ids   = sum(is.na(oui_id) | oui_id == ""),
        na_names = sum(is.na(company_name) | company_name == "")
      )
```
```
    # A tibble: 1 × 3
      total na_ids na_names
      <int>  <int>    <int>
    1 54342      0       37
```
```
    # Создаем чистый справочник (OUI -> Vendor)
    oui_map <- raw_manuf %>%
      mutate(
        oui_full = toupper(oui_id),
        oui_prefix = substr(oui_full, 1, 6),
        company_name  = if_else(is.na(company_name) | company_name == "", "Unknown", company_name)
      ) %>%
      group_by(oui_prefix) %>%
      summarise(
        vendor_name = first(company_name),
        .groups = "drop"
      )
    
    head(oui_map)
```
```
    # A tibble: 6 × 2
      oui_prefix vendor_name      
      <chr>      <chr>            
    1 000000     XEROX CORPORATION
    2 000001     XEROX CORPORATION
    3 000002     XEROX CORPORATION
    4 000003     XEROX CORPORATION
    5 000004     XEROX CORPORATION
    6 000005     XEROX CORPORATION
```
Объединение (Join) AP и справочника вендоров.
```
    # Функция-хелпер для извлечения OUI из MAC
    get_oui_prefix <- function(mac_string) {
      mac_string %>%
        str_replace_all("[:\\-]", "") %>%
        toupper() %>%
        substr(1, 6)
    }
    
    ap_enriched <- access_points %>%
      mutate(oui_prefix = get_oui_prefix(bssid)) %>%
      left_join(oui_map, by = "oui_prefix")
    
    head(ap_enriched)
```
```
    # A tibble: 6 × 20
      bssid     t_start             t_end               channel speed privacy cipher
      <chr>     <dttm>              <dttm>                <int> <dbl> <chr>   <chr> 
    1 BE:F1:71… 2023-07-28 09:13:03 2023-07-28 11:50:50       1   195 WPA2    CCMP  
    2 6E:C7:EC… 2023-07-28 09:13:03 2023-07-28 11:55:12       1   130 WPA2    CCMP  
    3 9A:75:A8… 2023-07-28 09:13:03 2023-07-28 11:53:31       1   360 WPA2    CCMP  
    4 4A:EC:1E… 2023-07-28 09:13:03 2023-07-28 11:04:01       7   360 WPA2    CCMP  
    5 D2:6D:52… 2023-07-28 09:13:03 2023-07-28 10:30:19       6   130 WPA2    CCMP  
    6 E8:28:C1… 2023-07-28 09:13:03 2023-07-28 11:55:38       6   130 OPN     <NA>  
    # ℹ 13 more variables: authentication <chr>, power <dbl>, n_beacons <dbl>,
    #   n_iv <dbl>, lan_ip <chr>, id_length <dbl>, essid <chr>, key <lgl>,
    #   id_len <int>, duration_s <dbl>, duration_m <dbl>, oui_prefix <chr>,
    #   vendor_name <chr>
```
Шаг 2: Анализ точек доступа
---------------------------

2.1. Поиск небезопасных сетей (OPN).
```
    unsafe_aps <- ap_enriched %>%
      filter(str_detect(privacy, "OPN"))
    
    unsafe_aps %>%
      select(bssid, essid, vendor_name, privacy, cipher, authentication, channel, speed, power)
```
```
    # A tibble: 42 × 9
       bssid     essid vendor_name privacy cipher authentication channel speed power
       <chr>     <chr> <chr>       <chr>   <chr>  <chr>            <int> <dbl> <dbl>
     1 E8:28:C1… MIRE… Eltex Ente… OPN     <NA>   <NA>                 6   130   -63
     2 E8:28:C1… MIRE… Eltex Ente… OPN     <NA>   <NA>                 6   130   -63
     3 E8:28:C1… <NA>  Eltex Ente… OPN     <NA>   <NA>                 6   130   -63
     4 E8:28:C1… <NA>  Eltex Ente… OPN     <NA>   <NA>                 6    -1    -1
     5 00:25:00… <NA>  Apple, Inc. OPN     <NA>   <NA>                44    -1    -1
     6 E8:28:C1… MIRE… Eltex Ente… OPN     <NA>   <NA>                11   130   -67
     7 E8:28:C1… <NA>  Eltex Ente… OPN     <NA>   <NA>                 6   130   -82
     8 E8:28:C1… MIRE… Eltex Ente… OPN     <NA>   <NA>                 6   130   -69
     9 E8:28:C1… MIRE… Eltex Ente… OPN     <NA>   <NA>                 1   130   -69
    10 E8:28:C1… MIRE… Eltex Ente… OPN     <NA>   <NA>                11   130   -78
    # ℹ 32 more rows
```
2.2. Определение производителя для устройств.
```
    ap_enriched %>%
      select(bssid, essid, vendor_name) %>%
      slice_head(n = 20)
```
```
    # A tibble: 20 × 3
       bssid             essid                    vendor_name            
       <chr>             <chr>                    <chr>                  
     1 BE:F1:71:D5:17:8B C322U13 3965             <NA>                   
     2 6E:C7:EC:16:DA:1A Cnet                     <NA>                   
     3 9A:75:A8:B9:04:1E KC                       <NA>                   
     4 4A:EC:1E:DB:BF:95 POCO X5 Pro 5G           <NA>                   
     5 D2:6D:52:61:51:5D <NA>                     <NA>                   
     6 E8:28:C1:DC:B2:52 MIREA_HOTSPOT            Eltex Enterprise Ltd.  
     7 BE:F1:71:D6:10:D7 C322U21 0566             <NA>                   
     8 0A:C5:E1:DB:17:7B AndroidAP177B            <NA>                   
     9 38:1A:52:0D:84:D7 EBFCD57F-EE81fI_DL_1AO2T Seiko Epson Corporation
    10 BE:F1:71:D5:0E:53 C322U06 9080             <NA>                   
    11 1E:93:E3:1B:3C:F4 Galaxy A71               <NA>                   
    12 1C:7E:E5:8E:B7:DE <NA>                     D-Link International   
    13 38:1A:52:0D:97:60 EBFCD593-EE81fI_DMJ1AOI4 Seiko Epson Corporation
    14 38:1A:52:0D:90:A1 EBFCD597-EE81fI_DMN1AOe1 Seiko Epson Corporation
    15 E8:28:C1:DC:B2:50 MIREA_GUESTS             Eltex Enterprise Ltd.  
    16 E8:28:C1:DC:B2:51 <NA>                     Eltex Enterprise Ltd.  
    17 8E:55:4A:85:5B:01 Vladimir                 <NA>                   
    18 E8:28:C1:DC:FF:F2 <NA>                     Eltex Enterprise Ltd.  
    19 00:25:00:FF:94:73 <NA>                     Apple, Inc.            
    20 00:26:99:F2:7A:E2 GIVC                     Cisco Systems, Inc     
```
2.3. Устройства с поддержкой WPA3 (SAE).
```
    wpa3_devices <- ap_enriched %>%
      filter(
        str_detect(privacy, "WPA3") | str_detect(authentication, "SAE")
      )
    
    wpa3_devices %>%
      select(bssid, essid, vendor_name, privacy, cipher, authentication, channel, speed)
```
```
    # A tibble: 8 × 8
      bssid            essid vendor_name privacy cipher authentication channel speed
      <chr>            <chr> <chr>       <chr>   <chr>  <chr>            <int> <dbl>
    1 26:20:53:0C:98:…  <NA> <NA>        WPA3 W… CCMP   SAE PSK             44   866
    2 A2:FE:FF:B8:9B:… "Chr… <NA>        WPA3 W… CCMP   SAE PSK              6   130
    3 96:FF:FC:91:EF:…  <NA> <NA>        WPA3 W… CCMP   SAE PSK             44   866
    4 CE:48:E7:86:4E:… "iPh… <NA>        WPA3 W… CCMP   SAE PSK             44   866
    5 8E:1F:94:96:DA:… "iPh… <NA>        WPA3 W… CCMP   SAE PSK             44   866
    6 BE:FD:EF:18:92:… "Дим… <NA>        WPA3 W… CCMP   SAE PSK              6   130
    7 3A:DA:00:F9:0C:… "iPh… <NA>        WPA3 W… CCMP   SAE PSK              6   130
    8 76:C5:A0:70:08:…  <NA> <NA>        WPA3 W… CCMP   SAE PSK              6   130
```
2.4. Сортировка по времени связи (с учетом разрывов сессий).
```
    # Логика разделения на сессии (если разрыв > 45 мин)
    ap_session_log <- ap_enriched %>%
      group_by(bssid) %>%
      arrange(t_start, .by_group = TRUE) %>%
      mutate(
        time_gap = as.numeric(difftime(
          t_start,
          lag(t_end, default = t_start[1]),
          units = "mins"
        )),
        is_new_session = if_else(is.na(time_gap) | time_gap > 45, 1L, 0L),
        session_idx    = cumsum(is_new_session)
      ) %>%
      group_by(bssid, session_idx, essid, channel, speed, vendor_name, privacy, cipher, authentication) %>%
      summarise(
        start_ts   = min(t_start),
        end_ts     = max(t_end),
        avg_power  = mean(power, na.rm = TRUE),
        total_bcns = sum(n_beacons, na.rm = TRUE),
        dur_sec    = as.numeric(difftime(end_ts, start_ts, units = "secs")),
        dur_min    = dur_sec / 60,
        .groups = "drop"
      )
    
    # Топ по длительности
    longest_sessions <- ap_session_log %>%
      arrange(desc(dur_min))
    
    longest_sessions %>%
      select(bssid, essid, vendor_name, dur_min, channel, speed, privacy) %>%
      head(20)
```
```
    # A tibble: 20 × 7
       bssid             essid         vendor_name     dur_min channel speed privacy
       <chr>             <chr>         <chr>             <dbl>   <int> <dbl> <chr>  
     1 00:25:00:FF:94:73 <NA>          Apple, Inc.        163.      44    -1 OPN    
     2 E8:28:C1:DD:04:52 MIREA_HOTSPOT Eltex Enterpri…    163.      11   130 OPN    
     3 E8:28:C1:DC:B2:52 MIREA_HOTSPOT Eltex Enterpri…    163.       6   130 OPN    
     4 08:3A:2F:56:35:FE <NA>          Guangzhou Juan…    162.      14    -1 WPA    
     5 6E:C7:EC:16:DA:1A Cnet          <NA>               162.       1   130 WPA2   
     6 E8:28:C1:DC:B2:50 MIREA_GUESTS  Eltex Enterpri…    162.       6   130 OPN    
     7 48:5B:39:F9:7A:48 <NA>          ASUSTek COMPUT…    162.       1   270 WPA2   
     8 E8:28:C1:DC:B2:51 <NA>          Eltex Enterpri…    162.       6   130 OPN    
     9 E8:28:C1:DC:FF:F2 <NA>          Eltex Enterpri…    162.       6    -1 OPN    
    10 8E:55:4A:85:5B:01 Vladimir      <NA>               162.       6    65 WPA2   
    11 00:26:99:BA:75:80 GIVC          Cisco Systems,…    162.      11    54 WPA2   
    12 00:26:99:F2:7A:E2 GIVC          Cisco Systems,…    162.       1    54 WPA2   
    13 1E:93:E3:1B:3C:F4 Galaxy A71    <NA>               161.       6   180 WPA2   
    14 0C:80:63:A9:6E:EE <NA>          TP-LINK TECHNO…    160.       3   270 WPA2   
    15 9A:75:A8:B9:04:1E KC            <NA>               160.       1   360 WPA2   
    16 00:23:EB:E3:81:F2 GIVC          Cisco Systems,…    160.       1    54 WPA2   
    17 9E:A3:A9:DB:7E:01 <NA>          <NA>               159.      14    65 WPA2   
    18 E8:28:C1:DC:C8:32 MIREA_HOTSPOT Eltex Enterpri…    159.       1   130 OPN    
    19 1C:7E:E5:8E:B7:DE <NA>          D-Link Interna…    159.      14    65 WPA2   
    20 00:26:99:F2:7A:E1 IKB           Cisco Systems,…    158.       1    54 WPA2   
```
2.5. Топ-10 самых быстрых точек доступа.
```
    fastest_aps <- ap_session_log %>%
      filter(speed > 0) %>%
      arrange(desc(speed), desc(dur_min)) %>%
      distinct(bssid, .keep_all = TRUE) %>%
      head(10) %>%
      select(bssid, essid, vendor_name, speed, channel, dur_min, privacy)
    
    fastest_aps
```
```
    # A tibble: 10 × 7
       bssid             essid             vendor_name speed channel dur_min privacy
       <chr>             <chr>             <chr>       <dbl>   <int>   <dbl> <chr>  
     1 96:FF:FC:91:EF:64 <NA>              <NA>          866      44   32.1  WPA3 W…
     2 26:20:53:0C:98:E8 <NA>              <NA>          866      44   17.4  WPA3 W…
     3 8E:1F:94:96:DA:FD iPhone (Анастаси… <NA>          866      44    6.92 WPA3 W…
     4 CE:48:E7:86:4E:33 iPhone (Анастаси… <NA>          866      44    4.92 WPA3 W…
     5 9A:75:A8:B9:04:1E KC                <NA>          360       1  160.   WPA2   
     6 E8:28:C1:DD:04:40 MIREA_HOTSPOT     Eltex Ente…   360      52  157.   OPN    
     7 E8:28:C1:DD:04:41 MIREA_GUESTS      Eltex Ente…   360      52  157.   OPN    
     8 E8:28:C1:DC:B2:40 MIREA_HOTSPOT     Eltex Ente…   360      48  154.   OPN    
     9 14:EB:B6:6A:76:37 Gnezdo_lounge 2   TP-Link Sy…   360       3  149.   WPA2   
    10 E8:28:C1:DC:B2:42 <NA>              Eltex Ente…   360      48  145.   OPN    
```
2.6. Сортировка по частоте beacon-фреймов.
```
    ap_beacon_analysis <- ap_session_log %>%
      mutate(
        safe_dur = pmax(dur_sec, 1), # избегаем деления на 0
        bps      = total_bcns / safe_dur # beacons per second
      ) %>%
      arrange(desc(bps))
    
    ap_beacon_analysis %>%
      select(bssid, essid, vendor_name, total_bcns, dur_min, bps, channel) %>%
      head(20)
```
```
    # A tibble: 20 × 7
       bssid             essid          vendor_name total_bcns dur_min   bps channel
       <chr>             <chr>          <chr>            <dbl>   <dbl> <dbl>   <int>
     1 00:03:7F:12:34:56 "MT_FREE"      Atheros Co…          1  0      1           6
     2 00:26:CB:AA:62:72 "GIVC"         Cisco Syst…          1  0      1           1
     3 02:CF:8B:87:B4:F9 "MT_FREE"      <NA>                 1  0      1          11
     4 6A:B0:1A:C2:DF:49  <NA>          <NA>                 1  0      1           6
     5 76:5E:F3:F9:A5:1C "Redmi 9C NFC" <NA>                 1  0      1           1
     6 76:E4:ED:B0:5C:9A "Инет от Пути… <NA>                 1  0      1          13
     7 A2:FE:FF:B8:9B:C9 "Christie’s"   <NA>                 1  0      1           6
     8 BA:2A:7A:DD:38:3E "Айфон (Oleg)" <NA>                 1  0      1           6
     9 C2:B5:D7:7F:07:A8 "DIRECT-a8-HP… <NA>                 1  0      1           6
    10 E0:D9:E3:49:00:B1  <NA>          Eltex Ente…          1  0      1           1
    11 E8:28:C1:DC:BD:52 "MIREA_HOTSPO… Eltex Ente…          1  0      1          11
    12 E8:28:C1:DE:47:D0 "MIREA_GUESTS" Eltex Ente…          1  0      1           1
    13 E8:28:C1:DE:47:D1  <NA>          Eltex Ente…          1  0      1           1
    14 F2:30:AB:E9:03:ED "iPhone (Ulia… <NA>                 6  0.117  0.857       1
    15 B2:CF:C0:00:4A:60 "Михаил's Gal… <NA>                 4  0.0833 0.8         6
    16 3A:DA:00:F9:0C:02 "iPhone XS Ma… <NA>                 5  0.15   0.556       6
    17 00:3E:1A:5D:14:45 "MT_FREE"      <NA>                 1  0.0333 0.5        11
    18 02:BC:15:7E:D5:DC "MT_FREE"      <NA>                 1  0.0333 0.5        11
    19 76:C5:A0:70:08:96  <NA>          <NA>                 1  0.0333 0.5         6
    20 D2:25:91:F6:6C:D8 "Саня"         <NA>                 5  0.217  0.385      12
```
Шаг 3: Анализ клиентов
----------------------

3.1. Определение вендоров клиентов.
```
    clients_enriched <- clients %>%
      mutate(oui_prefix = get_oui_prefix(mac_addr)) %>%
      left_join(oui_map, by = "oui_prefix")
    
    clients_enriched %>%
      select(mac_addr, oui_prefix, vendor_name, t_start, t_end, probes) %>%
      head(20)
```
```
    # A tibble: 20 × 6
       mac_addr       oui_prefix vendor_name t_start             t_end              
       <chr>          <chr>      <chr>       <dttm>              <dttm>             
     1 CA:66:3B:8F:5… CA663B     <NA>        2023-07-28 09:13:03 2023-07-28 10:59:44
     2 96:35:2D:3D:8… 96352D     <NA>        2023-07-28 09:13:03 2023-07-28 09:13:03
     3 5C:3A:45:9E:1… 5C3A45     CHONGQING … 2023-07-28 09:13:03 2023-07-28 11:51:54
     4 C0:E4:34:D8:E… C0E434     AzureWave … 2023-07-28 09:13:03 2023-07-28 11:53:16
     5 5E:8E:A6:5E:3… 5E8EA6     <NA>        2023-07-28 09:13:04 2023-07-28 09:13:04
     6 10:51:07:CB:3… 105107     Intel Corp… 2023-07-28 09:13:05 2023-07-28 11:56:06
     7 68:54:5A:40:3… 68545A     Intel Corp… 2023-07-28 09:13:06 2023-07-28 11:50:50
     8 74:4C:A1:70:C… 744CA1     Liteon Tec… 2023-07-28 09:13:06 2023-07-28 09:20:01
     9 8A:A3:5A:33:7… 8AA35A     <NA>        2023-07-28 09:13:06 2023-07-28 10:20:27
    10 CA:54:C4:8B:B… CA54C4     <NA>        2023-07-28 09:13:06 2023-07-28 11:55:04
    11 BC:F1:71:D4:D… BCF171     Intel Corp… 2023-07-28 09:13:07 2023-07-28 10:57:52
    12 4A:C9:28:46:E… 4AC928     <NA>        2023-07-28 09:13:08 2023-07-28 11:36:29
    13 A6:EC:3C:AB:B… A6EC3C     <NA>        2023-07-28 09:13:08 2023-07-28 09:13:09
    14 4C:44:5B:14:7… 4C445B     Intel Corp… 2023-07-28 09:13:09 2023-07-28 09:47:44
    15 9E:01:46:3E:E… 9E0146     <NA>        2023-07-28 09:13:09 2023-07-28 09:13:09
    16 A0:E7:0B:AE:D… A0E70B     Intel Corp… 2023-07-28 09:13:09 2023-07-28 11:34:42
    17 00:95:69:E7:7… 009569     LSD Scienc… 2023-07-28 09:13:11 2023-07-28 11:56:07
    18 00:95:69:E7:7… 009569     LSD Scienc… 2023-07-28 09:13:11 2023-07-28 11:56:13
    19 14:13:33:59:9… 141333     AzureWave … 2023-07-28 09:13:12 2023-07-28 10:26:21
    20 10:51:07:FE:7… 105107     Intel Corp… 2023-07-28 09:13:13 2023-07-28 10:25:59
    # ℹ 1 more variable: probes <chr>
```
3.2. Обнаружение реальных (не рандомизированных) MAC-адресов.
```
    # Функция для получения первого октета в десятичном виде
    get_first_octet <- function(mac_str) {
      mac_str %>%
        str_split("[:\\-]", simplify = TRUE) %>%
        .[, 1] %>%
        strtoi(base = 16)
    }
    
    clients_analysis <- clients_enriched %>%
      mutate(
        octet_1      = get_first_octet(mac_addr),
        is_mcast     = bitwAnd(octet_1, 1L) == 1L,
        is_local_bit = bitwAnd(octet_1, 2L) == 2L,
        # MAC рандомизирован, если стоит бит локального администрирования
        # и это не мультикаст
        is_randomized = is_local_bit & !is_mcast
      )
    
    real_macs <- clients_analysis %>%
      filter(!is_randomized)
    
    real_macs %>%
      select(mac_addr, vendor_name, t_start, t_end, probes) %>%
      head(20)
```
```
    # A tibble: 20 × 5
       mac_addr          vendor_name  t_start             t_end               probes
       <chr>             <chr>        <dttm>              <dttm>              <chr> 
     1 5C:3A:45:9E:1A:7B CHONGQING F… 2023-07-28 09:13:03 2023-07-28 11:51:54 C322U…
     2 C0:E4:34:D8:E7:E5 AzureWave T… 2023-07-28 09:13:03 2023-07-28 11:53:16 C322U…
     3 10:51:07:CB:33:E7 Intel Corpo… 2023-07-28 09:13:05 2023-07-28 11:56:06 <NA>  
     4 68:54:5A:40:35:9E Intel Corpo… 2023-07-28 09:13:06 2023-07-28 11:50:50 C322U…
     5 74:4C:A1:70:CE:F7 Liteon Tech… 2023-07-28 09:13:06 2023-07-28 09:20:01 <NA>  
     6 BC:F1:71:D4:DB:04 Intel Corpo… 2023-07-28 09:13:07 2023-07-28 10:57:52 <NA>  
     7 4C:44:5B:14:76:E3 Intel Corpo… 2023-07-28 09:13:09 2023-07-28 09:47:44 <NA>  
     8 A0:E7:0B:AE:D5:44 Intel Corpo… 2023-07-28 09:13:09 2023-07-28 11:34:42 Andro…
     9 00:95:69:E7:7F:35 LSD Science… 2023-07-28 09:13:11 2023-07-28 11:56:07 nvrip…
    10 00:95:69:E7:7C:ED LSD Science… 2023-07-28 09:13:11 2023-07-28 11:56:13 nvrip…
    11 14:13:33:59:9F:AB AzureWave T… 2023-07-28 09:13:12 2023-07-28 10:26:21 <NA>  
    12 10:51:07:FE:77:BB Intel Corpo… 2023-07-28 09:13:13 2023-07-28 10:25:59 <NA>  
    13 10:51:07:CB:33:BF Intel Corpo… 2023-07-28 09:13:13 2023-07-28 11:56:17 <NA>  
    14 BC:F1:71:D5:17:8B Intel Corpo… 2023-07-28 09:13:13 2023-07-28 11:50:47 <NA>  
    15 48:68:4A:93:DF:B4 Intel Corpo… 2023-07-28 09:13:13 2023-07-28 11:50:22 Galax…
    16 28:7F:CF:23:25:53 Intel Corpo… 2023-07-28 09:13:14 2023-07-28 11:51:50 KC    
    17 00:95:69:E7:7D:21 LSD Science… 2023-07-28 09:13:15 2023-07-28 11:56:17 nvrip…
    18 BC:F1:71:D5:0E:53 Intel Corpo… 2023-07-28 09:13:17 2023-07-28 11:50:10 <NA>  
    19 8C:55:4A:DE:F2:38 Intel Corpo… 2023-07-28 09:13:17 2023-07-28 11:56:16 MIREA…
    20 BC:F1:71:D6:10:D7 Intel Corpo… 2023-07-28 09:13:19 2023-07-28 11:50:42 <NA>  
```
3.3. Кластеризация запросов (Probes) и анализ времени присутствия.
```
    # Разворачиваем список SSID из строки
    client_probes_exploded <- clients_enriched %>%
      mutate(
        probes = if_else(
          is.na(probes) | probes == "",
          NA_character_,
          probes
        )
      ) %>%
      separate_rows(probes, sep = ",") %>%
      mutate(probes = str_trim(probes)) %>%
      filter(!is.na(probes), probes != "")
    
    # Группировка
    probe_clusters <- client_probes_exploded %>%
      group_by(mac_addr, probes, vendor_name) %>%
      summarise(
        first_seen = min(t_start, na.rm = TRUE),
        last_seen  = max(t_end,  na.rm = TRUE),
        dur_min    = as.numeric(difftime(last_seen, first_seen, units = "mins")),
        samples    = n(),
        .groups    = "drop"
      )
    
    probe_clusters %>%
      arrange(desc(dur_min)) %>%
      head(20)
```
```
    # A tibble: 20 × 7
       mac_addr   probes vendor_name first_seen          last_seen           dur_min
       <chr>      <chr>  <chr>       <dttm>              <dttm>                <dbl>
     1 00:95:69:… "nvri… LSD Scienc… 2023-07-28 09:13:11 2023-07-28 11:56:13    163.
     2 00:95:69:… "nvri… LSD Scienc… 2023-07-28 09:13:15 2023-07-28 11:56:17    163.
     3 8C:55:4A:… "Gala… Intel Corp… 2023-07-28 09:13:17 2023-07-28 11:56:16    163.
     4 8C:55:4A:… "MIRE… Intel Corp… 2023-07-28 09:13:17 2023-07-28 11:56:16    163.
     5 00:95:69:… "nvri… LSD Scienc… 2023-07-28 09:13:11 2023-07-28 11:56:07    163.
     6 70:66:55:… "MIRE… AzureWave … 2023-07-28 09:14:09 2023-07-28 11:56:21    162.
     7 CA:54:C4:… "GIVC" <NA>        2023-07-28 09:13:06 2023-07-28 11:55:04    162.
     8 F6:4D:98:… "GIVC" <NA>        2023-07-28 09:14:37 2023-07-28 11:55:29    161.
     9 C0:E4:34:… "C322… AzureWave … 2023-07-28 09:13:03 2023-07-28 11:53:16    160.
    10 5C:3A:45:… "C322… CHONGQING … 2023-07-28 09:13:03 2023-07-28 11:51:54    159.
    11 28:7F:CF:… "KC"   Intel Corp… 2023-07-28 09:13:14 2023-07-28 11:51:50    159.
    12 34:E1:2D:… "Cnet" Intel Corp… 2023-07-28 09:13:29 2023-07-28 11:51:50    158.
    13 88:D8:2E:… "POCO… Intel Corp… 2023-07-28 09:13:19 2023-07-28 11:51:24    158.
    14 68:54:5A:… "C322… Intel Corp… 2023-07-28 09:13:06 2023-07-28 11:50:50    158.
    15 68:54:5A:… "Gala… Intel Corp… 2023-07-28 09:13:06 2023-07-28 11:50:50    158.
    16 48:68:4A:… "Gala… Intel Corp… 2023-07-28 09:13:13 2023-07-28 11:50:22    157.
    17 FA:E5:B5:… "\\xA… <NA>        2023-07-28 09:16:55 2023-07-28 11:54:02    157.
    18 FA:E5:B5:… "Дом"  <NA>        2023-07-28 09:16:55 2023-07-28 11:54:02    157.
    19 78:2B:46:… "Redm… Intel Corp… 2023-07-28 09:19:44 2023-07-28 11:53:03    153.
    20 2A:2C:2C:… "GIVC" <NA>        2023-07-28 09:24:41 2023-07-28 11:55:30    151.
    # ℹ 1 more variable: samples <int>
```
3.4. Оценка стабильности сигнала внутри кластера (по SD).
```
    stability_analysis <- client_probes_exploded %>%
      group_by(mac_addr, probes, vendor_name) %>%
      summarise(
        first_seen = min(t_start, na.rm = TRUE),
        last_seen  = max(t_end,  na.rm = TRUE),
        dur_min    = as.numeric(difftime(last_seen, first_seen, units = "mins")),
        pwr_mean   = mean(power, na.rm = TRUE),
        pwr_sd     = sd(power, na.rm = TRUE),
        samples    = n(),
        .groups    = "drop"
      ) %>%
      filter(samples >= 2) # Нужна хотя бы пара измерений для SD
    
    most_stable <- stability_analysis %>%
      arrange(pwr_sd) %>%
      head(20)
    
    most_stable
```
```
    # A tibble: 1 × 9
      mac_addr    probes vendor_name first_seen          last_seen           dur_min
      <chr>       <chr>  <chr>       <dttm>              <dttm>                <dbl>
    1 BA:2B:CE:7… podval <NA>        2023-07-28 10:44:38 2023-07-28 10:44:38       0
    # ℹ 3 more variables: pwr_mean <dbl>, pwr_sd <dbl>, samples <int>
```
Оценка результата
-----------------

Были проанализированы файлы радиомониторинга. Изучены механизмы защиты (OPN/WPA3), определены производители оборудования через OUI-префиксы, выявлены стационарные клиенты без MAC-рандомизации.

Вывод
-----

Закреплены навыки работы с библиотеками `dplyr`, `readr`, `stringr` для парсинга и очистки нестандартных датасетов (смешанная структура AP/Station в одном файле).

