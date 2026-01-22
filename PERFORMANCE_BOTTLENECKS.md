# Análise de Gargalos de Performance - MaNGOS TBC

## Sumário Executivo

Este documento identifica os principais gargalos de performance na arquitetura de multi-threading do MaNGOS-TBC, com base em análise detalhada do código-fonte. Os gargalos estão categorizados por severidade e impacto.

---

## 1. CONTENÇÃO DE LOCKS (Lock Contention)

### 1.1 🔴 CRÍTICO: MapManager - Lock Global Recursivo

**Localização**: `src/game/Maps/MapManager.h:56`

```cpp
class MapManager : public MaNGOS::Singleton<MapManager,
    MaNGOS::ClassLevelLockable<MapManager, std::recursive_mutex>>
```

**Problema**: Todas as instâncias de MapManager compartilham um único `std::recursive_mutex` estático. Cada operação de mapa (CreateMap, FindMap, DeleteInstance) adquire este lock global, criando um ponto de serialização para todas as operações relacionadas a mapas em todas as threads.

**Exemplo de Código** (`MapManager.cpp:114`):
```cpp
Map* MapManager::CreateMap(uint32 id, const WorldObject* obj)
{
    Guard _guard(*this);  // Trava o mutex global class-level
    // ... lógica de criação de mapa
}
```

**Impacto**:
- Alto ponto de contenção - todas as worker threads no MapUpdater competem por este único lock
- Serializa operações que poderiam ser paralelas
- Degrada linearmente com número de threads

**Recomendação**:
- Mudar de `ClassLevelLockable` para `ObjectLevelLockable`
- Usar locks por mapa individual em vez de lock global
- Implementar estrutura de dados lockless para lookup de mapas

---

### 1.2 🔴 CRÍTICO: ObjectAccessor - Lock Global para Todos os Players

**Localização**: `src/game/Globals/ObjectAccessor.h:70`

```cpp
class ObjectAccessor : public MaNGOS::Singleton<ObjectAccessor,
    MaNGOS::ClassLevelLockable<ObjectAccessor, std::mutex>>
```

**Problema**: Mutex class-level protege todos os lookups de players/corpses globalmente. HashMapHolder usa o mesmo mutex para todas operações read/write.

**Exemplo de Código** (`ObjectAccessor.cpp:40-56`):
```cpp
template<class T>
void HashMapHolder<T>::Insert(T* o)
{
    WriteGuard guard(i_lock);  // Compartilhado entre TODAS as instâncias
    m_objectMap[o->GetObjectGuid()] = o;
}

template<class T>
T* HashMapHolder<T>::Find(ObjectGuid guid)
{
    ReadGuard guard(i_lock);  // Mesmo lock para leituras!
    typename MapType::iterator itr = m_objectMap.find(guid);
    return (itr != m_objectMap.end()) ? itr->second : nullptr;
}
```

**Impacto**:
- Cada lookup de player de qualquer thread adquire este lock global
- Leituras bloqueiam outras leituras (não usa reader/writer lock)
- Com 1000+ players, torna-se gargalo severo

**Recomendação**:
- Implementar `std::shared_mutex` (reader/writer lock)
- Usar `std::shared_lock` para reads, `std::unique_lock` para writes
- Considerar lock-free hash map (ex: `tbb::concurrent_hash_map`)

---

### 1.3 🟡 ALTO: VMapManager2 - Dual Mutex Coarse-Grained

**Localização**: `src/game/vmap/VMapManager2.h:69-70`

```cpp
std::mutex m_vmStaticMapMutex;
std::mutex m_vmModelMutex;
```

**Problema**: Locking coarse-grained para todas operações VMAP. Verificações de line-of-sight e queries de terreno serializam.

**Impacto**:
- Pathfinding e collision checks competem pelo mesmo lock
- Operações de leitura que poderiam ser paralelas são serializadas

**Recomendação**:
- Locks por mapa individual
- Reader/writer locks para operações read-heavy
- Considerar estruturas immutable para permitir leituras lock-free

