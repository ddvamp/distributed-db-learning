## Lecture 2. Линеаризуемость. Репликация регистра, алгоритм ABD

#### Изучаем KV Store

В этой лекции начнём проектировать KV Store со следующим API
- $Set \left \lparen key, value \right \rparen$
- $Get \left \lparen key \right \rparen$
- (диапазоны, транзакции и другие операции в последующих лекциях)

и продолжим это делать на протяжении большей части курса

Почему этот класс распределённых систем? Первой хотели сделать распределённую базу данных, но это оказалось слишком сложно. Из неё выкинули фичи (транзакции, запросы, таблицы), и оставшуюся часть ($key \to value$) удалось сделать распределённой и причём хорошо. Уже после поверх отказоустойчивого и распределённого слоя KV реализовали привычный интерфейс базы данных, получив т.н. **SQL over KV**. К тому же доступные документы о `BigTable` и `Amazon DynamoDB` с интерфейсом KV были опубликованы одними из первых

#### Слои KV Store

KV должен быть масштабируемым и отказоустойчивым, поэтому в нём много слоёв
- **1 уровень - Local Storage**: построим в пределах одного узла эффективную систему с произвольным доступом (API) поверх последовательного диска (Пример: `LevelDB`, `RocksDB`). Полагаем, что данные помещаются на один узел, диск используем на случай рестартов (Durability), а отказов не существует. Вопросы больших данных и отказов решаем на следующих уровнях
- **2 уровень - Replication**: организуем отказоустойчивость на случай смерти диска/узла для небольшого объёма данных. Реплицируем данные одного узла на $3-5$ репликах (**replica set**). *Реплика*, потому что одинаковые. Если необходимо реплицировать неизменяемые данные, то можно применить *erasure codes* и сократить количество реплик в два раза без потери надёжности
- **3 уровень - Distribution**: когда данные более не помещаются на одной машине, необходимо распределить их по кластеру (*шардировать*). Имеем много узлов и большой диапазон ключей, *Key space* |-||-||-||-|, где |-| назовём *range (или регион)* $\left \lbrack b, d \right \rbrack$, где $b$ и $d$ ключи. Ключи в пределах range упорядочены. Каждый range реплицируется независимо на некотором replica set. Replica set разных range могут пересекаться, но в целом произвольны. Так каждый range будто самостоятельный KV Store
- **4 уровень (последний для целей изучения) - Transactions**: для более сложных операций, чем запись одного ключа. С их помощью реализуются транзакции в SQL (для распределённых баз данных). На этом уровне заканчивается KV Store
- **5 уровень (не входит в KV Store, часть базы данных) - Queries**: распределённое выполнение запросов и табличная модель SQL поверх них

#### Репликация регистра

Решаем задачу репликации одной ячейки памяти - регистра (задача моделирования разделяемой памяти в распределённой системе). После реализации Local Storage решение можно будет расширить на весь диск. Задачу будем решать в слещующей модели: асинхронная, неизвестны скорость и порядок доставки сообщений, неизвестна скорость работы узлов, неизвестна скорость дрейфа часов, узлы перезагружаются или отказывают навсегда, нет повреждений памяти (есть коррекция), нет византийских отказов

Операции регистра
- $Write \left \lparen v \right \rparen$
- $Read \left \lparen \right \rparen$

Обобщим задачу синхронизации памяти в процессоре, поскольку он является распределённой системой, хоть и крайне безопасной. Здесь же всё сложнее из-за отказов и ограничений (скорости) сети: на разных узлах в произвольные моменты времени значения ячейки будут отличаться. Система должна скрывать это от пользователя и предоставлять ему набор гарантий, например видимость своей же записи или видимость чужой записи по причинности (happens-before по внешнему каналу)

#### Последовательная и конкурентная истории

