#1. Введение
Replication is a fundamental concept of distributed systems, well studied by the distributed algorithms community. Much work focuses on maintaining a global total order of operations [24] even in the presence of faults [8]. However, the associated serialisation bottleneck negatively impacts performance and scalability, while the CAP theorem [13] imposes a tradeoff between consistency and partition-tolerance.

An alternative approach, eventual consistency or optimistic replication, is attractive to practioners [37, 41]. A replica may execute an operation without synchronising a priori with other replicas. The operation is sent asynchronously to other replicas; every replica eventually applies all updates, possibly in different orders. A background consensus algorithm reconciles any conflicting updates [4, 40]. This approach ensures that data remains available despite network partitions. It performs well (as the consensus bottleneck has been moved off the critical path), and the weaker consistency is considered acceptable for some classes of applications. However, reconciliation is generally complex. There is little theoretical guidance on how to design a correct optimistic system, and ad-hoc approaches have proven brittle and error-prone.

In this paper, we study a simple, theoretically sound approach to eventual consistency. We propose the concept of a convergent or commutative replicated data type (CRDT), for which some simple mathematical properties ensure eventual consistency. A trivial example of a CRDT is a replicated counter, which converges because the increment and decrement operations commute (assuming no overflow). Provably, replicas of any CRDT converge to a common state that is equivalent to some correct sequential execution. As a CRDT requires no synchronisation, an update executes immediately, unaffected by network latency, faults, or disconnection. It is extremely scalable and is fault-tolerant, and does not require much mechanism. Application areas may include computation in delay-tolerant networks, latency tolerance in wide-area networks, disconnected operation, churn-tolerant peer-to-peer computing, data aggregation, and partition-tolerant cloud computing.

Since, by design, a CRDT does not use consensus, the approach has strong limitations; nonetheless, some interesting and non-trivial CRDTs are known to exist. For instance, we previously published Treedoc, a sequence CRDT designed for co-operative text editing [32].

Previously, only a handful of CRDTs were known. The objective of this paper is to push the envelope, studying the principles of CRDTs, and presenting a comprehensive portfolio of useful CRDT designs, including variations on registers, counters, sets, graphs, and sequences. We expect them to be of interest to practitioners and theoreticians alike.

Some of our designs suffer from unbounded growth; collecting the garbage requires a weak form of synchronisation [25]. However, its liveness is not essential, as it is an optimisation, off the critical path, and not in the public interface. In the future, we plan to extend the approach to data types where common-case, time-critical operations are commutative, and rare operations require synchronisation but can be delayed to periods when the network is well connected. This concurs with Brewer’s suggestion for side-stepping the CAP impossibility [6]. It is also similar to the shopping cart design of Alvaro et al. [1], where updates commute, but check-out requires coordination. However, this extension is out of the scope of the present study.

In the literature, the preferred consistency criterion is linearisability [18]. However, linearisability requires consensus in general. Therefore, we settle for the much weaker quiescent consistency [17, Section 3.3]. One challenge is to minimise “anomalies,” i.e., states that would not be observed in a sequential execution. Note also that CRDTs are weaker than non-blocking constructs, which are generally based on a hardware consensus primitive [17]. 

Some of the ideas presented here paper are already known in the folklore. The contributions of this paper include:
* In Section 2: (i) An specification language suited to asynchronous replication. (ii) A
formalisation of state-based and operation-based replication. (iii) Two sufficient conditions
for eventual consistency.
* In Section 3, an comprehensive collection of useful data type designs, starting with
counters and registers. We focus on container types (sets and maps) supporting both
add and remove operations with clean semantics, and more complex derived types,
such as graphs, monotonic DAGs, and sequence.
* In Section 4, a study of the problem of garbage-collecting meta-data.
* In Section 5, exercising some of our CRDTs in a practical example, the shopping cart.
* A comparison with previous work, in Section 6.
Section 7 concludes with a summary of lessons learned, and perspectives for future work.

#2. Общие положения и модель системы
Мы рассматриваем распределённую систему состояющую из процессов, соединённых в сеть с асинхронной передачей сообщений. Сеть может распадаться и восстанавливаться, узлы сети могут какое-то время работать отсединёнными от сети. Процессы могут аварийно завершаться и затем восстаналиваться, память процессов устойчива к аварийным завершениям. Так же подразумеваем "невизантийское" поведение, т.е. без процессов стремящихся наруть работу системы.

