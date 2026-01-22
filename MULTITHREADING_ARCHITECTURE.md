# Multi-Threading Architecture - MaNGOS TBC

## Visão Geral

Este documento descreve a arquitetura de multi-threading do projeto MaNGOS-TBC, um emulador de servidor para World of Warcraft: The Burning Crusade. O sistema foi projetado para alta performance com processamento paralelo de mapas, operações assíncronas de banco de dados, e I/O de rede não-bloqueante.

---

## 1. Bibliotecas e Frameworks de Threading

### Biblioteca Principal: C++ Standard Library

**Localização**: `src/shared/Multithreading/Threading.h`

A biblioteca padrão C++ (`std::thread`) é a base do sistema de threading:

```cpp
class Thread {
    Thread(Runnable* instance);
    bool wait();                    // Join thread
    void destroy();                 // Force join
    void setPriority(Priority p);   // Define prioridade da thread
    static void Sleep(unsigned long msecs);
    static std::thread::id currentId();
private:
    std::thread m_ThreadImp;
    std::thread::id m_iThreadId;
    Runnable* const m_task;
};
```

**Características**:
- Wrapper customizado `MaNGOS::Thread` encapsulando `std::thread`
- Suporte a prioridades de thread (Windows nativo)
- Interface `Runnable` para tarefas executáveis

### Bibliotecas Secundárias

**Boost.Thread**:
- **Localização**: `src/shared/Util/TSS.h`
- Usado para Thread-Specific Storage via `boost::thread_specific_ptr`
- Wrapper customizado `MaNGOS::thread_local_ptr`

**Boost.Asio**:
- Usado para I/O assíncrono e networking
- `boost::asio::io_context` para event-driven I/O
- Múltiplas instâncias de io_context para diferentes subsistemas

---

## 2. Implementações de Thread Pool

### MapUpdater Thread Pool

**Localização**: `src/game/Maps/MapUpdater.h` e `.cpp`

**Propósito**: Atualizações paralelas de mapas para processamento do mundo do jogo

**Estrutura**:
```cpp
class MapUpdater {
    ProducerConsumerQueue<Worker*> _queue;
    std::vector<std::thread> _workerThreads;
    std::atomic<bool> _cancelationToken;
    std::mutex _lock;
    std::condition_variable _condition;
    size_t pending_requests;
};
```

**Características**:
- Número configurável de threads via `CONFIG_UINT32_NUM_MAP_THREADS`
- Workers incluem: `MapUpdateWorker`, `GridCrawler`, `ObjectUpdateWorker`
- Padrão producer-consumer com work-stealing
- Sincronização via condition variables para espera de conclusão

**Fluxo de Trabalho**:
```cpp
void MapUpdater::schedule_update(Worker* worker) {
    std::lock_guard<std::mutex> lock(_lock);
    ++pending_requests;
    _queue.Push(std::move(worker));
}

void MapUpdater::WorkerThread() {
    while (true) {
        Worker* request = nullptr;
        _queue.WaitAndPop(request);  // Bloqueia até ter trabalho

        if (_cancelationToken) {
            delete request;
            return;
        }

        request->execute();
        delete request;
    }
}
```

### Database Delay Thread

**Localização**: `src/shared/Database/SqlDelayThread.h`

- Thread de background única por conexão de banco de dados
- Processa operações SQL assíncronas
- Usa interface `MaNGOS::Runnable`
- Fila de operações SQL processadas sequencialmente

### Metrics System Thread Pool

**Localização**: `src/shared/Metric/Metric.h`

Duas threads dedicadas:
- `m_queueServiceThread`: Gerencia fila de métricas
- `m_writeServiceThread`: Persiste métricas
- Usa Boost.Asio io_context para operações assíncronas

---

## 3. Mecanismos de Mutex e Sincronização

### Tipos de Mutex Utilizados

#### 1. std::mutex (Mais Comum)

Usado em: `ObjectAccessor`, `Log`, `Config`, `MoveMap`, `VMapManager2`

Exemplos:
```cpp
std::mutex m_queueLock;        // ProducerConsumerQueue
std::mutex m_messageMutex;     // Messager
std::mutex m_closeMutex;       // AsyncSocket
std::mutex m_worldLogMtx;      // Sistema de Log
```

#### 2. std::recursive_mutex

Usado em: `MapManager`, `SqlConnection`, `Database`

Permite que a mesma thread faça lock múltiplas vezes:
```cpp
class MapManager : public MaNGOS::Singleton<MapManager,
    MaNGOS::ClassLevelLockable<MapManager, std::recursive_mutex>>
```

#### 3. Lock Guards

- `std::lock_guard<std::mutex>`: Locking baseado em RAII
- `std::unique_lock<std::mutex>`: Para uso com condition variables
- `GeneralLock<MUTEX>`: Template customizado em ThreadingModel.h

