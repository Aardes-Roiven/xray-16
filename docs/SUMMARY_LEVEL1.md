## Уровень 1: Архитектурные паттерны (Design Patterns)

> **Для кого:** сеньор React‑разработчик, который уже освоил базовые концепции C++ из уровня 0  
> **Цель уровня:** понять основные паттерны проектирования, используемые в движке OpenXRay, чтобы видеть общую архитектуру и связи между компонентами. Для непонятных терминов используй [словарь знаний](KNOWLEDGE_GLOSSARY.md).

---

## 1.1. Observer Pattern (MessageRegistry)

### Концепция: что это такое и зачем нужно

#### Проблема, которую решает паттерн Observer

Представь ситуацию: у тебя есть центральный объект (например, игровой движок), который должен уведомлять множество других объектов о важных событиях. События могут быть разными:
- Начался новый кадр игры
- Нужно отрисовать сцену
- Игра потеряла фокус (пользователь переключился на другое окно)
- Сбросилось графическое устройство

**Проблема:** если движок будет напрямую вызывать методы всех заинтересованных объектов, получится:
- **Жёсткая связанность:** движок должен знать о каждом объекте, который хочет получать уведомления
- **Сложность расширения:** чтобы добавить нового слушателя, нужно менять код движка
- **Нарушение принципа единственной ответственности:** движок занимается не только своей логикой, но и управлением списками подписчиков

#### Решение: паттерн Observer

**Observer (Наблюдатель)** — это паттерн проектирования, который позволяет объектам подписываться на события и получать уведомления, когда эти события происходят.

**Ключевая идея:** вместо того, чтобы движок сам искал всех, кого нужно уведомить, объекты сами регистрируются в списке подписчиков. Когда происходит событие, движок просто проходит по списку и уведомляет всех.

#### Как это работает в общих чертах

1. **Создание реестра сообщений (MessageRegistry):**
   - Это контейнер, который хранит список объектов-подписчиков
   - Каждый подписчик может иметь приоритет (кто-то должен обработать событие раньше, кто-то позже)

2. **Регистрация подписчиков:**
   - Объект, который хочет получать уведомления, регистрируется в реестре
   - При регистрации указывается приоритет обработки

3. **Генерация события:**
   - Когда происходит событие (например, начался новый кадр), движок вызывает метод `Process()` у реестра
   - Реестр проходит по всем зарегистрированным объектам в порядке приоритета и вызывает у них соответствующий метод

4. **Отписка:**
   - Когда объект больше не нужен или уничтожается, он отписывается от реестра

#### Преимущества такого подхода

- **Слабая связанность:** движок не знает конкретных типов подписчиков, только интерфейс
- **Легко расширять:** новый объект может подписаться на событие без изменения кода движка
- **Гибкость:** можно менять порядок обработки через приоритеты
- **Безопасность:** можно безопасно добавлять/удалять подписчиков даже во время обработки события

#### Аналогия из реального мира

Представь систему рассылки новостей:
- **Реестр сообщений** — это список подписчиков на рассылку
- **Событие** — это новая новость, которую нужно разослать
- **Подписчики** — это люди, которые хотят получать новости
- **Приоритет** — это способ доставки: кому-то SMS (высокий приоритет), кому-то email (низкий приоритет)

Когда выходит новая новость, система просто проходит по списку подписчиков и отправляет им сообщение. Новые подписчики могут добавиться в любой момент, старые — отписаться, и система продолжит работать.

#### Особенности реализации в движке

В OpenXRay реализация Observer Pattern имеет несколько особенностей:

1. **Приоритеты обработки:**
   - Объекты обрабатываются в порядке приоритета (от высокого к низкому)
   - Это позволяет контролировать порядок выполнения (например, сначала обновить физику, потом отрисовать)

2. **Режим захвата (CAPTURE):**
   - Если объект имеет максимальный приоритет (CAPTURE), только он получает уведомление
   - Остальные подписчики игнорируются
   - Полезно для модальных окон или специальных режимов (например, меню паузы)