##2.1. Атомы и объекты
Процесс может хранить атомы и объекты. Атом -- бызовый немутируемый тип данных, идетифицируемый своим содержимым. Атомы могу быть скопированы из одного процесса в другой, атомы равны, если имеют одинковое содержимое. Типы атом, рассмотриваемые в этой статье, включают в себя числа, строки, множества, кортежи и.т.д. с их обычными немутирующими операциями. Название типа атома мы обозначаем словом в нижнем регистре, например set.

Объект -- мутируемый реплицируемый тип даннх. Название типа обекта будем писать с первой заглавной буквы (например, Set). Объект имеет идентификатор, содержимое (payload), который может состояить из произвольного количества атомов или объектов, начальное состояние, и интерфейс состоящий из набора операций. Два объекта с одним идентификатором, но которые находятся на разных репликах называются репликами друг-друга. Например на рис. 1, изображен логический объект x, его реплики в процессах 1, 2 и 3 и текущее состояние реплики 3.

Мы подрузумеваем, что объекты независимы и не рассматриваем транзакции. Поэтому без потери общности мы будем рассмотривать однин объект и будем использовать слова процесс и реплика как одно и тоже.

##2.2 Операции
Окружении состоит и неизвестных клиентов, которые запрашиваюит и меняют состояние объекта посредством вызова операций его интерфейса, выбирая реплики по своему усмотрению (которую будетм называть исходной репликой). Запрос исполняется локально, т.е. полностью на одной из реплик. Изменение объекта состоить из двух фаз: 1-я, клиент вызывает операцию на исходной реплике, которая может выполнить какую то начальную обработку запроса. Потом операция изменения передаётся асинхронно другим репликами, это называется downstream-фаза. В литературе [37] выделяются state-based и operation-based стили, которые мы рассмотрим далее.


| Спецификация 1. Схема state-based спецификации объекта. Преусловия, аргументы, возвращаемые значения и выражения. |
|-----|
```
payload: Тип соедржимого, которые имеется на всех репликах
query: Query(args): returns
    pre: пре-условия
    let: выполняется на реплике источнике синхронно, без побочных эффектов
update: Source-local operation (args): returns
    pre: пре-условия
    let: выполняется  на реплике источнике синхронно, 
        Побочные эффекты на источнике исполняются синхронно
compare (value1, value2): boolean b
    is values1 <= value2 в полурешетке.
merge (value1, value2): payload mergedValue
    LUB слияние valu2 и value2, на любой реплике
```

###2.2.1 State-based репликация
При state-based (или пассивной репликации) обновления возникают целиком на источнике, потом распространяются передачей изменённого контента между репликами (рис 4).
Мы специицируем state-based объект-типы как показано на Спец. 1. Payload означает - тип полезных данных, initial value -- начальное значение на каждой реплике. Update -- операции изменения, query -- запрос. Оба могут иметь аргументы и возвращаемое значение. Не мутирующие выражения помечены let, palyload меняется оператором :=. Операции выполняются атомарно.
Для собрадения безопастнсти опарции воможны только если соблюдены пре-условия (помеченные pre) хранимые в текущем состоянии источника. Пре-условия опускаются если действую постоянно, например увеличение или уменьшение счётчика. И наобороь, non-null персловие могут быть, например элемент множества может быть удалён только если он есть во множестве.
Система передаёт состояние между произвольными парами реплик, чтобы распространицть изменения. Это изменяет payload получтеля результатом операции merge, вызываемую с двумя аргументами, локальным payload и полученным payload. Операция compare сравнивает состояния реплик, о чём мы скоро расскажем.
Мы определяем каузальную историю С репли какого-нибудь объекта как:
О1:
для любой реплики x_i объекта x:
- C(x_i) = /0
- После выполнения update-операции f, C(f(x_i)) = C(x_i) _U_ {f}
- после выполнения merge на x_i, x_j, C(merge(x_i, x_j)) = C(x_i) _U_ C(x_j)
классическое опеределния happens-before [24], отноение может быть определено как f -> g <=> С(f) _c_ C(g).
Живучесть требует, чтобы любое обновление достигло всех каузальной истории всех реплик. Для этого эффекта, мы предполагаем, что система передаёт сообщения между парами реплик за неизвестное время, возможно бесконечное, и что реплики соединены в связанный граф.

