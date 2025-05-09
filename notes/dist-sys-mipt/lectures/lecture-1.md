## Lecture 1. Модель распределённой системы

#### Типы распределённых систем

Все современные сетевые сервисы используют распределённые системы различных типов
- **KV Store** - огромная распределённая хэш-таблица с типовыми операциями $Set \left( key, value \right)$ и $Get \left( key \right)$, со случайным доступом поверх последовательных дисков и упорядоченностью элементов (like `std::map`). Весь диапазон ключей шардируется некоторым способом, не обязательно хэш-функцией, и в пределах шардов ключи упорядочиваются. Пример: `Cassandra`, `BigTable`
- **File System** - распределённая система хранения файлов огромных размеров или количеств, не умещающихся на одной машине, с последовательным доступом к данным. Отличается от KV Store паттерном доступа. Пример: `GoogleFS`, `Colossus`, `HDFS`
- **Coordination Service** - распределённая система для построения других распределённых систем. Предоставляет примитивы синхронизации и согласованности: выбор лидера, lease и т.д. Выполняет аналогичную роль что и атомик concurrency. Пример: `ZooKeeper`, `Chubby`
- **Message Queue** - распределённый асинхронный транспорт данных для многих одновременных продюсеров и консьюмеров. Пример: `Apache Kafka`
- **Database** - таблицы/колонки, индексы, транзакции, только распределённо. Самый сложный тип распределённых систем. Пример: `YDB`, `Google Spanner`, `CockroachDB`

В первую очередь это разные модели данных, и лишь во вторую имеют различия в построении. Также все эти системы имеют нераспределённые варианты

#### Причины использовать распределённые системы

- **масштабируемость (Scalability)** - возможность свободно добавлять новые вычислительные ресурсы и/или дисковые ёмкости, или же расти *горизонтально*, когда стало слишком много данных/операций для текущего количества машин, адаптируясь к нагрузке и используя при этом стандартное оборудование (свобода от Vendor Lock)
- **отказоустойчивость (Durability + Reliability + Fault-tolerance + Availability)** - машина/диск может сбоить или даже отказать, поэтому в большом кластере всегда есть машины на обслуживании. Необходимо переживать сбои узлов, проблемы сети, электросети и человеческие ошибки, автоматически (без помощи человека) адаптироваться и скрывать это от пользователя
  - **Durability (Надёжность хранения)** - успешно завершённые операции не будут потеряны даже при сбое системы. **Persistent state** - надёжная запись на диск, которой противопоставляется **Volatile state** - хранение в памяти
  - **Reliability (Надёжность работы)** - система выполняет свои функции корректно и стабильно в течение длительного времени. Думаем как предотвратить сбой узла (обслуживание оборудования, логи и мониторинги, тестирование, профилирование, стабилные версии библиотек и ОС и т.д.)
  - **Fault-tolerance (Устойчивость к сбоям)** - корректность работы системы не нарушается при отказе любого из компонентов (узлов, сети, дисков и т.п.). Когда сбой произойдёт, думаем как не пострадать. Достигается некоторой степенью **избыточности (Redundancy)** - увеличенным использованием компонента. Например отказ диска можно пережить используя *RAID-массив*
  - **Availability (Доступность)** - даже если часть компонентов недоступна, система продолжает отвечать на запросы в разумное время, возможно с ограничениями. $4/5$ девяток ($99.99\\%$ или $99.999\\%$)- сколько времени в год сервер должен быть доступен
- **безопасность (Safity)** - нельзя доверять машине/группе машин, поскольку они могут оказаться злоумышленниками и обманывать остальные машины. Особенно важно в блокчейне

Эти причины независимы. К примеру, отказоустойчивость может быть необходима там, где данные помещаются на одну машину (`ZooKeeper`). Аналогично с безопасностью (`Bitcoin`)

- **согласованность (Consistency)** - ещё одно свойство распределённых систем, гарантировать в той или иной степени, что геораспределённые клиенты будут способны конкурентно взаимодейтсовать с системой и видеть непротиворечивые данные

#### Описание модели

