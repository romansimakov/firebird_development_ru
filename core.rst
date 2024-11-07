Ядро
================

ShaperMemory
------------

Назначение
~~~~~~~~~~~~~~
Из-за архитектуры Classic возникает проблема: как создавать общие статические переменные, доступные во всех процессах, и обмениваться данными.
Одним из решений является использование шареной (общей) памяти. Это память, к которой есть доступ из нескольких процессов.

Классы
~~~~~~~~~~~~~~
В Firebird есть готовый класс `SharedMemoryBase` (isc_sync.cpp). Он работает через мапленый в память файл.
Для его использования необходимо унаследоваться от класса `IpcObject`, а также создать наследника от `Firebird::MemoryHeader`.
Шареная память делится на заголовок и содержимое. Как правило, заголовок содержит метаинформацию, например, позицию чтения/записи.
Например:

.. code-block:: cpp

    struct TraceLogHeader : public Firebird::MemoryHeader
    {
        static const USHORT TRACE_LOG_VERSION = 2;

        ULONG readPos;
        ULONG writePos;
        ULONG maxSize;
        ULONG allocated;        // ноль, когда читатель ушел
        ULONG flags;
    };

Для основного класса нужно переопределить ряд функций. Пример:

.. code-block:: cpp

    USHORT getType() const override { return Firebird::SharedMemoryBase::SRAM_TRACE_LOG; }
    USHORT getVersion() const override { return TraceLogHeader::TRACE_LOG_VERSION; }
    const char* getName() const override { return "TraceLog"; }

В функции `initialize` происходит первичная настройка заголовка. Пример:

.. code-block:: cpp

    bool PluginLogWriter::initialize(SharedMemoryBase* sm, bool init)
    {
        auto header = reinterpret_cast<PluginLogWriterHeader*>(sm->sh_mem_header);

        if (init)
            initHeader(header);

        return true;
    }

Эта функция вызывается, когда кто-то (процесс) открывает этот файл.
Если файл уже открыт, т.е. существует, то переменная `init` будет false. Если же файл новый, т.е. его никто не держит, то необходимо инициализировать заголовок.

Безопасная запись в файл
~~~~~~~~~~~~~~~~~~~~~~~~~
Запись происходит по адресу заголовка. Следует вначале установить `writePos`, равным размеру заголовка.
При создании экземпляра указывается начальный размер открываемой области.

.. code-block:: cpp

    Firebird::SharedMemory<TraceLogHeader> sharedFile(fileName.c_str(), INIT_SIZE, this);

Тут важная особенность: если файл большего размера, то откроется только область `INIT_SIZE`. При выходе за границы может произойти ошибка. Поэтому перед записью рекомендуется всегда проверять, соответствует ли открытая область размеру всего файла. При этом размер всего файла нигде не хранится, поэтому его нужно записывать в заголовке и при обращении к файлу обновлять через функцию `remapFile`.
Пример:

.. code-block:: cpp

    if (header->allocated != m_sharedMemory->sh_mem_length_mapped)
    {
        LocalStatus ls;
        CheckStatusWrapper s(&ls);

        if (!m_sharedMemory->remapFile(&s, header->allocated, false))
            status_exception::raise(&s);

        header = m_sharedMemory->getHeader();

        fb_assert(header->allocated == m_sharedMemory->sh_mem_length_mapped);
    }

Изменение размера файла
~~~~~~~~~~~~~~~~~~~~~~~~~
Изменить размер файла можно через метод `remapFile`. Причём важно, что у этой функции есть 2 режима работы (булевский флаг):

1. `false` - Изменение мапленой области в памяти;
2. `true` - Изменение размера всего файла.

Важно не путать эти режимы. Изменение размера всего файла должно выполняться через `remapFile(true)`. Режим с `false` должен вызываться перед чтением/записью файла, чтобы размер мапленой области совпадал с физическим размером.
Также нельзя уменьшать размер файла, так как это может не работать, если кто-то другой держит область, которую хотите уменьшить. Т.е. в идеале, нужно чтобы все процессы освободили эту область (`remapFile(false)`), а потом уже менять физический размер (`remapFile(true)`).
На практике лучше вообще никогда не уменьшать размер файла, так как с этим могут возникнуть проблемы на Windows.

Конфликты
~~~~~~~~~~~~~~
Для предотвращения конфликтов записи/чтения необходимо использовать механизмы блокировки. Они вызывают методы `lock` и `unlock` у файла `sharedMemory`. Самая простая реализация:

.. code-block:: cpp

    class TraceLogGuard
    {
    public:
        explicit TraceLogGuard(TraceLog* log) : m_log(*log)
        {
            m_log.lock();
        }

        ~TraceLogGuard()
        {
            m_log.unlock();
        }

    private:
        TraceLog& m_log;
    };

В методе `lock` необходимо вызвать:

.. code-block:: cpp

    m_sharedMemory->mutexLock();

и в 'unlock'

.. code-block:: cpp

    m_sharedMemory->mutexUnlock();

Удаление файла не происходит автоматически, необходимо вызвать метод `removeMapFile`. Чтобы понять, когда именно удалять файл, можно вести учёт использовании в заголовке. Однако с этим есть риск.
В момент между созданием файла и получением блокировки может возникнуть серая зона, и в этот момент другой процесс/поток может удалить файл, думая, что он не занят.