**Конкурентная история** - глобальная история во времени конкурентных действий (чтений/записей) с регистром многих пользователей. Операции не атомарны, и в истории визуально отображены их начало и конец, а также результат. Начало означает инициацию операции пользователем, а конец - получение пользователем ответа. При этом не гарантировано, что в момент конца записи в регистре находится записанное значение - оно уже может быть перезаписано. Операции либо одна заканчивается раньше начала другой, и говорят, что они состоят в отношении до/после (**упорядочены во времени, Real-time ordered**), либо пересекаются, и называются **конкурентными**. Первые, это либо последовательные операции одного клиента, либо операции, упорядоченные по happens-before (внешний канал связи). Вторые, все остальные операции. Возможные конкурентные истории порождаются конкретной реализацией регистра с определёнными гарантиями (моделью согласованности) в процессе работы с ним, и для пересекающихся операций можно считать, что они выполняются в произвольном порядке, т.е. мы вольны предположить тот порядок, который соответствует ожидаемым от реализации гарантиям (модели согласованности), и напротив, если ни один из порядков не подходит, реализация, в рамках ожидаемой модели, некорректна

**Последовательная история** - история последовательных действий с регистром, которые не пересекаются во времени, будто все действия запрашиваются одним пользователем в одном потоке и выполняются мгновенно. Пример $w \left \lparen 1 \right \rparen w \left \lparen 2 \right \rparen r \to 2 \hspace{0.5em} w \left \lparen 3 \right \rparen r \to 3$. Сама по себе последовательная история не может быть правильной или нет, например в случае $w \left \lparen 42 \right \rparen r \to 41$, только при рассмотрении в рамках какой-либо модели согласованности. Последовательность действий с регистром каждого отдельного клиента является последовательной историей, которая обязана согласовываться с конкурентной историей

**Спецификация (Spec) регистра** - набор допустимых последовательных историй (или наблюдаемых поведений). Спецификация описывает, как может вести себя регистр (какие последовательности записей/чтений возможны), если к регистру нет конкурентных обращений. Спецификацию задают при проектировании регистра, например *Atomic* или *Regular*

**Consistency Model (модель согласованности)** - отвечает на вопрос: какие (корректные) конкурентные истории может порождать реализация с заданной спецификацией

Объединяя всё вместе, спецификация говорит, как ведёт себя регистр в идеальной среде (один пользователь, синхронные операции, нет сбоев), а модель согласованности говорит, как регистр с такой спецификацией будет выглядеть в реальной среде (с конкурентным доступом и сбоями). Т.о. спецификация и модель независимы, но обе влияют на поведение

Случаи нарушения гарантий/модели реализацией регистра
- нарушение внешней причинности ($hb$), когда модель требует обратного
- невозможность упорядочить конкурентные операции в конкурентной истории
- невозможность последовательной истории клиента по модели
- несогласованность последовательной истории клиента и конкурентной
- несогласованность последовательных историй клиентов в совокупности, когда модель требует обратного

#### Линеаризуемость

Модель согласованности, которую хотим от регистра, это **Linearizability** (**линеаризуемость, External Consistency** - намёк на внешнюю коммуникацию) - модель, в которой для любой конкурентной истории операций с регистром существует допустимая по его спецификации последовательная история, называемая *Линеаризацией*, которая построена из тех же операций с теми же результатами, и которая уважает Real-time ordering любой пары упорядоченных операций $o_1$ и $o_2$

Линеаризуемость не предъявляет требований к пересекающимся во времени операциям, поэтому если пользователь сам не упорядочил операции по Real-time ordering, то он не должен ожидать от них какого-либо порядка в линеаризации. Линеаризуемость является более общей моделью, чем Sequential consistency, т.е. даёт больше гарантий. Одновременно может существовать несколько линеаризаций для одной истории, что не является проблемой

**Регистр линеаризуем (atomic register)**, если его реализация порождает только линеаризуемые истории. Для пользователя это означает, что он может думать о системе как о не распределённой, будто она действует атомарно, операции случаются мгновенно одна за одной, и конкуренции нет

Линеаризация по сути единый логический порядок операций, которым клиенты могут объяснить изменения в истории системы, как если бы она не была распределённой и конкурентной, а операции выполнялись мгновенно. При линеаризуемости история изменений, которую видит любой клиент, является подмножеством линеаризации. Если же видение клиентами системы противоречиво, линеаризуемость невозможна