Для построения эффективных и полезных распределённых отказоустойчивых алгоритмов, определим модель, отражающую реальный мир без излишних подробностей. Доказав корректность алгоритмов в этой модели, можно будет утверждать, что они верны и в реальном мире

В этой модели система состоит из
- узлов
- сети (проводов и сообщений)
- клиентов

**Узел (node, process в старых статьях/книгах)** - это автономная единица вычислений и/или хранения, участвующая в совместной работе системы. Это машина, процесс или программная единица, например актор (таблетка в `YDB`). Это сетевой сервис

Узлы взаимодействуют посредством отправки односторонних сообщений (message) без ожидания ответов (reply), используя для этого сеть. **Сеть** есть совокупность проводов. Для простоты, между каждой парой узлов есть прямой и надёжный провод (link, TCP), по которому движутся сообщения, и узлы видят только его. Это модель **Message Passing**

Цель описываемой модели предоставить *гарантии*. Допустим, асинхронно отправили сообщение $m$ узлу $p$ - $Send \left( m, p \right)$. Оно не может дойти моментально, но и верхняя граница неизвестна, только то, что eventually на узле $p$ сработает обработчик сообщения $m$. В реальности вместо ответов применяется механизм Retry-ев, сети более сложные чем прямой провод, а скорость передачи ограничена физически (скорость света + среда/материал) и географически (удалённые узлы, локальность), что вносит задержки, и распределённые системы должны это учитывать. Здесь появляется понятие *стоимости*

Также есть **клиенты**, которые взаимодействуют с узлами по модели **Client-Server** - посылают запросы и ждут ответы. Есть несколько способов взаимодействия
- http api
- RPC - Remote Procedure Call
- client library - плюсовые обёртки над другими API + полезная функциональность, например `Asio`

Клиенты (почти) не знают об узлах. Они отправляют запросы на единый сетевой адрес, которые проксируются в конкретные узлы. Альтернативно балансировка может выполняться в клиентской библиотеке

Клиент выполняет запрос $Set \left( k, X \right)$ и ждёт подтверждение, система отказоустойчиво сохраняет $X$ и посылает это подтверждение. Если затем клиент сообщит об этом другому по внешнему каналу связи, второй ожидает по $Get \left( k \right)$ увидеть $X$, т.е. *наличие причинности*. Клиенты не отправляют сообщения в систему (явно), не видят сотни узлов, не наблюдают распределённость и не думают о ней. Для них система выглядит как бесконечно ёмкий, бесконечно мощный, бесконечно устойчивый компьютер, и в коде это конкурентный объект, поэтому внутри системы удобно использовать модель Message Passing, а снаружи модель **Shared Memory** (Consistency models). При наличии причинности система должна учитывать возможность внешней коммуникации при обеспечинии согласованности. Также возможны распределённые системы, которые заставляют пользователей передавать друг другу служебную информацию для обеспечения причинности по side-channel (за пределами системы). Пример: `Cassandra`, `Dynamo`

В реальности для реализации пересылки узлами друг другу сообщений могут использоваться более высокоуровневые абстракции, например RPC, которые ближе к клиент-сервер модели

#### Гарантии факта доставки сообщения

- **Fair-loss link** - модель честного канала: если узел $A$ будет бесконечно отправлять сообщение узлу $B$, то второй будет бесконечно его получать. Нет момента, когда сообщение навечно перестало доходить. Однако, нет порядка сообщений, нет гарантии доставки, возможны дубликаты. С такой абстракцией неудобно работать
- **Reliable link** - модель надёжного канала: каждое однократно отправленное сообщение будет доставлено, но могут быть дубликаты, и нет порядка
- **Perfect link** - модель идеального канала: каждое однократно отправленное сообщение будет доставлено и ровно один раз, но нет порядка. Пример: TCP - отслеживание дублей, повторная отправка до подтверждения, упорядочение на стороне доставки. Используется в литературе

Эти модели не рассматривают реальность (считают, что узлы/сеть не отказывают), но требуют, чтобы в какие-то периоды времени сеть становилась "идеальной" (доставляла сообщения). Также эти модели не допускают сообщений из воздуха