### Primitivas de Sincronização

#### Condition Variables

**Localização**: `ProducerConsumerQueue`, `MapUpdater`

Usado para sinalização e espera entre threads:
```cpp
std::condition_variable m_condition;

// Thread esperando trabalho
m_condition.wait(lock);

// Sinalizar um worker
m_condition.notify_one();

// Sinalizar todos workers
m_condition.notify_all();
```

**Exemplo no MapUpdater**:
```cpp
void MapUpdater::wait() {
    std::unique_lock<std::mutex> lock(_lock);
    while (pending_requests > 0)
        _condition.wait(lock);  // Espera todo trabalho completar
}

void MapUpdater::update_finished() {
    std::lock_guard<std::mutex> lock(_lock);
    --pending_requests;
    _condition.notify_all();  // Notifica threads esperando
}
```

#### Variáveis Atômicas

- **std::atomic<bool>**: Tokens de cancelamento, flags de shutdown
- **std::atomic<uint32>**: Contadores, IDs de instância
- **std::atomic_long**: Contagem de referências

```cpp
std::atomic<bool> m_shutdown;           // ProducerConsumerQueue
std::atomic<bool> _cancelationToken;    // MapUpdater
std::atomic<uint32> i_MaxInstanceId;    // MapManager
std::atomic_long m_refs;                // Runnable
```

---

## 4. Criação e Gerenciamento de Threads

### Padrão Runnable

**Localização**: `src/shared/Multithreading/Threading.h`

```cpp
class Runnable {
    virtual void run() = 0;
    void incReference();
    void decReference();
private:
    std::atomic_long m_refs;  // Contagem de referências
};
```

### Exemplos de Ciclo de Vida

**Threads Principais do Sistema**:
- **WorldRunnable**: Thread principal de atualização do mundo do jogo
- **FreezeDetectorRunnable**: Thread watchdog para detectar travamentos
- **SqlDelayThread**: Thread de operações assíncronas de banco de dados
- Worker threads no pool do MapUpdater

**Criação de Thread**:
```cpp
Runnable* task = new MyRunnable();
Thread* thread = new Thread(task);
// Thread inicia automaticamente

// Aguardar conclusão
thread->wait();
```

---

## 5. Estruturas de Dados Thread-Safe e Padrões

### ProducerConsumerQueue (Fila Thread-Safe)

**Localização**: `src/shared/Util/ProducerConsumerQueue.h`

Template genérico que funciona com qualquer tipo:

```cpp
template <typename T>
class ProducerConsumerQueue {
    void Push(const T&& value);     // Producer adiciona trabalho
    bool Pop(T& value);             // Tenta obter trabalho (não-bloqueante)
    void WaitAndPop(T& value);      // Bloqueia até trabalho disponível
    void Cancel();                  // Shutdown da fila
private:
    std::mutex m_queueLock;
    std::queue<T> m_queue;
    std::condition_variable m_condition;
    std::atomic<bool> m_shutdown;
};
```

**Implementação do WaitAndPop**:
```cpp
void WaitAndPop(T& value) {
    std::unique_lock<std::mutex> lock(m_queueLock);

    // Espera até ter item ou shutdown
    while (m_queue.empty() && !m_shutdown)
        m_condition.wait(lock);

    if (m_queue.empty() || m_shutdown)
        return;

    value = std::move(m_queue.front());
    m_queue.pop();
}
```

### HashMapHolder (Hash Map Thread-Safe)

**Localização**: `src/game/Globals/ObjectAccessor.h`

```cpp
template <class T>
class HashMapHolder {
    typedef std::mutex LockType;
    typedef std::lock_guard<std::mutex> ReadGuard;
    typedef std::lock_guard<std::mutex> WriteGuard;

    static void Insert(T* o);
    static void Remove(T* o);
    static T* Find(ObjectGuid guid);
private:
    static LockType i_lock;
    static MapType m_objectMap;
};
```

**Uso**: Acesso thread-safe a jogadores, corpses e outros objetos do jogo.

### Messager Pattern (Fila de Mensagens Thread-Safe)

**Localização**: `src/shared/Multithreading/Messager.h`

Execução de funções cross-thread:

```cpp
template <class T>
class Messager {
    void AddMessage(const std::function<void(T*)>& message);
    void Execute(T* object);
private:
    std::vector<std::function<void(T*)>> m_messageVector;
    std::mutex m_messageMutex;
};
```

**Técnica de Double-Buffering**: Troca buffers para reduzir contenção de locks durante execução.

---

## 6. Padrões Producer-Consumer e Filas de Trabalho

### Fila de Operações de Banco de Dados