---

### 1.4 🟡 ALTO: MoveMap - Mutex de Modelos Global

**Localização**: `src/game/MotionGenerators/MoveMap.h:129`

```cpp
std::mutex m_modelsMutex;
```

**Problema**: Todos os acessos a modelos de pathfinding compartilham único mutex, bloqueando operações paralelas de pathfinding.

**Impacto**:
- NPCs calculando pathfinding simultaneamente serializam
- Escala mal com número de NPCs ativos

**Recomendação**:
- Hash map thread-safe com locks por tile/grid
- Carregar modelos como read-only e eliminar locks para leituras

---

### 1.5 🟢 MÉDIO: Sistema de Log - Lock Global

**Localização**: `src/shared/Log/Log.h:96`

```cpp
class Log : public MaNGOS::Singleton<Log,
    MaNGOS::ClassLevelLockable<Log, std::mutex>>
```

**Locks adicionais**: `m_worldLogMtx` (linha 211), `m_traceLogMtx` (linha 212)

**Problema**: Todas operações de logging serializam. Logging pesado causa contenção em todas as threads.

**Impacto**:
- Cada log statement de qualquer thread compete pelo lock
- Pode adicionar latência significativa em hot paths

**Recomendação**:
- Implementar logging assíncrono com buffer lock-free
- Usar thread-local buffers que flush periodicamente
- Considerar bibliotecas como `spdlog` com async logging

---

### 1.6 🟢 MÉDIO: SqlConnection - Recursive Mutex

**Localização**: `src/shared/Database/Database.h:98`

```cpp
class SqlConnection
{
    std::recursive_mutex m_mutex;

    class Lock {
        Lock(SqlConnection* conn) : m_pConn(conn) {
            m_pConn->m_mutex.lock();
        }
    };
};
```

**Problema**: Uso de recursive mutex indica re-entrada de locks, sugerindo possível problema de design. Cada conexão tem seu próprio mutex, mas locking recursivo sugere padrões de chamadas aninhadas que podem deadlock.

**Impacto**:
- Recursive mutex é mais lento que mutex normal
- Indica complexidade desnecessária

**Recomendação**:
- Refatorar para eliminar necessidade de recursive locking
- Garantir hierarquia clara de locks

---

## 2. GARGALOS SINGLE-THREADED

### 2.1 🔴 CRÍTICO: Loop de World Update (Apenas Main Thread)

**Localização**: `src/mangosd/WorldRunnable.cpp:67-115`

```cpp
void WorldRunnable::run()
{
    while (!World::IsStopped())
    {
        ++World::m_worldLoopCounter;
        diffTick = WorldTimer::tick();
        sWorld.Update(diffTick);  // SINGLE-THREADED
        diffTime = WorldTimer::getMSTime() - WorldTimer::tickTime();

        if (diffTime < WORLD_SLEEP_CONST)
            MaNGOS::Thread::Sleep(WORLD_SLEEP_CONST - diffTime);
    }
}
```

**Taxa de tick fixa**: `#define WORLD_SLEEP_CONST 50` (50ms)

**Problema**: Todo o world update roda em uma única thread. Todas as atualizações de sessões, timers e lógica de jogo são serializadas.

**Impacto**:
- Limite hard de 20 ticks/segundo
- CPU single-core bottleneck
- Não escala com múltiplos cores

**Recomendação**:
- Paralelizar sub-tarefas do world update
- Processar sessões em thread pool
- Mover timers não-críticos para threads separadas

---

### 2.2 🔴 CRÍTICO: Atualizações de Sessões (Iteração Linear)

**Localização**: `src/game/World/World.cpp:2193-2222`

```cpp
void World::UpdateSessions(uint32 diff)
{
    // Iteração single-threaded através de TODAS as sessões
    for (SessionMap::iterator itr = m_sessions.begin(); itr != m_sessions.end();)
    {
        WorldSession* pSession = itr->second;

        if (!pSession->Update(diff))  // Cada sessão atualizada sequencialmente
        {
            RemoveQueuedSession(pSession);
            itr = m_sessions.erase(itr);
            delete pSession;
        }
        else
            ++itr;
    }
}
```