2.2.2 Op-based объекты.
В op-based (или активной) репликации сисема, передёт опрации как показано на рис.6. Этот стиль показн на Спец.2. payload и initial - такие же ка кв state based объектах. Операции которые не мутируют объект помечены query и выполняются полность на одной реплике. 
Операции обновления обозначаются словом update. Первая фаза, помеченная atSource, локальная к реплике источнику. Она активна только если пре-условия, помеченные pre, истино на локальной реплике, выполняется атомарно. Оно принимает аргументы из вызова операции, запрещено создавать побочные эффекты, можно вычилять результат, возвращаемый вызывателю, и/или подготовливать аргументы для второй фазы.

Спецификация 2. Общий план спицификаии op-based типа.
payload:
    initial
query 
    pre
    let
update
    atSource
        pre
        let
    downstream
        pre

Вторая фаза, помеченная downstream, выполяется после источник-локальной фазы, немедленно на источнике и асинхронно на других репликах, не может возвращать результат. Выполняется только если downstream pre верно. Оно обновляет состояние  downstream-а, его аргументы подготовливаются source-local фазой. Выполняется атомарно.
Как и выше определим каузальную историю релики C(x_i)
О2.2 (Каузальная история op based).
- C(x_i) = /o
- После выполнения downstream фазы операции f на реплике x_i, C(f(x_i)) = C(x_i) _U_ {f}
Живучесть труебст, чтобы каждый апдей быть eventually доставлен на каузальную испторию каждой реплики. Для этого эффекта мы подразумеваем, что система надежно досталяет каждый объект к каждой реплике в порядки <_d (называемом порядок доставки) где преусловия downstream верны. 
Как и в случае state-based happens-before отношение определяется как f -> g <=> C(f) _c_ C(g), Мы определяем каузальную досавку, <_-> как: если f -> g тогда f доставлено до доставки g. Отметим, что все downstream преусловия в этой статье удовлетворяюют каузальной доставки. например, delivery-порядок -- тоже самое или слабже каузального порядки. f <_d g => f <_-> g.

2.3 Сходимость
Формализуем сходимость
О2.3 Сходимость. Две реплики x_i и x_j объекта x сходяться отложенно если выполненые слею условия:
- Безопасность: ForAny i, j: C(x_i) = C(x_j) подразумевает что абстрактные состояние i и j эквиваленты
- Живучесть: ForAny i, j: f _C-_ C(x_i) подразумевает что отложенно f _C-_ C(x_j)
Более того, мы определим состояние эквивалентность как x_i и x_j имеют эквивалентное абстрактное состояни, если запросы, возвращают одно и тоже состояние.
Попарная отложенная сходимость подразумевает что любое непустое подмножество реплик обхект сходится как тольок все реплики получает все обновления.
2.3.1 State-based CRDT, сходимые реплицируемые типы данных. (CvRDT)
Верхняя полурештка (или далее просто полурешетка) и частиный порядок <=_v вместе с точной верхней гранью (LUB) |_|_u определяемая как
О2.4 (Точная верхня грань(ТВГ)) m = x |_|_u y это ТВГ {x, y} под <=_v тогда и только тогда x <=_v m и y <=_v m и нет такого m' <=_v m, что x <=_v m' и y <=_v m'
Это следует из определения что LUB_v это: коммутативная x LUB_v y = y LUB_v x, идемпотентная x LUB_v x = x и ассоциативная (x LUB y) LUB z = x LUB (y LUB z)

О2.5 (Верхня полурешетка) Упорядоченное множество (S, <=_v) это полурешетка титт ForAny x,y из S, x LUB y существует

