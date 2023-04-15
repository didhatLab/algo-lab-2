# Лабораторная работа по алгоритмам #2

## Задача работы

Реализация и сравнение трех алгоритмов для ответа на зпрос «Скольким прямоугольникам принадлежит точка (x,y)?».
Так же подготовка данных должна занимать мало времени.

## Входные данные

Прямоугольники в виде двух точек для каждого: {(2,2),(6,8)}, {(5,4),(9,10)}, {(4,0),(11,6)}, {(8,2),(12,12)}
И точки для запросов (2, 2), (12, 12)

## Релищации алгоритмов

Каждый алгоритм состоит из двух основных частей: 
1) Подготовка данных(ее нет в алгоритме перебором) - `Prepare()`
2) Сам ответ на запрос - `QueryPoint(point structs.Point)`

### Алгоритм перебора
Идея данного алгоритма очень проста, у нас нет никакой особой подготовки данных, мы просто кладем все прямоугольники 
в массив(в контексте go slice) и потом для поиска итерируемя по всеам прямоугльникам и проверям его принадлежность точке.
 
    func (ba BasicAlgo) QueryPoint(point structs.Point) int {
        answer := 0
    
        for _, rec := range ba.rects {
            if rec.LeftDown.X <= point.X && point.X <= rec.RightTop.X && rec.LeftDown.Y <= point.Y && point.Y <= rec.RightTop.Y {
                answer++
            }
        }
        return answer
    }
    
### Алгоритм на карте
Идея данного алгоритма состоит в том, чтобы сжать координаты по x и y.
После этого мы создаем двумерный массив(это и есть карта), позиции
в котором соответствуют сжатым координатам. После этого мы проходим по всем прямоугольникам, сжимаем их координаты, и в
двумерном массиве на соответствующих промежутках прибавляем количество прямоугольников.

Для обработки Query запроса мы также сжимаем координаты точки и берем соответсвующее значение из двумерного массива.
Это и будет количеством прямоугольников, которым принадлежит точка

Реализация подготовки:    
    
    func (ma *MapAlgo) Prepare() {

        for _, rec := range ma.recs {
            zippedLeft := ma.zipCords.GetZippedPoint(rec.LeftDown)
            zippedRight := ma.zipCords.GetZippedPoint(rec.RightTop)
            for i := zippedLeft.X; i <= zippedRight.X; i++ {
                for j := zippedLeft.Y; j <= zippedRight.Y; j++ {
                    ma.preparedMap[i][j]++
                }
            }
        }

    }


Поиск:

    func (ma *MapAlgo) QueryPoint(point structs.Point) int {
        zippedPoint := ma.zipCords.GetZippedPoint(point)
        if ma.zipCords.IsPointBeyondZippedField(point) {
            return 0
        }
	    return ma.preparedMap[zippedPoint.X][zippedPoint.Y]
    }


### Алгоритм на дереве

Для этого алгоритма используется сжатие координат и построение персистентного дерева отрезков.
Мы создаем массив `roots` в котором представлены деревья для каждого столбца по x или строки по y (зависит от реализации).
Координаты x и y сжаты, как во втором алгоритме. Когда же мы получаем запрос на поиск, то сжимаем координату, берем нужный
root дерева и получаем из него ответ для нашей точки.

#### Подробнее про построение дерева

Деревья мы создаем из так называемых событий, которые создаются из прямоугольника. Каждый прямоугольник это два события.
Первое старт прямоугольника, то есть в дереве отрезков нужно добавить новый прямоугольник на определенном отрезке, второе
это конец прямоугольника, то есть прямоугольник нужно вычесть из дерева отрезков. После создания всех событий, они сортируются.
После этого начинается построение дерева. Новое дерево добавляется в `roots` каждый раз когда в событиях начинается новый столбец/строка.
Новое дерево строится на основе старого, это и есть разные версии деревьев. 

Построение:

    func (pta *PersistentTreeAlgo) Prepare() {
        events := pta.createEventsForPersistentSegTree()
    pta.createPersistentSegmentTree(events)
    }
    
    func (pta *PersistentTreeAlgo) createPersistentSegmentTree(events []structs.Event) {
        root := structs.NewEmptySegTreeNode()
    
        prevZippedX := events[0].ZippedX
        var val int
        for _, ev := range events {
            if ev.ZippedX != prevZippedX {
                pta.roots = append(pta.roots, root)
                pta.rootsZippedX = append(pta.rootsZippedX, prevZippedX)
                prevZippedX = ev.ZippedX
            }
            if ev.IsStart {
                val = 1
            } else {
                val = -1
            }
            root = structs.AddToSegTree(root, 0, pta.zipCords.YSegmentsNumber(), ev.ZippedYStart, ev.ZippedYEnd, val)
        }
    
        pta.roots = append(pta.roots, root)
        pta.rootsZippedX = append(pta.rootsZippedX, prevZippedX)
    }
    
    func (pta *PersistentTreeAlgo) createEventsForPersistentSegTree() []structs.Event {
        events := make([]structs.Event, 0, len(pta.recs)*2)
    
        for _, rec := range pta.recs {
            event1 := structs.NewEvent(
                pta.zipCords.GetZippedX(rec.LeftDown.X),
                true,
                pta.zipCords.GetZippedY(rec.LeftDown.Y),
                pta.zipCords.GetZippedY(rec.RightTop.Y+1))
    
            event2 := structs.NewEvent(
                pta.zipCords.GetZippedX(rec.RightTop.X+1),
                false,
                pta.zipCords.GetZippedY(rec.LeftDown.Y),
                pta.zipCords.GetZippedY(rec.RightTop.Y+1))
            events = append(events, event1, event2)
        }
        sort.Slice(events, func(i, j int) bool {
            return events[i].ZippedX < events[j].ZippedX
        })
    
        return events
    }