3. **Безопасная модификация во время обработки:**
   - Если подписчик добавляется или удаляется во время обработки события, изменения применяются после завершения обработки
   - Это предотвращает проблемы с итераторами и гарантирует стабильность

4. **Типизированные события:**
   - Каждый тип события имеет свой интерфейс (например, `pureFrame`, `pureRender`)
   - Это обеспечивает типобезопасность: объект не может подписаться на событие, для которого у него нет обработчика

#### Типы событий в движке

В OpenXRay используются следующие типы событий:

- **Frame** — начало нового кадра (обновление логики игры)
- **FrameEnd** — конец кадра
- **Render** — событие отрисовки (рендеринг сцены)
- **AppActivate** — приложение получило фокус
- **AppDeactivate** — приложение потеряло фокус
- **AppStart** — запуск приложения
- **AppEnd** — завершение приложения
- **DeviceReset** — сброс графического устройства
- **UIReset** — сброс UI системы

Каждое событие имеет свой реестр подписчиков, и объект может подписаться на несколько событий одновременно.

---

### Реализация в движке: как это устроено технически

#### Структура MessageRegistry

В движке `MessageRegistry` — это шаблонный класс, который хранит список подписчиков и управляет их уведомлением. Вот его ключевые компоненты:

```28:126:src/xrEngine/pure.h
template<class T>
class MessageRegistry
{
    struct MessageObject
    {
        T* Object;
        int Prio;
    };

    xr_vector<MessageObject> messages;
    bool changed, inProcess;

public:
    MessageRegistry() : changed(false), inProcess(false) {}

    void Clear() { messages.clear(); }

    constexpr void Add(T* object, const int priority = REG_PRIORITY_NORMAL)
    {
        Add({ object, priority });
    }

    void Add(MessageObject&& newMessage)
    {
#ifdef DEBUG
        VERIFY(newMessage.Object);
        VERIFY(newMessage.Prio != REG_PRIORITY_INVALID);

        // Verify that we don't already have the same object with valid priority
        for (size_t i = 0; i < messages.size(); ++i)
        {
            auto& message = messages[i];
            VERIFY(!(message.Prio != REG_PRIORITY_INVALID && message.Object == newMessage.Object));
        }
#endif
        messages.emplace_back(newMessage);

        if (inProcess)
            changed = true;
        else
            Resort();
    }

    void Remove(T* object)
    {
        for (size_t i = 0; i < messages.size(); ++i)
        {
            auto& message = messages[i];
            if (message.Object == object)
                message.Prio = REG_PRIORITY_INVALID;
        }

        if (inProcess)
            changed = true;
        else
            Resort();
    }

    void Process()
    {
        if (messages.empty())
            return;

        inProcess = true;

        if (messages[0].Prio == REG_PRIORITY_CAPTURE)
            messages[0].Object->OnPure(messages[0].Object);
        else
        {
            for (size_t i = 0; i < messages.size(); ++i)
            {
                const auto& message = messages[i];
                if (message.Prio != REG_PRIORITY_INVALID)
                    message.Object->OnPure(message.Object);
            }
        }

        if (changed)
            Resort();

        inProcess = false;
    }

    void Resort()
    {
        if (!messages.empty()) {
            std::sort(std::begin(messages), std::end(messages),
                [](const auto& a, const auto& b) { return a.Prio > b.Prio; });
        }

        while (!messages.empty() && messages.back().Prio == REG_PRIORITY_INVALID)
            messages.pop_back();

        if (messages.empty())
            messages.shrink_to_fit();

        changed = false;
    }
};
```

**Ключевые моменты:**

1. **MessageObject** — структура, хранящая указатель на объект и его приоритет
2. **messages** — вектор всех зарегистрированных подписчиков
3. **inProcess** — флаг, показывающий, идёт ли сейчас обработка события (нужен для безопасной модификации)
4. **changed** — флаг, показывающий, были ли изменения во время обработки

#### Объявление интерфейсов событий

Интерфейсы событий объявляются через макрос `DECLARE_MESSAGE`:

