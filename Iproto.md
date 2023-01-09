# Iproto #
В этом документе представлена ​​информация о технической реализации протокола
iproto в Tarantool. Протокол является бинарным, потому что доступ к базе
данных осуществляется с помощью бинарного кода, а не через текстоый запрос
на языке Lua. Посредством iproto обеспечивается полный доступ к функционалу
Tarantool.

```diff
- Нужно ли описывать функционал или же этого хватит? Все-таки описываем техническую реализацию.
```

---

В Tarantool есть один основной транзакционный TX тред, в рамках которого
выполняются все транзакции в памяти. Для того, чтобы работать с
пользователем извне существует отдельный сетевой тред &mdash; iproto. Он
принимает запросы из сети, обрабатывает протокол Tarantool, передаёт запрос
в TX тред и запускает пользовательский запрос в отдельном файбере.

---

Рассмотрим как происходит общение между TX и Iproto тредами. В греческой
мифологии Харон &mdash; это перевозчик душ умерших через реку Стикс,
отделяюшую мир живых от мира мертвых.

Здесь:

* Харон &mdash; с помощью cbus message уведомляет iproto тред о новых данных
    в obuf и возвращает в tx тред позицию в obuf, которая была успешно
    сброшена в сеть.

* река Стикс &mdash; cpipe;
* лодка &mdash; cbus message;
* мир живых &mdash; TX тред;
* мир мертвых &mdash; iproto тред

(Пока такая картинка, чуть позже сделаю анимацию)
![Iproto](/images/iproto.svg)

На gif'ке изображено следующее: cbus message инициируется в тот момент
когда данные появляются в TX треде. Далее с помощью cbus message данные
передаются в iproto тред. Часть данных из iproto сбрасывается в сеть, т.е.
меняется wpos у obuf, соответсвенно это новое знвчение передается в TX тред
посредством cbus message. Аналогично делается и для ibuf только в другую
сторону (из iproto в TX). Число запросов одновременно находящихся в
cpipe'ах &mdash; 2. В процессе работы, отправив специальное сообщение
(`IPROTO_CFG_STOP`) это число можно изменить.


```C
/**
 * The maximal number of iproto messages in fly.
 */
static int iproto_msg_max = IPROTO_MSG_MAX_MIN;


/** Available iproto configuration changes. */
enum iproto_cfg_op {
	/** Command code to set max input for iproto thread */
	IPROTO_CFG_MSG_MAX,
	/**
	 * Command code to start listen socket contained
	 * in evio_service object
	 */
	IPROTO_CFG_LISTEN,
	/**
	 * Command code to stop listen socket contained
	 * in evio_service object. In case when user sets
	 * new parameters for iproto, it is necessary to stop
	 * listen sockets in iproto threads before reconfiguration.
	 */
	IPROTO_CFG_STOP,
	/**
	 * Command code do get statistic from iproto thread
	 */
	IPROTO_CFG_STAT,
};

/**
 * Since there is no way to "synchronously" change the
 * state of the io thread, to change the listen port or max
 * message count in flight send a special message to iproto
 * thread.
 */
struct iproto_cfg_msg: public cbus_call_msg
{
	/** Operation to execute in iproto thread. */
	enum iproto_cfg_op op;
	union {
		/** Pointer to the statistic stucture. */
		struct iproto_stats *stats;
		/** Pointer to evio_service, used for bind */
		struct evio_service *binary;
		/** New iproto max message count. */
		int iproto_msg_max;
	};
	struct iproto_thread *iproto_thread;
};
```

---

## IBuf ##
- [ ] Вставить картинку

Один из двух сетевых буферов, предназначенный для чтения из сокета.
Он работает с запросами, приходящими из сети, причем для обработки запросов
они должны быть непрерывны в памяти. Поэтому `IBuf` запрашивает у
`Slab cache` фрагмент памяти и использует его, а когда не хватает &mdash;
берет побольше и переносит информацию из предыдущего фрагмента. У IBuf
даже API нет, это просто структура с четырьмя указателями, буфером и
методом, который умеет делать realloc.

Удобнее всего использовать по два таких буфера на каждое сетевое
подключение. При чтении из одного сокета Tarantool вычитывает в один буфер
сразу много запросов. Очевидно, после обработки запроса он уже не нужен,
но, поскольку он живет в одном буфере с еще нужными запросами, удалить
его нельзя. Поэтому по мере накопления запросов в одном буфере берётся
следующий буфер &mdash; тогда рано или поздно все запросы из первого
буфера будут выполнены и его, буфер, можно будет целиком освободить.

## OBuf ##
- [ ] Вставить картинку

Второй из сетевых буферов, предназначенный для отправки ответа в сеть.
Он не обязан быть непрерывным в памяти. Самое главное, что он умеет
делать &mdash; сохранять позицию в своем буфере. Когда Tarantool отвечает
на запрос по сети, первые несколько байтов ответа &mdash; это размер
ответа. А размер мы не знаем, пока не сформируем весь ответ. Поэтому мы
запоминаем позицию в памяти, дописываем все данные, которые потребовались,
после чего возвращаемся на ту самую позицию, меняем уже посчитанный размер
и работаем дальше.