- **FIFO link** - модель канала, гарантирующая порядок. Ортогональна предыдущим

Будем работать в немного изменённой Reliable модели, где сообщение либо доставится, либо соединение сообщит о разрыве. Такая модель отражает, что в реальности разрывы не так просто обработать. В момент разрыва неизвестно, выполнилась ли операция или нет, поэтому возможны два плохих сценария
- операция не выполнилась, и, если её не повторить, она потеряется
- операция выполнилась, и, если её повторить, и она не *идемпотентная*, система сломается

**Идемпотентность** - свойство операции в случае многократного выполнения с одним и тем же входом не менять результат после первого применения

Задача по обработке разрывов является отдельным инженерным вызовом

#### Гарантии времени доставки сообщения

- **Synchronous model (Cинхронная модель)** - время доставки сообщений ограничено сверху $delay \left( m \right) \le \delta$. Непрактична, поскольку в реальности система ненадёжна (разрывы, баги в коммутаторах, высокий трафик, обслуживание, человеческие ошибки и т.д.)
- **Asynchronous model (Асинхронная модель)** - ограничения на время доставки сообщений нет. В этой модели доказывается корректность алгоритмов, т.е. что система обладает свойством **Safity** - не уходит в некорректные состояния (не делает ничего плохого). Но такие алгоритмы/системы бесполезны
- **Partially Synchronous model (Частично синхронная модель)** - модель неопределённо долго асинхронная, но в некоторый неизвестный момент времени $t^\*$ гарантированно станет произвольно коротко синхронной (например, сеть перестала сбоить). Алгоритм должен быть устойчив к худшему сценарию - ждать, когда момент $t^\*$ настанет, и совершить прогресс. В этой модели доказывается, что система делает что-то хорошее, т.е. обладает свойством **Liveness** - eventually совершает прогресс (делает что-то хорошее). Говоря иначе, частично синхронная модель означает, что сеть работает непредсказуемо и не даёт никаких гарантий о времени доставки, но периодически такие гарантии появляются

В зависимости от ситуации будем пользоваться асинхронной или частично синхронной моделью

#### Моделирование сети

- **Partition** - разделение системы на два связных, полностью изолированных друг от друга кластера в результате нарушений сети (отказ link-ов). Также разделяются и пользователи (по модулю проксирования запросов на узлы кластеров)
- **Split Brain** - ситуация, когда при partition кластеры думают, что они вся система, и продолжают работать независимо. Её нельзя допускать, поскольку узлами она ненаблюдаема до и невосстановима после починки сети - консистентность системы нарушится, и клиенты пронаблюдают наличие узлов. Вариант решения: одна из частей должна остановиться до починки (CAP теорема), или обе части дожны перестать работать на запись (?кажется здесь возможна проблема отсутствия последних записей, которые состоялись, но не успели попасть в меньший кластер)

Модель уже учитывает partition - случай бесконечно медленных link-ов на некотором срезе

#### Моделирование узлов

Для целей модели узел есть однопоточный автомат, который получает из сети сообщения и вызывает для них обработчики, меняет своё состояние и, возможно, отвечает сообщениями в сеть/систему. Многопоточность в реализации не влияет на свойства системы, но увеличивает производительность и удобство построения. Для доказательства теорем/алгоритмов можно думать, что все действия узла атомарны, что не повлияет на корректность реальных (неатомарных) систем

Скорость работы узлов произвольная (например может уменьшиться из-за начала сборки мусора/удаления большой структуры объектов). Узел может выйти из строя (Crash, "взорваться", отказ диска). Узел может перезагрузиться (Restart), что требует добавить персистентное состояние (диск). Также возможны Византийские отказы (Byzantine), когда узел ведёт себя непредсказуемым образом, например является злоумышленником и отвечает всем по разному. Последнее сильно усложняет алгоритмы (помимо количества отказов нужно закладывать процент возможных злоумышленников), но и учитывается лишь в безопасных системах (например, криптовалюты)

В зависимости от алгоритма/архитектуры узлы могут быть как равноправными, так и иметь разные роли и даже сменять их (например, лидер)

#### Время