```11:26:src/xrEngine/pure.h
#define DECLARE_MESSAGE(name)\
struct pure##name\
{\
    virtual void On##name() = 0;\
    static ICF void __fastcall OnPure(pure##name* self) { self->On##name(); }\
}

DECLARE_MESSAGE(Frame); // XXX: rename to FrameStart
DECLARE_MESSAGE(FrameEnd);
DECLARE_MESSAGE(Render);
DECLARE_MESSAGE(AppActivate);
DECLARE_MESSAGE(AppDeactivate);
DECLARE_MESSAGE(AppStart);
DECLARE_MESSAGE(AppEnd);
DECLARE_MESSAGE(DeviceReset);
DECLARE_MESSAGE(UIReset);
```

Этот макрос создаёт структуру (например, `pureFrame`) с виртуальным методом `OnFrame()`, который должен быть реализован в классах-подписчиках.

#### Реестры событий в Device

В классе `CRenderDevice` (главный объект устройства рендеринга) хранятся реестры для всех типов событий:

```95:102:src/xrEngine/device.h
    // Registrators
    MessageRegistry<pureRender> seqRender;
    MessageRegistry<pureAppActivate> seqAppActivate;
    MessageRegistry<pureAppDeactivate> seqAppDeactivate;
    MessageRegistry<pureAppEnd> seqAppEnd;
    MessageRegistry<pureFrame> seqFrame;
    MessageRegistry<pureFrame> seqFrameMT;
    MessageRegistry<pureDeviceReset> seqDeviceReset;
    MessageRegistry<pureUIReset> seqUIReset;
```

**Примечание:** `seqFrameMT` — это отдельный реестр для многопоточных обработчиков кадра (MT = Multi-Threaded).

#### Пример 1: Регистрация объекта на событие Frame

Рассмотрим, как класс `IGame_Persistent` подписывается на события:

```28:33:src/xrEngine/IGame_Persistent.h
class ENGINE_API IGame_Persistent :
    public pureFrame,
    public pureAppActivate,
    public pureAppDeactivate,
    public IEventReceiver
{
```

Класс наследуется от интерфейсов событий, которые его интересуют. В конструкторе происходит регистрация:

```31:33:src/xrEngine/IGame_Persistent.cpp
    Device.seqFrame.Add(this, REG_PRIORITY_HIGH + 1);
    Device.seqAppActivate.Add(this);
    Device.seqAppDeactivate.Add(this);
```

А в деструкторе — отписка:

```50:52:src/xrEngine/IGame_Persistent.cpp
    Device.seqFrame.Remove(this);
    Device.seqAppActivate.Remove(this);
    Device.seqAppDeactivate.Remove(this);
```

Реализация обработчика события:

```552:588:src/xrEngine/IGame_Persistent.cpp
void IGame_Persistent::OnFrame()
{
    ZoneScoped;

    SpatialSpace.update();
    SpatialSpacePhysic.update();

#ifndef _EDITOR
    if (!Device.Paused() || Device.dwPrecacheFrame)
        Environment().OnFrame();

    stats.Starting = ps_needtoplay.size();
    stats.Active = ps_active.size();
    stats.Destroying = ps_destroy.size();
    // Play req particle systems
    while (ps_needtoplay.size())
    {
        CPS_Instance* psi = ps_needtoplay.back();
        ps_needtoplay.pop_back();
        psi->Play(false);
    }
    // Destroy inactive particle systems
    while (ps_destroy.size())
    {
        // u32 cnt = ps_destroy.size();
        CPS_Instance* psi = ps_destroy.back();
        VERIFY(psi);
        if (psi->Locked())
        {
            Log("--locked");
            break;
        }
        ps_destroy.pop_back();
        psi->PSI_internal_delete();
    }
#endif
}
```

#### Пример 2: Регистрация с приоритетом

В классе `CEngine` регистрация происходит с явным указанием приоритета:

```73:76:src/xrEngine/Engine.cpp
    Device.seqFrame.Add(this, REG_PRIORITY_HIGH + 1000);

    Device.seqFrame.Add(&g_sound_processor, REG_PRIORITY_NORMAL - 1000); // Place it after Level update
    Device.seqFrameMT.Add(&g_sound_renderer);
```