State-based обхект чби данные берут значения из полурешеки и где merge(x,y) = x LUB y сходяится к LUB начального и обновленного значения. Если более того, обновления монотонно двигаются вверх соотв. <=_v (например, данные после обновления больши или равне значению до), тогда они сходятся к LUB последних значений. Давайте назовём это монотонной полурешеткой.
Тип данных с этими свойствами называется CRDT или CvRDT, Мы требуем, в CvRDT, compare(x, y) возвращал x <=_v y, чтобы абстрактные состояни были эувивалентными если x <=_v y & y <=_v x и merge все был определён. Например, рис. 5 иллюстрирет CvRDT с int-ом где <=_v это порядок числе и где merge = max.
Отложенная сходимость требует чтобы все реплики получили обновления. Каналы связи в CvRDT могут иметь довольно слабые свойства. Если merge идемпотентный и коммутативный (в соотв. со свойствами LUB) сообщения могут быть потеряны или получены в разном порядки или много раз, но если новое состояние достигает всех реплик, напряму или нет через успешный merge. Обновления распространяются надёжно даже если нет распаласть на частили и затем через какое-то времия восстановилась.
Утв.2.1. Любый две реплики объекта CvRDT отложенно сходятся, предполагаю что система передёт данные часто между парами реплик через отложенно-надежный point-to-point каналы.
Док-во. ...

2.3.2. Op-based CRDT: Коммутативные реплицируемые типы данных. (CmRDT)
Для op-based объектов, надежное вещательный канал гаратирует, что все обновления достигнут каждой реплики, в delivery order - <_d определяемом типом данных. Операции, которые не построены в порядке <_d называеются конкурентными, форманого f ||_d g <=> f !< _d g & g !< _d f. Если все конкурентные операции commute,  тогда все порядки выполнения консистены с delivery order-ом эквивалентны и все реплики сходятся к одному состоянию. Такой объект называется CmRDT.
Как сказано ранее, для всех типов данных в этой статье каузальная доставка <_-> (которая легко реализуется и статических рапределённых системах и не требует консенсуса) удовлетворяет delivery order-у. Для некоторых типов, слабый порядок достаточен, но тогда большее колво пар операций требуют коммутативности.
О2.6. Коммутативность. Операции f и g коммутативны, титт когда для любой достиимой реплики состояния S где их преусловия источника выполняется, преусловия источника f (resp. g) остаётся выполненым в состоянии S*g (resp. S*f) и S*f*g являются эквивалентными состояними.
Утв. 2.2 Две любые реплики CmRDT отложенно сходятся на надежном вещательном канале, которые доставляет операции в порядке <_d.

Док.во. ...

Вспомнис, что надёжная каузальная доставка не требует подтверждения. Она не боится разделений, главное чтобы соединённыое подмножедество могло доставить другие обновления и обновления отложенно доставились ко всем репликам. И посколько delivery order никогда не ограничен каузальной доставкорй, это верно для всех CmRDTs.

2.4. Связь двух подходов.
Мы показали два подхода к отложенной сходимости CvRDT и CmRDT, которые вместе называются CRDT. Если несколько сходств и отличий между ними.
Рассуждать о CvRDTs проще, т.к. все необходимая информация находится в состоянии. Они требуеют слабыл допущений о канале связи, допуская неизвестное число реплик. Однако отпарка состояни может быть неудобной для больших объектов; что можно порешать доставляя дельты, но это требует механизма соотв. op-based подходу. Исторически, state-based подход используется  в файловых системах таких как NFS, AFS, Code и key-value сторах типа Dinamo или Riak.
Определения op-based объхектоы может быть более сложно, т.к. требует рассуждений о истории, но наобборот имеет большую выразительную силу. Payload может быть проще, ибо часть состояния фактически выгружено в канал. Op-based репликация более требовательно к каналу, т.к. требует надёжного вещания, которая в общем требует учёта членства в группе. ИсторическиЮ op-based подходы используется в кооперативных системах типа, Bayou, Rover [21] IceCube [33], Telex [4].

2.4.1 Op-based Эмуляция state-based объекта
Интересно, что всегда можно эмулировать state-based объект через op-based подход и наоборот.
В спец.3 мы показываем,  op-based эмуляцию state-based объекта (довольно свободно в нотации). Игнорируя запросы, эмуляция op-based обхекта имеет один апдейтЮ который вычисляет некоторовый state-based апдейт (после проверки пре-условий). и выполняет merge downstream-а. Пре условие downstream-а пустое ибо merge должен быть доступен при любых достижимых состояниях. Эмуляция не использует comapre
------
Спецификация3 Эмуляция State-based через Op-based.
-----
payload: State based S
    initial: initial payload