**Время (time)** абсолютно для всех узлов, и каждое событие в системе имеет координату на общей оси времени. Время не влияет на Safity, но должно уважать его при учёте в Liveness. Однако узлы не имеют доступа ко времени, только к **часам (clock)**, которые отображают абсолютное время в **timestamp** $c \left( t \right) \to ts$. В идеале это тождество, но в действительности это оценка $Now \left( \right) \to ts$ от физического процесса в часовом механизме, например кварцевом (quartz, в ПК) или атомном (atomic, точные и дорогие, на специальных машинах)

Принципиально все использования часов сводятся к двум действиям
- $t_1 \lt t_2$ - сравнение timestamp-ов для упорядочивания событий (согласование порядка действий с внешней причинностью)
- $t_2 - t_1$ - вычисление интервала времени для установки timeout-а или организации failure-detection (время ожидания подтверждения доставки сообщения до узла, время отчёта лидера об его активности - heartbeat-ы)

#### Погрешности часов

Часы строят из объектов реального мира, поэтому они имеют погрешности, что необходимо учитывать в алгоритмах
- **Skew** - рассинхронизация часов на разных узлах. Проблема для упорядочивания, когда порядок по часам не соответствует порядку во времени. Пример: клиент $A$ выполнил $Set \left( key, X \right)$ на узле со временем $12:00$, после чего сообщил об этом клиенту $B$, который выполнил $Set \left( key, Y \right)$ на узле со временем $11:59$, но в системе остался $X$ - записи переупорядочились
- **Drift (дрейф, скитание)** - явление нестабильности скорости измерения времени часами, когда часы начинают спешить или отставать. Проблема для точных интервалов. **ppm (parts per million, миллионные доли)** - на сколько микросекунд часы могут отклониться за одну секунду. У кварцевых часов ppm равен $20$ ($\approx 1.7$ секунд в сутки)

***Конец описания модели***

#### Синхронизация часов (невозможна)

Drift неизбежен, но можно ли устранить Skew сведя часы к одному показанию, не обязательно к "реальному" времени? Докажем, что это невозможно

Попытаемся синхронизировать два узла. Узнаем на узле время $t_1 = Now \left( \right)$, запросим время у другого узла и при получении ответа $t_3$ снова узнаем время $t_2 = Now \left( \right)$. Примерная задержка ответа $\delta \approx \frac{t_2 - t_1}{2}$. Значит, если сеть симметрична, на другом узле сейчас $t_3 + \delta$

Пусть время доставки сообщений $delay \left( m \right) \in \left[ \Delta - u, \Delta \right]$, где $u$ - uncertainty (неопределённость), и дрейфа нет. Синхронизировать часы означает выбрать для каждого узла некоторую поправку $O_k$ к текущему показанию его часов $SC_i \left( t \right) = C_i \left( t \right) + O_i$. Предположим, есть произвольный алгоритм, который после запуска за некоторое время даст $\left| SC_i \left( t \right) - SC_j \left( t \right) \right| \le \varepsilon$. Определим нижнюю оценку $\varepsilon$, т.е. насколько точно можно синхронизировать часы

Пусть мы управляем миром, в том числе работой сети (задержками) и начальным отклонением часов, иными словами, можем вообразить любые возможные исполнения (эта техника полезна при доказательстве алгоритмов). Построим два различных исполнения, в которых поправки вычисляются независимо и в общем случае должны различаться, но таких, что алгоритм не сможет уловить различий и выберет в обоих случаях одинаковые поправки

$t_{i \to j}$ - время доставки сообщения от $i$ к $j$. Пусть $t_{1 \to 2} = \Delta$, $t_{2 \to 1} = \Delta - u$. На такой конфигурации запускаем алгоритм и вычисляем $\varepsilon$. Затем делаем **shift** - оставляем события на первом узле на тех же местах, но переворачиваем сеть ($t_{1 \to 2} = \Delta - u$, $t_{2 \to 1} = \Delta$), и также переводим часы второго узла на $+u$, чтобы ответы остались такими же. Оба узла не видят разницы, значит в обоих случаях алгоритм выберет одинаковые поправки $O_k$, но для второго узла