## iproto_stream ##
Имеется хеш-таблица потоков для каждого соединения. Когда новый запрос
приходит с ненулевым идентификатором потока, ищем поток с таким ID в этой
таблице и если его нет, мы его создаем. Заявка помещается в очередь
ожидающих запросов, и если эта очередь была пуста на момент ее поступления,
то она передается в поток tx для обработки. Когда запрос возвращается в
сетевой поток (запрос обработан tx тредом), мы берем следующий запрос из
очереди ожидающих запросов и отправляем его в поток tx. Если нет ожидающих
запросов, мы удаляем объект из хеш-таблицы и уничтожаем его. Запросы с
`stream ID = 0` обрабатываются по старинке, т.е. без использования
iproto_stream. Структура, описывающая iproto_steram представлена ниже:

```C
struct iproto_stream {
	/** Currently active stream transaction or NULL */
	struct txn *txn;
	/**
	 * Queue of pending requests (iproto messages) for this stream,
	 * processed sequentially. This field is accesable only from
	 * iproto thread. Queue items has iproto_msg type.
	 */
	struct stailq pending_requests;
	/** Id of this stream, used as a key in streams hash table */
	uint64_t id;
	/** This stream connection */
	struct iproto_connection *connection;
	/**
	 * Pre-allocated disconnect msg to gracefully rollback stream
	 * transaction and destroy stream object.
	 */
	struct cmsg on_disconnect;
	/**
	 * Message currently being processed in the tx thread.
	 * This field is accesable only from iproto thread.
	 */
	struct iproto_msg *current;
};
```

В каждом iproto треде содержится пул iproto_stream
```C
struct iproto_thread {
    ...
    /*
    * Iproto thread memory pools
    */
    struct mempool iproto_msg_pool;
    struct mempool iproto_connection_pool;
    struct mempool iproto_stream_pool;
    ...
};
```

Выделение памяти и инициализация iproto_stream
```C
static struct iproto_stream *
iproto_stream_new(struct iproto_connection *connection, uint64_t stream_id)
{
    struct iproto_thread *iproto_thread = connection->iproto_thread;
    struct iproto_stream *stream = (struct iproto_stream *)
        mempool_alloc(&iproto_thread->iproto_stream_pool);
    if (stream == NULL) {
		diag_set(OutOfMemory, sizeof(*stream), "mempool_alloc", "stream");
        return NULL;
    }
    ...
    stream->txn = NULL;
    stream->current = NULL;
    stailq_create(&stream->pending_requests);
    stream->id = stream_id;
    stream->connection = connection;
    return stream;
}
```

Если больше нет сообщений для текущего stream и нет стартующих транзакций,
то iproto_stream можно удалить.
```C
static void
iproto_stream_delete(struct iproto_stream *stream)
{
	assert(stream->current == NULL);
	assert(stailq_empty(&stream->pending_requests));
	assert(stream->txn == NULL);
	mempool_free(&stream->connection->iproto_thread->iproto_stream_pool, stream);
}
```

## mempool ##
- [ ] Вставить картинку

Классический пул аллокатор. Как и прочие подобные, этот аллокатор умеет
выделять блоки одного фиксированного размера и предназначен для длительного
хранения данных, удаление блоков происходит в произвольном порядке. Mempool
берет из Slab cache большие slabы и размечает их под требуемый размер.
Интересна стратегия переиспользования удаляемых блоков. В каждом slabе
хранится свой список удаленных из него блоков (free list). При этом slabы
одного Mempoolа делятся по степени заполненности на горячие и холодные.
Для нового выделения используется free list по возможности горячего slabа
с минимальным адресом. Такая стратегия позволяет хоть как-то бороться с
общей проблемой всех пулов памяти &mdash; фрагментацией.

Представим себе типичную случайную нагрузку на такой аллокатор:
пользователь сначала выделил много блоков, а потом начинает циклично
выделять новый/удалять случайный старый, причем удалять старые блоки
приходится немного чаще, чем выделять новые. Очевидно Mempool не может
освободить slab до тех пор, пока в нем содержится хотя бы один используемый
блок. Поэтому при такой нагрузке появляется фрагментация &mdash; slabов
много, в них будет много свободной памяти, но вот освободить их для общих
нужд (например для других Mempool) этот Mempool не может. Если использовать
один общий free list (что является стандартным подходом при реализации
пула памяти) &mdash; то новые размещения в памяти будут попадать в
случайные slabы, и даже после полной ротации (когда каждый блок из
изначально выделенных был освобожден) фрагментация останется. Поэтому
Mempool в Tarantool старается новые размещения делать в более плотных и
каких-то определенных slabах, и при полной ротации блоков все прочие slabы
будут точно пусты и соответственно возвращены обратно в Slab cache.