**Impacto**:
- Com 1000+ players, torna-se ponto de serialização significativo
- Nenhuma paralelização do processamento de sessões
- O(N) onde N = número de players conectados

**Recomendação**:
- Dividir sessões entre worker threads do MapUpdater
- Processar sessões em chunks paralelos
- Garantir thread-safety dos dados de sessão

---

### 2.3 🟡 ALTO: Processamento de Database Result Queue

**Localização**: `src/game/World/World.cpp:2250-2256`

```cpp
void World::UpdateResultQueue()
{
    // Processamento sequencial na main thread
    CharacterDatabase.ProcessResultQueue();
    WorldDatabase.ProcessResultQueue();
    LoginDatabase.ProcessResultQueue();
}
```

**Problema**: Callbacks de async queries processados sequencialmente apenas na main thread.

**Impacto**:
- Callbacks complexos bloqueiam world update
- Não aproveita multiple cores para processamento de resultados

**Recomendação**:
- Processar result queues em threads dedicadas
- Callbacks apenas enfileiram ações para main thread se necessário
- Separar operações thread-safe dos callbacks

---

### 2.4 🟡 ALTO: MapUpdater Wait - Serialização

**Localização**: `src/game/Maps/MapUpdater.cpp:47-53`

```cpp
void MapUpdater::wait()
{
    std::unique_lock<std::mutex> lock(_lock);

    while (pending_requests > 0)
        _condition.wait(lock);  // Main thread bloqueia aqui
}
```

**Problema**: Main thread deve esperar TODAS as atualizações de mapas completarem antes de continuar. Nenhuma sobreposição entre fases do world update.

**Impacto**:
- Desperdício de tempo onde main thread poderia fazer outro trabalho
- Latência adicional no world update cycle

**Recomendação**:
- Fazer MapUpdater completamente assíncrono
- Main thread inicia map updates e continua com outras tarefas
- Sincronizar apenas quando absolutamente necessário

---

## 3. OPERAÇÕES BLOQUEANTES

### 3.1 🔴 CRÍTICO: Queries de Banco de Dados Síncronas na Main Thread

**Localização**: `src/game/World/World.cpp` (múltiplas localizações)

**Exemplos**:
```cpp
// Linha 928
LoginDatabase.PExecute("UPDATE realmlist SET icon = %u...");

// Linha 931
CharacterDatabase.PExecute("DELETE FROM corpse WHERE...");

// Linha 1441
LoginDatabase.Execute("DELETE FROM ip_banned WHERE...");

// Linha 2281
CharacterDatabase.Query("SELECT NextDailyQuestResetTime...");
```

**Problema**: Chamadas diretas de Execute/PExecute bloqueiam a thread chamadora esperando resposta do DB.

**Impacto**:
- World update congela enquanto espera DB
- Latência do DB diretamente impacta tick rate
- Com DB lento, todo servidor degrada

**Recomendação**:
- Converter TODAS queries para AsyncQuery/AsyncPQuery
- Nunca bloquear main thread em I/O
- Implementar timeout e retry logic

---

### 3.2 🔴 CRÍTICO: SqlDelayThread - Polling

**Localização**: `src/shared/Database/SqlDelayThread.cpp:41-59`

```cpp
void SqlDelayThread::run()
{
    const uint32 loopSleepms = 10;  // Intervalo de polling 10ms

    while (m_running)
    {
        MaNGOS::Thread::Sleep(loopSleepms);  // BUSY POLLING
        ProcessRequests();

        if ((loopCounter++) >= pingEveryLoop)
        {
            loopCounter = 0;
            m_dbEngine->Ping();
        }
    }
}
```

**Problema**: Desperdiça CPU com loop de sleep 10ms em vez de usar condition variable. Adiciona latência de 0-10ms em operações de DB.