#### Подробнее про поиск

При поиске мы сначала сжимаем координаты точки, потом по сжатой точке находим нужный root, то есть столбец к которому
принадлежит точка. После этого проходимся по нужному дереву и находим сумму. У меня в реализации сумма вычисляется на ходу,
но это никак не влияет на асимптотику, зато улучшает производительность.


Поиск:

    func (pta *PersistentTreeAlgo) QueryPoint(point structs.Point) int {
        if pta.zipCords.IsPointBeyondZippedField(point) {
            return 0
        }
        zippedPoint := pta.zipCords.GetZippedPoint(point)
    
        rootForAnswer := pta.roots[findPointPosition(pta.rootsZippedX, zippedPoint.X)]
    
        return structs.GetSum(rootForAnswer, 0, pta.zipCords.YSegmentsNumber(), zippedPoint.Y)

    }

Так же здесь используются методы для SegTree из пакета `src/structs` (structs.GetSum, structs.AddToSegTree). Они являются
больше вспомогательными, поэтому я не стал выносить это отдельно в README.

### Генерация тестовых данных

Генерация довольно простая. Для генерации прямоугольников используется
набор вложенных друг-в-друга с координатами {(10*i, 10*i), (10*(2N-i), 10*(2N-i))}.

Точки генерируются в промежутке, который занимают прямоугольники, поэтому в среднем они распределены равномерно
по всей зоне сгенерированных прямоугольников.

Так же количество сгенерированных точек равно количеству прямоугольников.

### Замеры выполнения

Для замеров времени выполнения использовался встроенный пакет для Golang `testing`, который позволяет делать 
бенчмарк тесты своего кода.

### Графики

Графики строились с помощью `python` с библиотекой `matplotlib`

### Замеры подготовки данных

На графике нет алгоритма перебором, так у него у него нет подготовки данных как таковой

![prepare.png](artefacts%2Fgraphs%2Fprepare.png)

#### Логарифмический график:

![prepare_log.png](artefacts%2Fgraphs%2Fprepare_log.png)

Как видно из графика алгоритм на карте самый медленный в построении, так асимптотика построения карты - O(n**3), так что 
вполне ожидаемо, что при более большом тестовом наборе данных построение занимает намного больше времени, чем два других алгоритма.
На моей машине резкий рост по сравнению с построением дерева начинается после 600 прямоугольников. Карта для построения становится 
слишком большой и заполнять ее становится долго. При больших значениях (50000+), мой компьютер совсем отказывался работать. 
Построения для алгоритма с деревом происходит значительно быстрее, так асимптотика для построения дерева - O(nlog(n)). И на 
всех тестах построение происходит достаточно быстро.

## Поиск(QueryPoint)

![query.png](artefacts%2Fgraphs%2Fquery.png)

#### Логарифмический график:

![query_log.png](artefacts%2Fgraphs%2Fquery_log.png)

Из первого графика видно, что алгоритм перебором обладает линейной сложностью и время запроса растет
линейно, что неудивительно ведь его сложность - O(n). Поиск с помощью карты производится быстрее всего,
так как все значения подсчитаны заранее и необходимо только сжать координату точки и взять значение
из двумерного массива. Запросы с деревом отрезков занимают больше времени, но все равно гораздо быстрее,
чем перебором.


### Вывод

- На небольшом количестве данных(<100 прямоугольников) разницы между алгоритмами почти что нет. Но
все же можно сделать выводы, какой алгоритм использовать для небольшого набора данных. Если прямоугольников,
как и точек совсем немного, то достаточно будет обычного алгоритма перебором (не нужно усложнять, там, где это не нужно).
Если же прямоугольников немного, но точек много, то тогда имеет смысл использовать алгоритм на карте. Сама карта
построится довольно быстро, а потом для каждой точки мы сможем очень быстро давать ответ. Смысла использовать
алгоритм на дереве при небольшом наборе данных особо нет.
- Для большего количества данных ситуация другая. Если нам нужно сделать запрос для одной или двух точек, то 
это можно сделать и обычным перебором, так как это быстрее, чем делать подготовку для двух других алгоритмов. Но если,
количество точек большое, то линейный алгоритм естественно не подойдет, что видно из графиков. Для большего
количества данных необходимо использовать алгоритм на карте или алгоритм на дереве. Для 100-500 прямоугольников
вполне можно взять любой из этих алгоритмов, так как для такого количества прямоугольников, как видно из графика,
время построения сильно не различается. Более того, если точек очень много, то алгоритм на карте будет даже более эффективным.
Но при больших данных (1000+ прямоугольников) время построения карты становится очень большим по сравнению с построением
дерева. И в этом случае уже точно стоит использовать алгоритм на дереве, так как в этом случае он становится гораздо более
эффективным, даже несмотря на немного большее время поиска.

### Решение в контесте

__Логин__: ddsolynin@edu.hse.ru

![ok.png](artefacts%2Fcontest%2Fok.png)

### Артефакты 

Все артефакты работы можно найти в папке `/artefacts`, там можно найти графики, данные по контесту и 
результаты бенчамарк тестов. Сами реализации алгоритмов находятся в `src/algo`, там они находятся
в структурированном виде, с тестами и прочим. Алгоритм на дереве, который отправлялся в контест находится
в `contest/solution.go`, там то же самое, что и в `src/algo`, но помещенное в один файл, чтобы отправить на проверку.

### Дополнительная информация по работе

Информация, которая на прямую не связана с работой (описание структуры, гайд по запуску) находятся в папке `/docs`