```cpp
class SqlDelayThread {
    bool Delay(SqlOperation* sql) {
        std::lock_guard<std::mutex> guard(m_queueMutex);
        m_sqlQueue.push(std::unique_ptr<SqlOperation>(sql));
        return true;
    }

    void run() {
        while (m_running) {
            // Processa operações SQL da fila
            std::unique_ptr<SqlOperation> op;
            {
                std::lock_guard<std::mutex> guard(m_queueMutex);
                if (!m_sqlQueue.empty()) {
                    op = std::move(m_sqlQueue.front());
                    m_sqlQueue.pop();
                }
            }
            if (op)
                op->Execute(m_dbConnection);
        }
    }
private:
    std::mutex m_queueMutex;
    std::queue<std::unique_ptr<SqlOperation>> m_sqlQueue;
    std::atomic<bool> m_running;
};
```

### Fila de Resultados de Queries Assíncronas

```cpp
class SqlResultQueue {
    void Add(MaNGOS::IQueryCallback* callback);
    void Update();  // Processa callbacks na thread principal
private:
    std::mutex m_mutex;
    std::queue<std::unique_ptr<MaNGOS::IQueryCallback>> m_queue;
};
```

**Sistema de Callbacks**:
```cpp
template<class Class>
bool AsyncQuery(Class* object,
                void (Class::*method)(QueryResult*),
                const char* sql);
```

---

## 7. Mecanismos de Comunicação entre Threads

### 1. Sinalização via Condition Variables

Ver exemplo do MapUpdater na seção de Condition Variables.

### 2. Message Passing (Messager)

- Chamadas de função cross-thread
- Double-buffering para evitar locks durante execução
- Mensagens acumuladas e executadas em lote

### 3. Flags Atômicas

```cpp
std::atomic<bool> m_allowAsyncTransactions;  // Database
std::atomic<bool> m_shutdown;                // ProducerConsumerQueue
std::atomic<bool> m_isWriting;               // AsyncSocket
```

### 4. Sistema de Callbacks

Queries assíncronas de banco de dados com callbacks:
```cpp
// Query executada em thread de DB, callback na thread principal
CharacterDatabase.AsyncPQuery(this, &Class::HandleQueryResult,
    "SELECT * FROM characters WHERE guid = %u", guid);
```

---

## 8. Seções Críticas e Hierarquia de Locks

### Sistema de Locking Baseado em Políticas

**Localização**: `src/framework/Policies/ThreadingModel.h`

#### Três Modelos de Threading

**1. SingleThreaded** (No-op locks para objetos single-threaded)
```cpp
template<class T>
class SingleThreaded {
    struct Lock {
        Lock() {}
        Lock(const T&) {}
    };
};
```

**2. ObjectLevelLockable** (Mutex por instância)
```cpp
template<class T, class MUTEX>
class ObjectLevelLockable {
    class Lock {
        Lock(ObjectLevelLockable<T, MUTEX>& host)
            : i_lock(host.i_mtx) {}
    private:
        GeneralLock<MUTEX> i_lock;
    };
private:
    MUTEX i_mtx;  // Cada objeto tem seu próprio mutex
};
```

**3. ClassLevelLockable** (Mutex estático compartilhado)
```cpp
template<class T, class MUTEX>
class ClassLevelLockable {
    class Lock {
        Lock(const T& /*host*/) {
            ClassLevelLockable<T, MUTEX>::si_mtx.lock();
        }
        ~Lock() {
            ClassLevelLockable<T, MUTEX>::si_mtx.unlock();
        }
    };
private:
    static MUTEX si_mtx;  // Compartilhado por todas as instâncias
};
```

### Thread Safety de Singletons

**Padrão Double-Checked Locking**:
```cpp
class MapManager : public MaNGOS::Singleton<MapManager,
    MaNGOS::ClassLevelLockable<MapManager, std::recursive_mutex>>
```

Aplicado a: `MapManager`, `ObjectAccessor`, `Log`

### Helpers de Lock

**Macros úteis**:
```cpp
// Lock.h
#define GUARD_RETURN(mutex, retval) \
    if (!mutex.try_lock()) return retval; \
    std::lock_guard<decltype(mutex)> guard(mutex, std::adopt_lock)
```

### SqlConnection Nested Lock

```cpp
class SqlConnection {
    class Lock {
        Lock(SqlConnection* conn) : m_pConn(conn) {
            m_pConn->m_mutex.lock();
        }
        ~Lock() { m_pConn->m_mutex.unlock(); }
    };
private:
    std::recursive_mutex m_mutex;  // Recursivo para chamadas aninhadas
};
```

---

## 9. Padrões de Threading Resumo