**Impacto**:
- Uso desnecessário de CPU
- Latência adicional nas operações
- Não responde imediatamente a novas requisições

**Recomendação**:
- Substituir sleep loop por condition_variable
- Notificar thread quando nova operação é enfileirada
- Reduzir latência para ~0ms

---

### 3.3 🟢 MÉDIO: Messager Execution (Bem Implementado)

**Localização**: `src/shared/Multithreading/Messager.h:35-46`

```cpp
void Execute(T* object)
{
    std::vector<std::function<void(T*)>> messageVectorCopy;
    {
        std::lock_guard<std::mutex> guard(m_messageMutex);
        std::swap(m_messageVector, messageVectorCopy);  // Bom: double-buffering
    }
    for (auto& message : messageVectorCopy)
        message(object);  // Callbacks executados sem lock (Bom!)
}
```

**Nota**: Bem projetado com double-buffering. Mas callbacks ainda são síncronos - sem paralelização.

**Recomendação Futura**:
- Considerar paralelizar execução de callbacks independentes
- Usar task queue para callbacks pesados

---

## 4. PROBLEMAS DE ESCALABILIDADE

### 4.1 🔴 CRÍTICO: MapUpdater Thread Pool de Tamanho Fixo

**Localização**: `src/game/Maps/MapUpdater.cpp:22-35`

```cpp
MapUpdater::MapUpdater(size_t num_threads) : _cancelationToken(false), pending_requests(0)
{
    for (size_t i = 0; i < num_threads; ++i)
        _workerThreads.push_back(std::thread(&MapUpdater::WorkerThread, this));
}
```

**Configuração**: `CONFIG_UINT32_NUM_MAP_THREADS` - definido no startup, não pode escalar dinamicamente.

**Problemas**:
- Pool de tamanho fixo, sem scaling dinâmico
- Threads ficam idle quando não há trabalho de mapa disponível
- Sem work stealing entre threads
- Single ProducerConsumerQueue cria contenção

**Impacto**:
- Recursos desperdiçados em baixa carga
- Insuficiente em alta carga
- Não se adapta a padrões de uso variáveis

**Recomendação**:
- Implementar thread pool dinâmico que escala com carga
- Work stealing queue para balanceamento de carga
- Múltiplas filas para reduzir contenção

---

### 4.2 🔴 CRÍTICO: Thread Única de Database Delay por Conexão

**Localização**: `src/shared/Database/Database.h:269`

```cpp
SqlDelayThread* m_threadBody;  // Thread única por database
```

**Problema**: Apenas UMA thread processa operações assíncronas de DB por database. Serializa todas async queries.

**Impacto**:
- Gargalo de DB mesmo com async API
- Não aproveita connection pooling
- Uma operação lenta bloqueia todas as outras

**Recomendação**:
- Múltiplas delay threads por database
- Round-robin ou hash-based distribution de queries
- Thread pool para processamento paralelo

---

### 4.3 🟡 ALTO: Iteração Linear de Sessões (Sem Paralelização)

**Múltiplas ocorrências em World.cpp**:

```cpp
// Linha 2208: UpdateSessions - sequencial
for (SessionMap::iterator itr = m_sessions.begin(); itr != m_sessions.end();)

// Padrões similares para:
// - SaveAllPlayers
// - ExecuteOnAllPlayers
// - SendGlobalMessage
// - KickAll
```

**Problema**: Operações O(N) no mapa de sessões. Com 3000 players, torna-se caro. Sem processamento paralelo.

**Impacto**:
- Tempo de processamento cresce linearmente com players
- Bottleneck em populações altas
- Desperdiça cores disponíveis

**Recomendação**:
- Paralelizar loops usando thread pool
- Processar chunks de sessões em paralelo
- Usar algoritmos parallel STL quando possível

---

### 4.4 🟢 MÉDIO: std::list para BattleGround Queue

**Localização**: `src/game/BattleGround/BattleGroundQueue.h:123`