```math
C^{'}_2 \left( t \right) = C_2 \left( t \right) + u \Rightarrow SC^{'}_2 \left( t \right) = SC_2 \left( t \right) + u \Rightarrow SC^{'}_2 \left( t \right) - SC_2 \left( t \right) = u
```

И поскольку разница между двумя часами может быть от $-\varepsilon$ до $\varepsilon$, то

```math
SC_2 \left( t \right) \in \left[ SC_1 \left( t \right) - \varepsilon , SC_1 \left( t \right) + \varepsilon \right], SC^{'}_2 \left( t \right) \in \left[ SC_1 \left( t \right) - \varepsilon , SC_1 \left( t \right) + \varepsilon \right] \Rightarrow 2\varepsilon \ge u \Rightarrow \varepsilon \ge \frac{u}{2}
```

В случае n узлов

```math
\varepsilon \ge \left( 1 - \frac{1}{n} \right) u
```

Мы не можем оценить асимметрию сети, только измерить время roundtrip-а - путишествия сообщения туда-обратно. Стоит отказаться от синхронизации времени, поскольку время это разделяемое между узлами состояние, которое не является когерентным. Именно поэтому safety не должно зависеть (и не зависит) от времени, лишь liveness

Всё становится лучше, если отказаться от асимметрии, например, синхронизировать одни часы по другим, но не обратно

#### Почему GPS синхронизирует часы

Из соображений геометрии нужны три спутника для точного определения положения в пространстве. Достаточно пересечь сферы расстояний и взять точку на поверхности Земли. Это выливается в систему трёх неизвестных $\left| X - X_i \right| = d_i$. Однако расстояние до спутников измеряется при помощи локальных часов, которые имеют погрешность $\delta$ относительно времени GPS, так что система на самом деле выглядит так $\left| X - X_i \right| = \upsilon * \left( \Delta t_i + \delta \right)$, что складывается в четыре неизвестных и четыре уравнения - *навигационные уравнения GPS*. Поэтому синхронизация часов является побочным результатом. Чтобы это работало, часы на спутниках имеют огромную точность (у них очень маленький drift, избыточность кристаллов, и их постоянно синхронизируют)

#### TrueTime

**Google.TrueTime** - локальный сервис для синхронизации часов. Вызов $TT.Now \left( \right)$, используя локально сохранённое состояние на узле, возвращает диапазон $\left[ e \left( arliest \right) , l \left( atest \right) \right]$, в пределах которого находится истинное время. Гарантируется, что $\left[ t_0, t_1 \right] \cup \left[ e, l \right] \ne \emptyset$ и $l - e \le 6ms$, где $t_0$ - истинное время отправки запроса и $t_1$ - истинное время получения ответа (например, запроса внутри `Spanner`). Для этого каждые $30$ секунд демон сервиса TT на узле синхронизирует локальное состояние (интервал $\left[ e_0, l_0 \right]$), неизменное до следующей синхронизации, с одним из *тайм мастеров* кластера (в пределах одного датацентра), а ppm устанавливается в $200$, что заведомо больше, чем в часах узла. При запросе $TT.Now \left( \right)$ сервис берёт копию локального состояния и сдвигает границы, имитируя дрейф, используя монотонные часы, нижнюю c $ppm = -200$, а верхнюю с $ppm = 200$. Так интервал с настоящим дрейфом заведомо попадает в вычисленный

Тайм мастеров два вида, либо узел с *GPS антеной* (для синхронизации своих часов по GPS), либо *Armagedon Master* с атомными часами. Они независимы и имеют разные сценарии отказа

TrueTime используется для замены долгого межконтинентального общения ($\sim10-100ms$) ожиданием не дольше $6ms$ (в настоящее время сильно меньше). Этот сервис был построен для использования в `Google Cloud Spanner`

[↑ Содержание ↑](https://github.com/ddvamp/distributed-db-learning/tree/main/notes/dist-sys-mipt#содержание)\
[Семинар 1 →](https://github.com/ddvamp/distributed-db-learning/blob/main/notes/dist-sys-mipt/seminars/seminar-1.md)