update State based update (операция f, args a): state s
    atSource: (f, a): s
        pre: S.f.precondition(a)
        let s = S.f(a)
    downstream (s)
        S : = merge (S, s)

2.4.2 Эмуляция Op-based через State-based.
State-based эмуляция op-base Объекта формалицет механика широкого вещания, как показано на спец.4. Снова, мы игнорирует запросы, которые не вызывают проблем. Вызов op-based обновления добавлет их к множеству M сообщений для доставки, merge берёт обхекдинение двух наборов сообщений.

----
Спецификация 4. State-based эмуляция op-based объекта
-----
payload: Operation based P, множество M, множество D (данные эмулируемого объекта, сообщения, доставлено)
    intial: initial p, /0, /0
update: op-based update(update f, args 1): return 
    pre: P.f.atSource.pre(a)
    let returns = P.f.atSource(a)
    let u = unique();
    M := M _U_ {(f, a, u)}
    deliver();
update deliver()
    for (f, a, u) in (M - D): f.downstream.pre(a) do
        P := P.f.downstream(a)
        D := D _U_ {(f, a, u)}
compare (R, R') : boolean
    let b = R.M <= R'.M & R.D <= R'.D
merge (R, R'): payload R''
    let R''.M = R.M _U_ R'.M
    R''.deliver();

Когда precondition downstream update-а верен, соотв. сообщение доставляется посредством ваполнения downstream части соотв. апдейта. Чтобы избежать дублирования доставок, доставленные сообщения сохраняются во множество D.
Отметим, что состояние эмуляции обхекта формирует монотонную полурешетку. Вызов или доставка операции добавляет её к соотв. множеству сообщений и затем изменяет состояние соотв. частином порядке. Merge определен, чтобы обхекнить мноржества M и таким образом это LUB операция. Отметим, что M идентично каузальной истории репли, не конкрентные огбновления возникают в M в каузальном порядке. Если эмулируемый op-based объект это CmRDT, тогда delivery order соблюден. Конкурентные операции возникают в M в любом порядки, и если эфулируемый объект CmRDT, тогда они компенсируются. Таким образом, после merge двух реплик, их множества D идентичны и P жфквивалентно.

3. Портфолио базовых CRDT.
Чтобы показать полезности концепции CRDT, мы покажем набор проектов CRDT. Они интеренсть для понимая вызовов, возможность и ограничений CRDT. Также они составляют библиотеку типов, которыю могут быть переиспользованы и скомбинированы для построеня распределённых систем. Мы начнём с простых типов (счётчики и регистры), затем переёдём к коллекциям типов (множествам) , и наконец к типам с более сложными требованиями (графы, DAG-и и последовательности).
Наши спецификации написаны с ясностью для понимания, не эффективности. Во многих случаял они эквиваленты подходам который экономят место, но мы в большистве предпочитаем болле простые для понимания версии.
Мы используем как state- так и op- based спецификации, как удобнее. Для каждого state-based примера, мы обязуемся докатать, что их состояния образуют полурешеку и что merge вычиляет LUB. Для каждого op-based примера, мы должны показать, что delivery order существует и что конкурентные обновления компенсируеются.

3.1. Счётчики.

Счётчики это перлицируемые числа, поддерживающие операции инкремента и декремента для обновления, value для получения их значения. Семантика такова что значение сходится к числу инкрементов минус кол-во декрементов (расширяется до операций прибавления и вычитания как аргмунт).  А CRDT-счётчики полезны во многих peer-to-peer приложениях, например для посчёт кол-ва залогиненых пользователей. 
В этом разделе мы обсудим разные варианты для реализации CRDT-счётчиков. Несмотря на простоту, каунтеры выявляют некоторые нюансы дизайна CRDT.

3.1.1 Op-based счётчик
Op-based счётчки представена в спецификации 5. Его payload -- число. Пустое atSource выражение опущено; downstream-фаза просто добавляет или убавляет локаотно. Это хорошо извествно, что удаление и добавление переставляются, предполагаю отсуствие переполенения. В этом случае этот тип данных является CmRDT.

----
Specification 5 op-based Counter
-----
payload integer i
    initial 0
query value () : integer j
    let j = i
update increment ()
    downstream ()
        i := i + 1
update decrement ()
    downstream ()
        i := i − 1

3.1.2. State-based только инкрементируемый счётчик (G-Counter)
State-based счётчик не насколько простой как могло показаться. ДЛя упрощения проблемы начнём со счётчика только с инкрементом.
Предположи, что payload это одно число и merge вычиляет max. Этот тип данных -- CvRDT, .к. его состояния -- мономорфная полурешетка. Рассмотрим две реплики, с один и темже начальным состоянием 0, на каждой клиент вызывает increment. Они сойдутся к 1, а не к двум как хотелось бы.
Теперь допустим мы будем складывать два значения payload-а. Это не CvRDT, т.к. merge не идемпотентен. 

---
Specification 6 State-based increment-only counter (vector version)
----
payload integer[n] P (n - кол-во реплик)
    initial [0, 0, . . . , 0]
update increment ()
    let g = myID()
    P[g] := P[g] + 1
query value () : integer v
let v = P.sum()
compare (X, Y) : boolean b
    let b = ForAny i X.P[i] <= Y.P[i]
merge (X, Y) : payload Z
    Z.P[i] = max(X.P[i], Y.P[i])


Мы предложим теперь конструкцию на спец. 6 (вдохновленную векторными часами). Payload - это верктор чисел, каждая реплика назначено значение. Для увеличения добавим один в ячейку соотв. реплике. Значение это сумма всем ячеерк. Определим частичный порядок, на двумя состояними как ForAny i X.P[i] <= Y.P[i], где n - число реплик. Merge берёт максимальное значение каждой ячейки. Этот тип данным -- CvRDT, т.к. состояни образую мономорфную полурешетку и merge вызысляет LUB.
Эта версия делает для важных предположения: payload не переполняется, и множество реплик известно. Отметим, что op-based версия неявно делает эти два преположения. 
Напротив, G-Set (будет описан далее) может предоставить только increment-only счётчик. G-Set работает даже когда набор реплик неизвестен.
Положительный счётчик полезен например для подсчёта кол-ва кликов на ссылке и P2P-реплицируемой web-странице, или в P2P Like,DontLike кнопке, в соц-сетях.

3.1.3 State based PN-counter.
Не так просто обеспечить операцию декремента, ибо операция будент нарушать монотонность полурешетки. Кроме того, т.к. merge = max, декременты не будут иметь эффекта.
Наше решение, PN-счётчик (Спец. 7) основана на двух G-counter-ах. Its payload consists of two vectors: P to register increments, and N for decrements. Its value is the difference between the two corresponding G-Counters, its partial order is the conjunction of the corresponding partial orders, and merge merges the two vectors. Proving that this is a CRDT is left to the reader.
Such a counter might be useful, for instance, to count the number of users logged in to a P2P application such as Skype. To avoid excessively large vectors, only super-peers would replicate the counter. Due to asynchrony, the count may diverge temporarily from its true value, but it will eventually be exact.

3.1.4. Неотричатльный счётчик
Некоторый приложение требуют неотрицатльных счётчиков, например, для подлсчёта оставшегося кредита аватара в P2P играх.
Однако, довольно трудно 

However, this is quite difficult to do while preserving the CRDT properties; indeed, this is a global invariant, which cannot be evaluated based on local information only. For instance, it is not sufficient for each replica to refrain from decrementing when its local value is 0: for instance, two replicas at value 1 might still concurrently decrement, and the value converges to −1.

One possible approach would be to maintain any value internally, but to externalize negative ones as 0. However this is flawed, since incrementing from an internal value of, say, −1, has no effect; this violates the semantics required in Section 3.1.

A correct approach is to enforce a local invariant that implies the global invariant: e.g., rule that a client may not originate more decrements than it originated increments (i.e., ∀g : P[g] − N[g] ≥ 0). However, this may be too strong.

Note that one of the Set constructs (described later, Section 3.3) might serve as a nonnegative counter, using add to increment and remove to decrement. However this does not have the expected semantics: if two replicas concurrently remove the same element, the result is equivalent to a single decrement.

Sadly, the remaining alternative is to synchronise. This might be only occasionally, e.g., by reserving in advance the right to originate a given number of decrements, as in escrow transactions [28].