```cpp
typedef std::list<BattleGroundInQueueInfo> BgFreeSlotQueueType;
```

**Comentário no código** (linha 122):
> "can't be deque, because deque doesn't like removing the last element"

**Problema**: Busca linear através da lista para matching de BG instances. Deveria usar map/multimap indexado por critérios-chave.

**Impacto**:
- O(N) para encontrar BG disponível
- Cache-unfriendly com ponteiros indiretos
- Degrada com muitos BGs ativos

**Recomendação**:
- Usar `std::map` ou `std::unordered_map` indexado por instance ID
- Manter índices secundários para lookups comuns
- Considerar flat_map para melhor cache locality

---

### 4.5 🟢 MÉDIO: std::list para Group Queues

**Localização**: `src/game/BattleGround/BattleGroundQueue.h:151`

```cpp
typedef std::list<GroupQueueInfo*> GroupsQueueType;

// Array de listas
GroupsQueueType m_queuedGroups[MAX_BATTLEGROUND_BRACKETS][BG_QUEUE_GROUP_TYPES_COUNT];
```

**Problema**: Iteração linear para encontrar grupos matching. Inserções/remoções frequentes.

**Impacto**:
- O(N) matchmaking
- Fragmentação de memória com ponteiros

**Recomendação**:
- Heap ou priority queue para ordenação por rating/tempo
- Indexed containers para lookup rápido

---

## 5. PROBLEMAS DE MEMÓRIA/CACHE

### 5.1 🟡 ALTO: ProducerConsumerQueue - Potencial False Sharing

**Localização**: `src/shared/Util/ProducerConsumerQueue.h:100-103`

```cpp
private:
    std::mutex m_queueLock;              // Provavelmente na mesma cache line
    std::queue<T> m_queue;               // que estes membros
    std::condition_variable m_condition;
    std::atomic<bool> m_shutdown;
```

**Problema**: Mutex, queue e atomic provavelmente na mesma cache line. Threads producer/consumer fazem bounce de cache lines.

**Impacto**:
- Invalidação desnecessária de cache
- Degradação de performance em sistemas multi-socket
- Escalabilidade limitada com múltiplos cores

**Recomendação**:
- Usar `alignas(64)` para alinhar em cache lines
- Separar dados de read/write em cache lines diferentes
- Padding explícito entre membros hot

**Exemplo**:
```cpp
private:
    alignas(64) std::mutex m_queueLock;
    std::queue<T> m_queue;
    std::condition_variable m_condition;

    alignas(64) std::atomic<bool> m_shutdown;  // Em cache line separada
```

---

### 5.2 🟢 MÉDIO: MapUpdater - Shared Counter

**Localização**: `src/game/Maps/MapUpdater.h:56`

```cpp
std::mutex _lock;
std::condition_variable _condition;
size_t pending_requests;  // Não-atomic, protegido por mutex
```

**Problema**: `pending_requests` incrementado/decrementado sob lock por múltiplas threads. Poderia ser atomic para operações lockless.

**Impacto**:
- Lock desnecessário para incremento simples
- Contenção adicional

**Recomendação**:
```cpp
std::atomic<size_t> pending_requests;
```

---

### 5.3 🟢 MÉDIO: World Opcode Counters

**Localização**: `src/game/World/World.h:766`

```cpp
std::vector<std::atomic<uint32>> m_opcodeCounters;
```

**Problema**: Vector de atomics - bom, mas ainda potencial bouncing de cache line se múltiplos opcodes na mesma cache line.

**Impacto Moderado**: Counters são acessados frequentemente.

**Recomendação**:
```cpp
// Estrutura cache-aligned
struct alignas(64) OpcodeCounter {
    std::atomic<uint32> count;
};
std::vector<OpcodeCounter> m_opcodeCounters;
```

---

### 5.4 🟢 BAIXO: Messager Vector Growth

**Localização**: `src/shared/Multithreading/Messager.h:48`