Грубые объяснения линеаризуемости
- любая операция системы выполняется под мьютексом
- любая операция произошла в точке на её отрезке в конкурентной истории, и линеаризация суть множество этих точек
- в графе порядков отсутствуют циклы

#### Каким образом линеаризуемость уважает Real-time ordering

Ни система, ни клиенты не могут использовать $rt$, поскольку доступа ко времени нет, а часы не идеальны и не синхронизируемы. Система выставляет операциям монотонные timestamp-ы, которые увеличиваются со временем, $ts \implies rt$. Клиенты в свою очередь думают о **причинности (happens-before)** (порядок локальных действий, реакции на внешние сообщения), $hb \implies rt$ (от противного, действие не может стать причиной события в прошлом), и ожидают, что система это учтёт. И хотя система не знает о $hb$, а клиенты не знают о $ts$ (в некоторых моделях знают), два эти порядка согласованы через общее время $rt$, и следовательно система учитывает $hb$. Так мы без времени знаем какие события следуют до каких

#### Задача репликации регистра

Atomic Register
- $Write \left \lparen v, ts \right \rparen $, $ts$ - timestamp
- $Read$

Операции блокирующие, Asynchronous model, Crash/Restart

Отказоустойчивость (или допустимое число отказов) зависит от размера replica set. Пока неизвестно как, поэтому интуитивно возьмём три реплики (затем алгоритм можно будет обобщить). Дополнительные упрощения
- конфигурация статична - не умеем заменять отказавшие узлы, добавлять новые. Устраняется в [лекции 8](https://github.com/ddvamp/distributed-db-learning/blob/main/notes/dist-sys-mipt/lectures/lecture-8.md#реконфигурация)
- клиенты знают количество и адреса машин, и общаются с ними напрямую. Реплики не общаются между собой. Устраняется в [семинаре 2](https://github.com/ddvamp/distributed-db-learning/blob/main/notes/dist-sys-mipt/seminares/seminar-2.md)
- пишет только один конкретный клиент и строго последовательно, после его смерти писателей нет, читателей произвольное число. Устраняется [после](https://github.com/ddvamp/distributed-db-learning/blob/main/notes/dist-sys-mipt/lectures/lecture-2.md#случай-нескольких-писателей) построения алгоритма

Writer посылает запрос на три реплики и ждёт подтверждения с двух (одно не отказоустойчиво, трёх можем не дождаться). Каждая реплика хранит значение регистра, которые в моменте не синхронны. Записи могут задерживаться и обгонять друг друга, и чтобы реплика не перетёрла новое значение старым, writer снабжает записи $ts$ (счётчик операций). Реплика последовательно обрабатывает приходящие записи, при большем $ts$ записывает значение, иначе отбрасывает (в MVCC системах (`YDB`) всё равно записывается), и безусловно посылает подтверждение. **Blind write** - безусловная запись без сравнения с текущим значением

При чтении аналогично, посылаем запрос на три реплики и ждём двух ответов, из которых выбираем значение с большим $ts$

#### Quorum System

При $3$ репликах необходимы $2$ ответа, и допустим $1$ отказ, причём необходимое число ответов не уменьшается при отказах. В случае $n$ реплик число отказов $f \lt \left \lceil \frac{n}{2} \right \rceil$, и необходимо $\left \lfloor \frac{n}{2} \right \rfloor + 1$ подтверждений. Как видно, множества подтверждений чтения и записи имеют общие узлы по принципу Дирихле, т.е. пересекаются, поэтому чтение видит актуальную запись. Это обобщается в **Quorum System**: $P$ - множество всех узлов, $Q$ - система кворумов, $Q \subseteq 2^P$, если $\forall A, B \in Q: A \cap B \neq \emptyset$. В алгоритме, $A$ и $B$ это наборы реплик, отправившие подтверждения чтения и записи, а используемая система кворумов **Majorities** - минимальные строгие большинства. **Quorum** означает минимальное количество голосов для принятия решения. Говорят операция чтения или записи собирает кворум (нужное число подтверждений). Если запись $rt$ ordered before чтения, ожидаем, что кворум чтения увидит узлы из кворума записи. Если собрать кворум невозможно, система блокируется (нужно использовать таймауты). Т.о. Quorum System - это инструмент сокрытия отказов

#### Отказоустойчивость в QS

Не стоит выбирать чётное число реплик, поскольку при добавлении реплики размер кворума вырастает на единицу, а отказоустойчивость не изменяется. При этом есть гибкость выбора кворумов. Множества чтения и записи не обязаны быть равными. Можно уменьшить кворум чтения, тем самым ускорив операцию и увеличив число отказов за счёт большего кворума и более медленной операции записи, и меньшего числа отказов (например, $2 + 4$ на $5$ репликах с $3 + 1$ отказами). Если число отказов превысит допустимый предел, кворум не соберётся, а система перестанет отвечать (или сообщит о недоступности по таймауту), но главное не нарушит гарантий

Quorum System позволяет также избежать split brain. В нечётной конфигурации при partition будет меньшая часть, которая не сможет собирать кворумы и отключится. Либо, если кворум чтения меньше, обе части смогут обрабатывать операции чтения, но не записи. Примечательно то, что после восстановления partition система может продолжать функционировать без внешней синхронизации по построению алгоритма, поскольку значения в отключённой части будут отбрасываться (по $ts$), пока она не актуализируется

Также система кворумов поясняет, почему нельзя просто добавлять реплики. На новых будут кворумы корректного размера, но на старых кворумы относительно уменьшатся и смогут не собирать актуальные данные

#### Нарушение линеаризации

Описанный алгоритм не гарантирует линеаризуемость. Пусть неоконченная запись попала на реплику $1$, и клиент (или разные клиенты) делает два упорядоченных в $rt$ чтения, в первый раз получая ответы с реплик $1$ и $2$, а во второй с $2$ и $3$. Сначала он увидит новое значение, затем старое. Это не линеаризуемая история, и регистр не атомарен. Однако алгоритм и в таком виде используется в проде, например в `Apache Cassandra`, где решаемым задачам линеаризуемость не требуется, и за счёт этого упрощения можно сделать больше оптимизаций в другом слое

Суть в том, что кворум собирается не мгновенно, это продолжительное действие. Кворум чтения вместо пересечения с кворумом записи может захватить отдельный узел, а клиент, сам того не понимая, наблюдать отдельные локальные изменения и видеть незавершённую запись. Необходимо обеспечить "видимость" лишь собранных кворумов

#### Обеспечение линеаризуемости

Добавим *helping* и сделаем чтение двухфазным. Используя результат кворума чтения собираем кворум записи с тем же $ts$. Так по окончании чтения его результат в системе, и последующие в $rt$ order чтения гарантированно не увидят более ранние значения. Получили линеаризуемость за счёт усложнения/замедления операции чтения. И хотя исходный сценарий всё ещё возможен, теперь в нём две операции являются конкурентными, а значит их можно переупорядочить. Возможная оптимизация: не пишем на реплики с тем же $ts$, считая, что подтверждение записи от них уже есть

Итого, если клиент обеспечил $hb$, значит операции не пересекаются во времени, а значит они пересекаются по состоявшимся кворумам. Если же $hb$ нет, операции могут пересекаться, и порядок во времени может быть любым (при линеаризации выберем тот, что нам удобнее)

#### Доказательство линеаризуемости

Нужно найти линеаризацию для каждой возможной конкурентной истории. Построим последовательную историю, в которой записи разместим в порядке выполнения единственным писателем. Чтения поставим после прочитанных ими записей в произвольном порядке, но не раньше других чтений, которые $rt$ ordered before них. В получившейся истории операции выставлены по $ts$. Докажем, что этот порядок и есть линеаризация, то есть что он уважает $rt$. Рассмотрим две операции $o_1$ и $o_2$. Если они конкурентные, их порядок может быть любым. Пусть $o_1 \prec_{rt} o_2$. Если $o_2$ запись, то у неё самый новый $ts$ в системе. Если $o_2$ чтение, у $o_1$ обязательно есть фаза записи, от которой чтение получило $ts$. В обоих случаях $ts \left \lparen o_1 \right \rparen \le ts \left \lparen o_2 \right \rparen$, а значит в линеаризации операции стоят в таком же порядке, ч.т.д.

#### Случай нескольких писателей

Один неумирающий писатель невозможен. В случае нескольких нужно уметь распределённо выбирать единый монотонный timestamp, чтобы $w_1 \prec_{rt} w_2 \implies ts \left \lparen w_1 \right \rparen \lt \left \lparen w_2 \right \rparen$. **Узел координатор** (узел, который собирает кворумы и посылает ответ), на который попала операция записи, не может выбрать $ts = Now \left \lparen \right \rparen$ из-за рассинхронизации часов с другими координаторами

Сделаем запись двухфазной. Сперва выполним чтение, от которого возьмём $ts$ и монотонно продвинув переиспользуем его для фазы записи. Теперь операции чтения и записи обе состоят из двух фаз и равны по стоимости. Снова получили свойство, что кворумы на чтение и запись *разных* операций записи пересекаются в процессе их выполнения (это важно во многих алгоритмах)

В таком решении возможны одинаковые $ts$. Чтобы этого не допустить сделаем их парой $(ts, id)$, где $id$ - уникальный идентификатор узла координатора (или клиента, в зависимости от алгоритма генерации $ts$), задающийся при установке конфигурации. Нас не волнует неравнозначность реплик, поскольку без $hb$ пользователь и так ничего не ожидает

Доказательство линеаризуемости аналогично таковому для одного писателя

(см. Hybrid Logical Clock (`YDB`) как аналог, где нет фазы чтения)

#### Использование TrueTime

Для генерации распределённых монотонных $ts$ TrueTime позволяет использовать ожидание и упорядочение по атомным часам вместо дорогой коммуникации узлов в лишней фазе. Это называется **Commit Wait** (`Spanner`) - транзакция выполняется с $ts = TT.Now \left \lparen \right \rparen.latest$, но подтверждение клиенту посылается лишь когда $TT.Now \left \lparen \right \rparen .earliest \ge ts$. Так никакие последующие в $rt$ транзакции не выберут более ранний $ts$

#### Рестарты

Как поддержать рестарты? Путь реплики перед отправкой подтверждения координатору сохраняют состояние регистра на диск. При рестарте регистр можно восстановить, а состояние узла будет как после устранения partition-а. Теоретически, так можно было бы заменять отказавшие узлы, будто они со старта находились вне сети. В действительности [всё сложнее](https://github.com/ddvamp/distributed-db-learning/blob/main/notes/dist-sys-mipt/lectures/lecture-8.md#реконфигурация)

#### Оценка эффективности

Чтобы оценить эффективность алгоритма нужна модель стоимостей. Два основных расхода времени
- **RTT (RoundTrip Time)** - время хождения сообщения между двумя узлами туда-обратно
- **Disk** - время записи на диск вместе с flush-ем (*fsync* - отправить данные из страничного кэша на диск)

В зависимости от удалённости узлов и вида диска может превалировать либо одно, либо другое. Разумеется, необходимо минимизировать оба

Вся операция внутри системы (из двух фаз) называется **Quorum flush** - инициировать кворум, каждой репликой обработать запрос со сбросом диска, собрать ack-и. Его стоимостью и оценивается алгоритм

#### Итог

Задача решена без ожиданий от поведения часов и сети, скорости доставки сообщений и работы узлов. Если отказов меньше половины, eventually сообщения доходят, кворум собирается, и операция завершается с гарантией линеаризуемости

В [лекции 3](https://github.com/ddvamp/distributed-db-learning/blob/main/notes/dist-sys-mipt/lectures/lecture-3.md) рассмотрим как выполнять более сложные чем read/write операции

[↑ Содержание ↑](https://github.com/ddvamp/distributed-db-learning/tree/main/notes/dist-sys-mipt#содержание)\
[← Семинар 1](https://github.com/ddvamp/distributed-db-learning/blob/main/notes/dist-sys-mipt/seminars/seminar-1.md)
[Семинар 2 →](https://github.com/ddvamp/distributed-db-learning/blob/main/notes/dist-sys-mipt/seminars/seminar-2.md)
