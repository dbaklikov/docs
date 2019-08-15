Модуль репликации является составной частью VDisk'а и предназначен для восстановления содержимого утраченных блобов, то есть для восстановления данных. Модуль синхронизации, в отличие от репликации, занимается восстановлением метаданных.

### Общая схема работы

Модуль репликации представляет собой постоянно работающий сервис, который выполняет _циклы репликации_. Каждый цикл состоит из полного обхода базы метаданных блобов и поиска тех блобов, которые должны находиться на текущем VDisk'е, но фактически на нем отсутствуют.
В процессе обхода базы блобов, вырабатываются команды на вычитывание частей блобов к соседним VDisk'ам. Результаты выполнения этих команд агрегируются для каждого запрошенного блоба с целью восстановления его локальной части и эта часть записывается в базу.
Поскольку репликация, как правило, носит массовый характер (восстановление всех данных после замены диска), то запись производится SSTable'ами. Они формируются на лету по мере получения ответов от соседних VDisk'ов.

#### Алгоритм работы

Алгоритм работы репликации достаточно прост. Цикл репликации запускается актором планировщика (TReplScheduler) с периодичностью, заданной в конфигурации. Эта периодичность задается в виде интервала от времени окончания предыдущего цикла репликации до времени начала следующего. Настройка по умолчанию -- 60 секунд. Каждый //цикл репликации// состоит из последовательности //квантов//, в пределах которых выполняются два последовательных этапа: планирование и репликация.

Планирование в кванте репликации заключается в обходе БД и поиске в ней тех блобов, данные которых нужно восстановить локально. Результатом планирования является перечень идентификаторов блобов, подлежащих репликации.

Собственно репликация заключается в отправке запросов на соседние диски и восстановлении локальных копий данных для блобов, перечень которых был сформирован на этапе планирования.

Следующий квант репликации начинается с того ключа, на котором остановился предыдущий. Это не обеспечивает консистентности по ключам, однако в данном процессе она не важна. Продолжительность кванта определяется рядом настроек:

1. Ограничение на время планирования (настройка по умолчанию -- 100 мс), в течение которого актор планировщика ходит по снимку БД. Это ограничение введено с тем, чтобы batch pool не занимался надолго обработкой одного сообщения.
2. Ограничение по количеству восстанавливаемых байт (настройка по умолчанию -- 384 МБ). Число восстанавливаемых байт считается как суммарное количество полезных байт блобов, подлежащих репликации, которые содержатся в локальной копии. Таким образом, например, для кодирования mirror-3 это будет суммарное число байт в восстанавливаемых блобах, для block-4-2 это будет приблизительно 1/4 от этого количества.
3. Ограничение по числу восстанавливаемых блобов (настройка по умолчанию -- 100 000 штук). Это количество не должно быть большим, чтобы не затягивать квант репликации, а, кроме этого, чтобы не занимать большой объем памяти хранимым вектором метаданных для каждого восстанавливаемого блоба в алгоритмах репликации.

Для репликации отбираются только те блобы, которые удовлетворяют следующим требованиям:

1. Не помечены для сборки мусора (то есть либо имеют флажок Keep при отсутствии DoNotKeep, либо идентификаторы которых опережают барьер для указанной таблетки).
2. Согласно метаданным части блобов были замечены на локальном диске (по битам в Ingress).
3. Согласно метаданным имеют достаточное для восстановления количество реплик на других дисках.

Результатом планирования кванта является вектор идентификаторов блобов, каждому элементу которого соответствует битовая маска восстанавливаемых частей. Как правило, она содержит один бит, однако, в случае handoff-реплики, может содержать более одного бита.

После планирования для всех соседних дисков в группе создаются прокси репликации (ReplProxy), функция которых -- получить соответствующие части блобов с других дисков для восстановления. Запросы рассылаются только тем дискам, которые, по мнению метаданных, должны содержать части запрошенных блобов. Прокси работают асинхронно и передают полученные данные в центральный актор, который производит сортировку полученных блобов и восстанавливает данные. Восстановленные данные, если они не являются большими блобами, помещаются в SSTable'ы, которая строится в процессе репликации и создается на квант. Блобы идут упорядоченно по идентификатору, поэтому данные записываются в SSTable'ы без дополнительной обработки. Для больших блобов логика обработки отдельная, поэтому они отправляются сообщением в Skeleton, где уже обрабатываются особым образом.

После обработки всех запланированных блобов квант репликации завершается и, если не были обработаны все ключи БД, запускается новый квант. Если же все ключи были обработаны, то цикл завершается.

На выходе каждого кванта существует статистика, которая включает в себя следующие параметры:

1 Результаты планирования:

  1.1. ReplicaOk -- блобы, которые имеют все локальные реплики и не требуют репликации

  1.2. ReplicaNoMain -- блобы, которые имеют локальные реплики, но не имеют основной реплики на локальном диске, когда должны ее иметь (FIXME)

  1.3. RecoveryScheduled -- число запланированных к репликации блобов
 
  1.4. NoReplicsToRecover -- число блобов, которые нужно реплицировать, но имеют согласно метаданным недостаточное количество реплик на других дисках
 
  1.5. IgnoredDueToGC -- число блобов, которые уже не нужны и будут удалены при компакшене (придумать слово)

2 Результаты восстановления

2.1. DataRecoverySuccess -- число успешно восстановленных блобов (не их частей, а именно блобов)

2.2. DataRecoveryFailure -- число ошибок восстановлении блобов

2.3. DataRecoveryNoParts -- число блобов, для которых не было достаточного количества частей на других дисках

2.4. DataRecoverySkip -- число блобов, для которых на других дисках вообще не нашлось частей

2.5. DuplicateParts -- число частей-дубликатов (когда одна часть одного блоба пришла больше одного раза)

3 Детальная статистика

3.1. BytesRecovered -- число байт в восстановленных **частях**

3.2. LogoBlobsRecovered -- число восстановленных частей в небольших блобах

3.3. HugeLogoBlobsRecovered -- число восстановленных частей в больших блобах

3.4. ExcessBytesReceived -- число байт в лишних дубликатах (то есть начиная со второй копии каждой части)

3.5. ChunksWritten -- число записанных чанков (может считаться неправильно)

4 Статистика по времени выполнения разных фрагментов кода в репликаторе

SSTable'ы, записанные в результате репликации, помечаются специальным образом, как восстановленные в результате репликации и имеют особенность в контексте SyncLog'а. Дело в том, что для каждого блоба в обычного SSTable есть запись в логе, т.к. для каждого полученного блоба делается отдельная запись, и, как следствие, LSN. Этот LSN используется в SyncLog'е для передачи на другие диски при восстановлении лога после падения. При восстановлении SSTable'а, рожденного в результате репликации, реальных записей в логе и соответствующих восстановленным блобам LSN'ов нет, поэтому их приходится имитировать, выделяя под запись лога, в которой в индекс добавляется новый SSTable, диапазон LSN'ов по числу блобов + 1. 

При воспроизведении журнала, когда встречается запись с SSTable'ами, созданными в результате репликации, модуль восстановления смотрит, прошел ли LSN SyncLog'а последний элемент этой таблицы и, если да, то пропускает ее. Если же нет, то генерируются записи в SyncLog и рассылаются на соседние диски.

Удалить такой SSTable можно только когда LSN SyncLog'а переедет ее последний элемент. До этого момента, если она была удалена из индекса, она переезжает в специальное хранилище устаревших таблиц, где и покоится, ожидая своей скорбной участи.

### Особенности работы репликации

* фантомные блобы
* синхронизация и журнал
* циклы/кванты