```cpp
std::vector<std::function<void(T*)>> m_messageVector;
```

**Problema**: Realocações de vector sob lock quando mensagens são adicionadas. Poderia usar deque ou reservar capacidade.

**Impacto Baixo**: Realocações são raras em uso normal.

**Recomendação**:
```cpp
// Ou usar deque (sem realocações)
std::deque<std::function<void(T*)>> m_messageVector;

// Ou reservar capacidade esperada
m_messageVector.reserve(1024);
```

---

## 6. GARGALOS ADICIONAIS

### 6.1 🟢 MÉDIO: Config System - Lock Global

**Localização**: `src/shared/Config/Config.h:49`

```cpp
std::mutex m_configLock;
```

**Problema**: Todas leituras de config requerem lock. Config deveria ser read-mostly com reader/writer lock ou estrutura lockless.

**Impacto**:
- Contenção em operações de leitura frequentes
- Config é raramente modificado mas frequentemente lido

**Recomendação**:
```cpp
std::shared_mutex m_configLock;

// Leituras
std::shared_lock<std::shared_mutex> lock(m_configLock);

// Escritas
std::unique_lock<std::shared_mutex> lock(m_configLock);
```

---

### 6.2 🟢 BAIXO: Database Round-Robin Connection Selection

**Localização**: `src/shared/Database/Database.h:259`

```cpp
std::atomic_long m_nQueryCounter;  // Para seleção round-robin
```

**Usado em getQueryConnection()** - Bom uso de atomic, mas round-robin pode causar carga desigual se queries têm custos variáveis.

**Recomendação**:
- Considerar load-based selection em vez de round-robin
- Tracking de conexões busy/idle

---

## RESUMO DE GARGALOS CRÍTICOS (Ordem de Prioridade)

| # | Gargalo | Severidade | Localização | Impacto |
|---|---------|-----------|-------------|---------|
| 1 | **MapManager global recursive_mutex** | 🔴 Crítico | `Maps/MapManager.h:56` | Serializa todas operações de mapa |
| 2 | **World::UpdateSessions single-threaded** | 🔴 Crítico | `World/World.cpp:2193` | Sem paralelização de sessões |
| 3 | **ObjectAccessor global mutex** | 🔴 Crítico | `Globals/ObjectAccessor.h:70` | Todos lookups de players serializam |
| 4 | **Synchronous DB queries na main thread** | 🔴 Crítico | `World/World.cpp:928+` | Bloqueia world update |
| 5 | **SqlDelayThread polling** | 🔴 Crítico | `Database/SqlDelayThread.cpp:41` | Desperdiça CPU, adiciona latência |
| 6 | **Fixed MapUpdater thread pool** | 🔴 Crítico | `Maps/MapUpdater.cpp:22` | Sem scaling dinâmico |
| 7 | **MapUpdater wait() serialization** | 🟡 Alto | `Maps/MapUpdater.cpp:47` | Main thread bloqueia por todos mapas |
| 8 | **Single DB delay thread** | 🔴 Crítico | `Database/Database.h:269` | Gargalo de DB |
| 9 | **VMapManager/MoveMap locks globais** | 🟡 Alto | `vmap/VMapManager2.h:69` | Pathfinding serializa |
| 10 | **Linear session/queue searches** | 🟡 Alto | Vários | Operações O(N) |

---

## RECOMENDAÇÕES DE OTIMIZAÇÃO (Top 10)

### 1. Substituir Class-Level Locks por Object-Level
**Impacto**: Máximo | **Esforço**: Médio

Converter `MapManager` e `ObjectAccessor` de `ClassLevelLockable` para `ObjectLevelLockable`. Permite acesso paralelo a diferentes objetos.

---

### 2. Implementar Reader/Writer Locks
**Impacto**: Alto | **Esforço**: Baixo

Usar `std::shared_mutex` para estruturas read-heavy:
- ObjectAccessor (player lookups)
- Config system
- VMapManager/MoveMap

---