| Padrão | Descrição | Localização |
|--------|-----------|-------------|
| **Thread Pool** | MapUpdater com workers configuráveis | `Maps/MapUpdater.*` |
| **Producer-Consumer** | ProducerConsumerQueue para distribuição de trabalho | `Util/ProducerConsumerQueue.h` |
| **Active Object** | SqlDelayThread processa comandos assincronamente | `Database/SqlDelayThread.*` |
| **Double-Buffering** | Messager troca buffers para reduzir contenção | `Multithreading/Messager.h` |
| **Thread-Specific Storage** | boost::thread_specific_ptr para dados per-thread | `Util/TSS.h` |
| **Reference Counting** | Contagem de referências atômica em Runnable | `Multithreading/Threading.h` |
| **RAII Locking** | Uso extensivo de lock_guard e classes Lock customizadas | Vários |
| **Policy-Based Design** | Modelos de threading configuráveis via templates | `Policies/ThreadingModel.h` |
| **Singleton Thread Safety** | ClassLevelLockable para singletons | Vários |
| **Async I/O** | Boost.Asio para operações de rede não-bloqueantes | `Network/` |

---

## 10. Componentes Thread-Safe Notáveis

### Sistema de Banco de Dados
- **Localização**: `src/shared/Database/`
- Transações assíncronas
- Connection pools
- Filas de resultados thread-safe
- Operações atrasadas para evitar contenção

### Sistema de Mapas
- **Localização**: `src/game/Maps/`
- Atualizações paralelas de mapas
- Gerenciamento thread-safe de instâncias
- Workers especializados (MapUpdateWorker, GridCrawler, ObjectUpdateWorker)

### Gerenciamento de Objetos
- **Localização**: `src/game/Globals/ObjectAccessor.*`
- Lookups thread-safe de players/corpses via HashMapHolder
- Hash maps com read/write guards

### Sistema de Logging
- **Localização**: `src/shared/Log.*`
- Escritas de log thread-safe com mutexes
- Logs assíncronos para reduzir latência

### Sistema de Configuração
- **Localização**: `src/shared/Config/`
- Acesso thread-safe a configurações
- Leitura concorrente segura

### Camada de Rede
- **Localização**: `src/game/Server/`
- Sockets assíncronos Boost.Asio
- Operações de close thread-safe
- Múltiplos io_contexts para balanceamento

### VMAP/MoveMap
- **Localização**: `src/game/vmap/`, `src/game/MotionGenerators/`
- Queries thread-safe de terreno/pathfinding
- Dados compartilhados imutáveis para leitura concorrente

---

## 11. Considerações de Performance

### Minimização de Contenção
- Double-buffering no Messager
- Lock-free quando possível (atomic operations)
- Granularidade fina de locks (object-level vs class-level)

### Configurabilidade
- Número de threads do MapUpdater configurável
- Threads de I/O assíncronas escaláveis

### Work Stealing
- MapUpdater usa fila compartilhada
- Workers pegam trabalho dinamicamente

### Async I/O
- Boost.Asio evita bloqueio em operações de rede
- Callbacks processados em threads dedicadas

---

## 12. Prevenção de Deadlock

### Estratégias Implementadas

1. **Ordem de Lock Consistente**: Hierarquia implícita de locks
2. **Try-Lock com Timeout**: Macro GUARD_RETURN
3. **Lock de Escopo Curto**: RAII garante release automático
4. **Recursive Mutex**: Para componentes que precisam re-lock
5. **Lock-Free Atomics**: Para operações simples sem mutex

### Exemplo de Try-Lock
```cpp
bool TryExecute() {
    GUARD_RETURN(m_mutex, false);
    // Executa com lock, retorna false se não conseguir lock
    DoWork();
    return true;
}
```

---

## 13. Debugging e Diagnóstico

### FreezeDetectorRunnable
- **Localização**: `src/game/World/World.cpp`
- Detecta travamentos da thread principal
- Registra stack traces em caso de freeze

### Thread IDs
- Todas as threads armazenam `std::thread::id`
- Útil para logs e debugging
- `Thread::currentId()` para identificar thread atual

### Assertions Thread-Safe
- Macros de assert consideram contexto de threading
- Logs incluem informações de thread

---

## Conclusão

A arquitetura de multi-threading do MaNGOS-TBC demonstra um design maduro e pronto para produção, com atenção cuidadosa a:

- **Sincronização**: Uso adequado de mutexes, atomics e condition variables
- **Prevenção de Deadlock**: Estratégias múltiplas para evitar travamentos
- **Otimização de Performance**: Thread pools, async I/O, work stealing
- **Segurança**: RAII locking, thread-safe containers, atomic operations
- **Flexibilidade**: Policy-based design, configurabilidade

O sistema escala eficientemente para processamento paralelo de mapas, operações assíncronas de banco de dados, e I/O de rede não-bloqueante, essenciais para um servidor de MMORPG de alta performance.

---

**Última Atualização**: 2026-01-22
**Projeto**: MaNGOS-TBC
**Repositório**: https://github.com/cmangos/mangos-tbc