Здесь `CEngine` регистрируется с очень высоким приоритетом (`REG_PRIORITY_HIGH + 1000`), чтобы его метод `OnFrame()` вызывался одним из первых.

#### Пример 3: Режим захвата (CAPTURE)

В `AnselManager` используется режим захвата для перехвата всех событий кадра:

```89:89:src/xrGame/AnselManager.cpp
            Device.seqFrame.Add(mutable_this, REG_PRIORITY_CAPTURE);
```

Когда объект регистрируется с приоритетом `REG_PRIORITY_CAPTURE`, только он получает уведомление о событии, все остальные подписчики игнорируются. Это используется для специальных режимов, например, когда активен NVIDIA Ansel (фоторежим).

#### Вызов Process(): генерация события

События генерируются в основном цикле движка. Например, событие `Frame` вызывается в методе `CRenderDevice::FrameMove()`:

```484:484:src/xrEngine/device.cpp
    seqFrame.Process();
```

А событие `Render` вызывается в методе `CRenderDevice::Render()`:

```240:240:src/xrEngine/device.cpp
            seqRender.Process(); // all rendering is done here
```

Когда вызывается `Process()`, реестр:
1. Устанавливает флаг `inProcess = true`
2. Проверяет, есть ли объект с приоритетом `CAPTURE` (если да, вызывает только его)
3. Иначе проходит по всем подписчикам в порядке приоритета и вызывает у них метод `OnPure()`
4. Если во время обработки были изменения (добавления/удаления), вызывает `Resort()` для пересортировки
5. Сбрасывает флаг `inProcess = false`

#### Пример 4: Регистрация на событие Render

Класс `CActor` (игрок) подписывается на событие рендеринга в конструкторе:

```180:180:src/xrGame/Actor.cpp
    Device.seqRender.Add(this, REG_PRIORITY_LOW);
```

И отписывается в деструкторе:

```237:237:src/xrGame/Actor.cpp
    Device.seqRender.Remove(this);
```

Это позволяет актёру отрисовывать себя каждый кадр, когда вызывается `Device.seqRender.Process()`.

#### Безопасная модификация во время обработки

Важная особенность реализации — безопасность при модификации списка во время обработки:

1. Если подписчик добавляется/удаляется **до** вызова `Process()`, изменения применяются сразу через `Resort()`
2. Если подписчик добавляется/удаляется **во время** вызова `Process()`, устанавливается флаг `changed = true`, и `Resort()` вызывается **после** завершения обработки

Это предотвращает проблемы с итераторами и гарантирует, что все подписчики получат уведомление, даже если список изменяется во время обработки.

#### Приоритеты обработки

Приоритеты определены как константы:

```5:9:src/xrEngine/pure.h
constexpr int REG_PRIORITY_LOW = 0x11111111;
constexpr int REG_PRIORITY_NORMAL = 0x22222222;
constexpr int REG_PRIORITY_HIGH = 0x33333333;
constexpr int REG_PRIORITY_CAPTURE = 0x7fffffff;
constexpr int REG_PRIORITY_INVALID = std::numeric_limits<int>::lowest();
```

Объекты сортируются по убыванию приоритета (от большего к меньшему), поэтому объекты с высоким приоритетом обрабатываются первыми. Можно использовать относительные приоритеты, например `REG_PRIORITY_HIGH + 1000` или `REG_PRIORITY_NORMAL - 1000`, чтобы точно контролировать порядок выполнения.

#### Итоговая схема работы

1. **Инициализация:** объект наследуется от нужного интерфейса (`pureFrame`, `pureRender` и т.д.) и реализует виртуальный метод (`OnFrame()`, `OnRender()` и т.д.)
2. **Регистрация:** в конструкторе объект регистрируется в соответствующем реестре через `Device.seq*.Add(this, priority)`
3. **Обработка:** когда происходит событие, движок вызывает `Device.seq*.Process()`, который уведомляет всех подписчиков в порядке приоритета
4. **Отписка:** в деструкторе объект отписывается через `Device.seq*.Remove(this)`

Эта система обеспечивает гибкую и расширяемую архитектуру, где новые компоненты могут легко подписаться на события без изменения кода движка.

---