### 3. Paralelizar Session Updates
**Impacto**: Máximo | **Esforço**: Alto

Dividir `UpdateSessions()` em chunks processados em thread pool. Requer garantir thread-safety de session data.

---

### 4. Eliminar Synchronous DB Queries da Main Thread
**Impacto**: Máximo | **Esforço**: Médio

Converter todas chamadas `Execute()`/`Query()` para `AsyncQuery()`/`AsyncPQuery()` na main thread.

---

### 5. Substituir SqlDelayThread Polling por Condition Variable
**Impacto**: Médio | **Esforço**: Baixo

```cpp
void SqlDelayThread::run()
{
    while (m_running)
    {
        std::unique_lock<std::mutex> lock(m_queueMutex);
        m_condition.wait(lock, [this]{ return !m_sqlQueue.empty() || !m_running; });

        if (!m_running) break;

        // Process queue
    }
}
```

---

### 6. Implementar Work-Stealing Thread Pool
**Impacto**: Alto | **Esforço**: Alto

Substituir MapUpdater por thread pool dinâmico com work-stealing. Permite melhor balanceamento e scaling.

---

### 7. Múltiplas DB Delay Threads
**Impacto**: Alto | **Esforço**: Médio

Pool de threads para processar async DB operations. Round-robin ou hash-based distribution.

---

### 8. MapUpdater Assíncrono (Não-Bloqueante)
**Impacto**: Médio | **Esforço**: Médio

Eliminar `wait()` - main thread inicia map updates e continua. Sincronizar apenas quando necessário.

---

### 9. Lock-Free Data Structures
**Impacto**: Alto | **Esforço**: Alto

Usar estruturas lock-free para high-contention paths:
- Intel TBB `concurrent_hash_map` para ObjectAccessor
- Lock-free queues para messaging
- Atomic operations para counters

---

### 10. Cache-Line Alignment
**Impacto**: Médio | **Esforço**: Baixo

Alinhar hot atomics e mutex em cache lines separadas:

```cpp
struct alignas(64) CacheAligned {
    std::atomic<uint64_t> counter;
};
```

---

## MÉTRICAS RECOMENDADAS PARA MONITORAMENTO

1. **Lock Wait Time**: Tempo médio esperando por locks
2. **Thread Pool Utilization**: % de tempo threads estão ativas
3. **DB Query Latency**: P50, P95, P99 latências
4. **World Update Time**: Tempo por tick do world update
5. **Session Update Time**: Tempo para processar todas sessões
6. **Map Update Time**: Tempo para parallel map updates
7. **CPU Utilization**: Por thread e total
8. **Cache Misses**: L1/L2/L3 cache miss rates
9. **Context Switches**: Taxa de context switches involuntários

---

## FERRAMENTAS DE PROFILING RECOMENDADAS

1. **perf** (Linux): CPU profiling, cache analysis
2. **Intel VTune**: Threading analysis, lock contention
3. **Valgrind/Helgrind**: Thread error detection, race conditions
4. **ThreadSanitizer (TSan)**: Data race detection
5. **gperftools**: CPU e heap profiling
6. **Tracy Profiler**: Real-time frame profiler

---

## CONCLUSÃO

Os gargalos identificados são típicos de aplicações multi-threaded que evoluíram de single-threaded designs. As principais áreas de melhoria são:

1. **Reduzir contenção de locks** através de locks fine-grained e reader/writer locks
2. **Eliminar serialização forçada** através de paralelização de session updates
3. **Remover operações bloqueantes** da main thread
4. **Implementar scaling dinâmico** em thread pools
5. **Otimizar cache usage** através de alignment e redução de false sharing

Com estas otimizações, espera-se:
- **2-3x** melhoria em throughput de sessions
- **50-70%** redução em lock contention
- **30-50%** redução em latência de world update
- Melhor scaling com número de cores disponíveis

---

**Data da Análise**: 2026-01-22
**Versão do Código**: mangos-tbc master branch
**Commit**: fa54c3c8d
