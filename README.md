# [Определение оригинал/кавер для музыкальных треков](https://nbviewer.jupyter.org/github/Nanobelka/Yandex_Music_original_detection/blob/main/YM_original_detection.ipynb)

Решение для хакатона  
Заказчик: **Яндекс.Музыка**

---

### Общее описание задачи

Обнаружение каверов на оригинальные треки — важная продуктовая задача, которая может значительно улучшить качество рекомендаций музыкального сервиса. Если с высокой точностью классифицировать каверы и связывать их между собой в группы, можно предложить пользователю новые возможности для управления потоком треков.  

Например:
- по желанию пользователя можно полностью исключить каверы из рекомендаций;
- показать все каверы на любимый трек пользователя;
- контролировать долю каверов в ленте пользователя.

Возможные направления исследования:
- классифицировать треки на оригинал/каверы;
- группировать каверы и исходный трек в один кластер;
- находить оригинал в кластере.

---

### Решение

#### Результат EDA

В результате исследования данных выяснено:
- нет четкого определения термина "оригинал" (оригиналом необязательно считается наиболее ранний трек, оригиналом может считаться наиболее популярный вариант исполнения);
- несмотря на большое количество данных, доля размеченных данных крайне невелика (для размеченных данных указана ссылка на оригинал).

Принято решение:
- с помощью размеченных данных исследовать возможность определения оригинала в кластере "оригинал + каверы".
- продолжить исследование с целью возможности кластеризации неразмеченных треков вокруг размеченных оригиналов;
- протестировать созданный алгоритм на сформированных кластерах.


#### Результат моделирования

1. Модель помогла выявить ряд ошибок в разметке данных. Например, нередко оригинал имеет более позднюю дату выхода, чем каверы.
2. Принимая во внимание, что оригиналом может считаться не первое, а наиболее популярное исполнение, модель может считаться условно-работоспособной.
3. Если заменить термины "оригинал" на термины "центр" и "окружение", модель может быть **более полезной в реальном применении,** чем для установления факта исторического первенства. Под "центром" подразумевается наиболее значимое исполнение (не обязательно исторически первое).
4. Для повышения стабильности модели необходимо собрать (разметить) дополнительные данные с учетом вышеуказанного изменения в терминологии.
5. После достижения необходимого количества полностью размеченных данных (центр + окружение) можно продолжить исследование, размечая только "центры" и выявляя окружение по дополнительным признакам.

#### Описание принципа моделирования

1. Создание модели для предсказания, какой из двух треков более вероятно является оригиналом.  
    1.1. Отбор размеченных групп. В группе обязательно должен присутствовать оригинал и не менее 1 кавера.  
    1.2. Создание позитивных примеров. Из отобранных данных составлены пары ORIGINAL–COVER, где COVER — это кавер на конкретный ORIGINAL. Таргет для таких пар равен 'ORIGINAL'.  
    1.3. Создание негативных примеров. Все позитивные примеры инвертированы: настоящий оригинал будет выдан за кавер, а кавер — за оригинал. Таргет для таких пар равен 'COVER'.  
    1.4. Обучение модели, которая должна научиться по признакам определять таргет. Поскольку оригиналы будут повторяться в данных (каждому оригиналу может соответствовать более одного кавера), необходимо предпринять **меры против утечки данных**. Каждый оригинальный трек должен попасть либо в обучающую, либо в валидационную (тестовую) выборку.  
2. Тестирование модели на размеченных данных.
    2.1. Для тестирования будут использованы размеченные данные из валидационной выборки. Модель видела эти данные только при контроле переобучения: не идеально, но для создания отдельной тестовой выборки крайне мало данных.
    2.2. Для каждого кластера создаются все возможные пары треков (cross-tab).  
    2.3. С помощью созданной модели для каждой составленной пары оценивается вероятность каждого из треков пары быть "более оригинальным" (в сумме 1).  
    2.4. Для каждого трека рассчитывается его среднее значение вероятностей быть оригинальным, по всем парам, где встречается этот трек. Это будет оценкой для ранжирования треков.  
    2.5. Треки ранжируются по рассчитанной выше оценке. Наибольшую оценку имееет предполагаемый оригинал.  
    2.6. Нужно отметить, что модель может дать низкую оценку треку, помеченному в исходных данных как оригинал, ориентируясь на его признаки. Например, такой размеченный оригинал имеет более позднюю дату выхода, чем его каверы.  
3. Кластеризация (пока в стадии экспериментов).  
    3.1. Из текстов треков получены эмбединги.  
    3.2. Создан индекс FAISS, в который загружены все эмбединги. Мможно рассмотреть и другие алгоритмы кластеризации.  
    3.3. На основе размеченных данных предполагается оценить порог расстояния между треками, для последующей кластеризации треков по текстам.  
4. Запуск на неразмеченных данных (еще не реализован).  
    4.1. Для любого трека определить кластер похожих треков, исходя из установленного порога схожести текстов.  
    4.2. Для кластеризованных данных применить созданную модель, чтобы ранжировать кластер по убыванию оценки быть оригиналом.  
