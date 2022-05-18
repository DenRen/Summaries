4\. Интеллектуальные указатели
===

В С++11 есть 4 *интеллектуальных указателя*: *std::auto_ptr*, *std::unique_ptr*, *std::shared_ptr* и *std::weak_ptr*.
*std::auto_ptr* пришёл из C++98 и является устаревшим. *std::unique_ptr* во всех смыслах лучше *std::auto_ptr*.

## 4.1 Используйте *std::unique_ptr* для управления ресурсами путём исключительного владения.

*std::unique* имеет тот же размер, что и обычный указатель, а для большинства операций выполняются такие же команды.

В процессе конструирование объект *std::unique+ptr* можно настроить для использование *пользовательских удалителей (custom deleters)*.

```cpp
class Investment {                     // Инвестиции
public:
    virtual ~Investment ();            // Важная часть дизайна
};
class Stock :
    public Investment { /* ... */ };   // Акции
class Bond :
    public Investment { /* ... */ };   // Облигации
class RealEstate :
    public Investment { /* ... */ };   // Недвижимость

auto delInvmt = [] (Investment* pInvestment)
    {
        makeLogEntry (pInvestment);
        delete pInvestment;
    }

template <typename... Ts>
auto makeInvestment (Ts&&... params)    // Фабричная функция (C++14)
{
    std::unique_ptr <Investment, decltype (delInvmt)> pInv (nullptr, delInvmt);
    if ( /* Stock */ )
    {
        pInv.reset (new Stock (std::forward <Ts> (params)...));
    }
    else if ( /* Bond */ )
    {
        pInv.reset (new Bond (std::forward <Ts> (params)...));
    }
    else if ( /* RealEstate */ )
    {
        pInv.reset (new RealEstate (std::forward <Ts> (params)...));
    }

    return pInv;
}
```

Как только в *std::unique_ptr* появляется пользовательские удалители, его размер увеличивается на указатель. Но это происходит не всегда, а когда указывается функция. В случае *функциональных объектов без состояний* (например, лямба-выражения без захватов), увеличение размера не происходит, поэтому разумно пользоваться в качестве пользовательских удалителей *лямба-выражениями*.

Благодаря тому, что *std::unique_ptr* имеет две разновидности: *std::unique_ptr <T>* и *std::unique_ptr <T[]>*, т.е. для объектов и массивов, то не возникает неопределённости на какую сущность указывает *std::unique_ptr*. Но использовать встроенные массивы разумно разве что для С-образного API.

*std::unique_ptr* - способ выражения исключительного владения на С++11. Но очень привлекательно то, что его можно легко и эффективно преобразовать в *std::shared_ptr*:
```cpp
std::shared_ptr <Investment> sharedPointer = makeInvestment (args);
```

Благодаря этому *std::unique_ptr* очень хорошо подходит для фабричных функций, ведь последние не знают кто будет владеть объектом: один или совместно.

### <center>Следует запомнить</center>
* *std::unique_ptr* представляет собой маленький, быстрый, предназначенный только для перемещения интеллектуальный указатель для управления ресурсами с семантикой исключительного владения.
* По умолчанию освобождение ресурсов выполняется с помощью оператора *delete*, но могут применяться и *пользовательские удалители*. Удалители без состояний и указатели на функции в качестве удалителей увеличивают размеры объектов *std::unique_ptr*.
* Интеллектуальные указатели *std::unique_ptr* легко преобразуются в интеллектуальные указатели *std::shared_ptr*.

## 4.2 Используйте *std::shared_ptr* для управления ресурсами путём совместного владения

Благодаря совместному владению, в *sdt::shared_ptr* есть счётчик ссылок. Это значит, что он атамрный и операции его изменения дорогостоящи.

При перемещающем конструировании этот счётчик не меняется, следовательно такие операции дешёвые.

* Размер *sdt::shared_ptr* в два раза больше размера обычного указателя
* Память счётчика ссылок должна выделяться динамически (кроме *std::make_shared*)
* Инкремент и декремент счётчика должны быть атомарными

Для *std::unique_std* тип удалителя является частью типа интеллектуального указателя
Для *sdt::shared_ptr*это не так:
```cpp
auto loggingDel = [] (Widget* pw)
    {
        makeLogEntry (pw);
        delete pw;
    }

std::unique_ptr <Widget, decltype (loggingDel)
                > upw (new Widget, loggingDel);

std::shared_ptr <Widget> spw (new Widget, loggingDel);
```

В отличие от *std::unique_ptr*, *std::shared_ptr* с разными удалителями можно хранить в массиве.

*Управляющий блок* есть для каждого объекта, на которого ссылается *std::shared_ptr*.
*Управляющий блок*:
* Счётчик ссылок
* Слабый счётчик
* Прочие данные (пользовательский удалитель, распределитель и т.п.)

Правила при создании управляющего блока:
* Функция *std::make_shared* (см. раздел 4.4) всегда создаёт управляющий блок
* Управляющий блок создаётся тогда, когда *std:shared_ptr* создаётся из указателя с исключительным влдаением (т.е. std::unique_ptr или std::auto_ptr)
* Когда конструктор *std::shared_ptr* вызывается с обычным указателем, он создаёт управляющий блок

Последнее правило наталкивает на мысль, что если создать два *std::shared_ptr* от обычного, то будет неопределённое поведение. Простым решением этой проблемы - это передавать непосредственно результат оператора *new* или аналогично с другими указателями. Т.е. не создавать переменных, хранящих обычные указатели.

Далее возникает вопрос: что если в функции-члене вызывается *std::shared_ptr*, который много раз может принять this? Это снова неопределённое поведение:
```cpp
class Widget {
public:
    void process ();
};

std::vector <std::shared_ptr <Widget>> processedWidget;


void Widget::process () {
    processedWidget.emplace_back (this);    // Это неправильно!
}
```

Но что если на этот объект Widget снаружи уже указывает *std::shared_ptr*? Тогда у нас неопределённое поведение!

Но можно и безопансо создавать *std::shared_ptr* из *this:
```cpp
class Widget : public std::enable_shared_from_this <Widget> {
public:
    void process ();
};

// ...

void Widget::process () {
    processedWidget.emplace_back (shared_from_this ());    // Это верно!
}
```

Идея производного класса, порождённого от базового класса, шаблонизированного производным, - известный шаблон проетирования. Она называеся *странно повторяющийся шаблон* (*The Curiously Recurring Template Pattern - CRTP*).

Шаблон *std::enable_shared_from_this* определяет функцию-член *shared_from_this*, которая создаёт *std:shared_ptr* для текущего объекта, но деалет это, не дублируя управляющие блоки. *std::shared_from_this* полагается на то, что у этого объекта уже есть управляющий блок, иначе будет неопр. поведение, хотя обычно *std::shared_from_this* генерирует исключение.

Чтобы клиенты не вызывали методы, где используется *std::shared_from_this*,  конструкторы объявляют как private и заставляют клиентов создвать объекты через фабричные функции:
```cpp
class Widget : public std::enable_shared_from_this <Widget>
{
public:
    // Фабричная функция, пересылающая аргументы закрытому конструктору
    template <typename... Ts>
    static std::shared_ptr <Widget> create (Ts&&... params);
    // ...
    void process ();    // Как и ранее

private:
    // Конструкторы
};
```

Управляющий блок обычно имеет размер в несколько слов, но может и больше. Также в *std::shared_ptr* есть виртуальные функции, которые тоже замедляют работу, но они обычно используются один раз - для вызова деструктора.

В отличие от *std::unique_ptr*, *std::shared_ptr* не могут работать с массивами! Не существует *std::share_ptr <T[]>*. *std::shared_ptr* не имеет *operator[]*, так что даже если создать массив, по нему неудобно ходить.

### <center>Следует запомнить</center>
* *std::shared_ptr* предоставляет удобный подход к управлению времени жизни произвольных ресурсов, аналогичный сборке мусора.
* По сравнению с *std::unique_ptr* объекты *std::shared_ptr* обычно в два раза больше, привносят накрадные расхды на работу с управляющими блоками и требуют атомарной работы со счётчиком ссылок
* Освобождение ресурсов по умолчанию выполняется при помощи оператора *delete*, однако поддеживаются и пользовательские удалители. Тип удалителя не влияет на тип указателя *std::shared_ptr*.
* Избегайте создание указателей *std::shared_ptr* из переменных, тип которых - обычный встроенный указатель.

## 4.3 Используйте *std::weak_ptr* для *std::shared_ptr* - подобных указателей, которые могут быть висячими

*std::weak_ptr* не может быть ни разыменован, ни проверен на "нулёвость". *std::weak_ptr* не влияет на счётчик, и даже если счётик обнулился, а объект уничтожился, то этот указатель становится висячим, а *weak_pointer.expired () == true*.

Если нам нужно одновременно проверить висячесть и обратиться на этот объект, то нужно создавать *std::share_ptr* из *std::weak_ptr*, которая является атомарной операцией. Есть два способа это сделать, через *std::weak_ptr::lock ()*:
```cpp
std::shred_ptr <Widget> spw1 = wpw.lock (); // Если просрочен, то spw1 - нулевой

auto spw2 = wpw.lock ();    // Аналогично, но с auto
```

и через конструктор *std::shared_ptr*:
```cpp
std::shared_ptr <Widget> spw3 (wpw); // Если wpw просрочен, генерируется std::bad_weak_ptr
```

*std::weak_ptr* нужен, например, для хранения кэша указаелей на объекты. Представим есть дорогостоящая функция:
```cpp
std::unique_ptr <const Widget> loadWidget (Widget id);
```

Мы хотим её улучшить, добавим кэш (пример на скорую руку):
```cpp
std::shared_ptr <Widget> fastLoadWidget (Widget id)
{
    static std::unordered_map <WidgetID,
                               std::weak_ptr <Widget>> cache;

    auto objPtr = cache[id].lock (); // objPtr является shared_ptr

    if (!objPtr) {
        objPtr = loadWidget (id);

        cache[id] = objPtr;
    }

    return objPtr;
}
```

Здесь есть проблема что кэш может накапливать просроченные указатели, но мы не будем решать эту проблему, а рассмотрим ещё один пример применения *std::weak_ptr*.

Шаблон проектирования *Observer* (*Наблюдатель*). Основными компонентами этого шаблона являются субъекты и наблюдатели.

Приведём пример: пусть есть объекты A, B и C, где A и C совместно владеют B, т.е. хранят указатели *std:shared_ptr* на B:
```cpp
//    std::shared_ptr       std::shared_ptr
// A -----------------> B <----------------- C
```

Предположим, что полезно иметь указатель на A из B, тогда у нас есть три варианта:
* Обычный указатель
* Указатель *std::shared_ptr*
* Указатель *std::weak_ptr*

Понятно, что лучше всего подходит *std::weak_ptr*, т.к. нужно знать когда удалился A и в него больше не записывать и сделать код практичным (в данном случае *std::shared_ptr* этого не позволяет).

В деревьях можно делать следующие связи: от родительских к дочерним лучше всего предоставлять *std::unique_ptr*, а от дочерних к родительским обычные указатели, потому что дочерние заведомо не будут жить дольше родителей.

*std::shared_ptr* по эффективности примерно такой же, как *std::shared_ptr*. *std::weak_ptr* работет со вторым счётчиком ссылок в упарвляющем блоке *std::shared_ptr*.


### <center>Следует запомнить</center>
* Используйте *std::weak_ptr* как *std::shared_ptr* - образные указатели, которые могут быть висячими.
* Потенциальные применения *std::weak_ptr* включают кэширование, списки наблюдателей и предупреждение циклов указателей.

## 4.4 Предпочитайте использование *std::make_unique* и *std::make_shared* непосредственному использованию оператора *new*

C++11 имеет только *std::make_shared*, а *std::make_unique* появился только в С++14, но его базовую версию довольно просто написать самому:
```cpp
template <typename T, typename... Ts>
std::unique_ptr <T> make_unique (Ts&&... params)
{
    return std::unique_ptr <T> (
        new T (std::forward <Ts> (params)...)
    );
}
```

Третья *make-функция* - это *std::allocate_shared*. Это *std::shared_ptr*, только первым эллементом она принимает объект распределителя, использующийся для выделения динамической памяти.

Первая причина, почему лучше использвать *make-функции* - это отсутствие дублирования кода:
```cpp
auto upw1 = std::make_unique <Widget> ();    // С make-функцией
std::unique_ptr <Widget> upw2 (new Widget);  // Без make-функции

auto spw1 = std::make_shared <Widget> ();    // С make-функцией
std::shared_ptr <Widget> spw2 (new Widget);  // Без make-функции
```

Вторая причина  - это *безопасность исключений*. Рассмотрим пример, где используется *new*, вместо  *std::make_shared*:
```cpp
void processWidget (std::shared_ptr <Widget> spw, int priority);

// ...

processWidget (
    std::shared_ptr <Widget> (new Widget),
    computePriority ()
    );
```

А это потенциальная утечка ресурса!

Компилятору сначала нужно вычислить аргументы *processWidget*.

Он может это сделать в любом порядке:
1. Выполнить *new Widget*
2. Вызвать конструктор *std::shared_ptr*
3. Выполнить *computePriority*

Если в *2* возникнет исключение, то *std::shared_ptr* освободит ресурсы и всё будет хорошо.

**Но компилятор может генерировать код и в следующем порядке:**
1. Выполнить *new Widget*
2. Выполнить *computePriority*
3. Вызвать конструктор *std::shared_ptr*

Тогда если в *2* возникнет исключение, то созданный в *1* *Widget* будет потерян и у нас возникнет утечка!

Именно поэтому здесь лучше использвоать *std::make_unique*:
```cpp
processWidget (
    std::make_shared <Widget> (),    // Никаких проблем здесь не будет
    computePriority ()
    );
```

Причём *std::make_shared* быстрее чем *std::shared_ptr <Widget> spw (new Widget)*, потому что в первом случае произойдёт одно выделение памяти, а во втором два!

Но в *make-функциях* невозможно указать удалитель, а также аргументы передаются в прямой передаче через круглые скобки, а не фигурные. Для создания через *std::initializer_list* нужно использовать переменную, которая хранит этот лист (*auto = { 17, 3 }*).

Для *std::unique_ptr* и *std::make_unique* - это все проблемы, а вот *std::shared_ptr* и его *make-функций* нет.

Если пользовательские классы определяют собственные версии *operaot new* и *operaor delete*, то для них плохо подходит, например, *std::allocate_shared*, потому что кол-во динамически выделяемой памяти не совпадает с *sizeof (Widget)*.

*Счётчик ссылок* нужен для подсчёта *std::shared_ptr* (Для освобождения объекта).\
*Слыбый счётчик* нужен для подсчёта (Для освобождения управляющего блока) *std::weak_ptr*.

Если класс большой и нужно освобождать память объекта не дожидаясь удаления *std::weak_ptr*, то вместо
```cpp
class ReallyBigType { //... };

auto pBigObj = std::make_shared <ReallyBigType> ();
```

нужно писать:
```cpp
std::shared_ptr <ReallyBigType> pBigObj (new ReallyBigObj);
// Здесь применение std::make_shared неприемлемо
```

Если мы хотим создать *std::shared_ptr* c пользовательским удалителим и безопасным относительно исключений, то *std::make_shared* здесть неприменим и нужно писать так:
```cpp
void processWidget (std::shared_ptr <Widget> spw, int priority);
void customDeleter (Widget* ptr);

std::shared_ptr <Widget> spw (new Widget, customDeleter);
processWidget (spw, computePriority ();
```

Но это не так эффективно, потому что копирование *std::shared_ptr* включает себя дорогостоящее инкрементирование атомарного счётчика. Это легко исправить:
```cpp
processWidget (std::move (spw), computePriority ());
```

Теперь этот код работает также быстро, как и в небезопасном случае!

### <center>Следует запонмить</center>
* По сравнению с непосредственным использованием *new*, *make-функции* устраняют дублирование кода, повышвют безопасность кода по отношению к исключениям и в случаее *std::make_shared* и *std::allocate_shared* генерируют меньший по размеру и более быстрый код.
* Ситуайии, когда применение *make-функций*  неприемлемо, включают необходимость указания пользовательских удалителей и необходимость передачи инициализаторов в фигурных скобках.
* Для указателей *std::shared_ptr* дополнительными ситауциями, в которых применение *make-функций* может быть неблагоразумным, являются классы с пользовательским управлением памятью и системы, в которых проблемы с объёмом памяти накладываются на использование очень больших объектов и наличие указателей *std::weak_ptr*, время жизни которых существенно превышает время жизни указателей *std::shared_ptr*.

## 4.5 При использовании идиомы указателя на реализацию определяйте специальные функции-члены в файле реализации

Идиома *Pimple* (*pointer to implementation*), напремир, для ускорения компиляции можно в классе хранить не объекты пользовательских классов, заголовочные файлы которых постоянно обновляются, как следствие нужно *всё* зановово перекомпилировать (*долго*), а указатель на общую структуру (*реализацию*), а тип указателя делать неполным:
```cpp
class Widget {
public:
    Widget ();
    ~Widget ();

    //...

private:
    struct Impl;    // Объявление структуры реализации
    Impl* pImpl;    // и указателя на неё
};
```

А  в *Widget.cpp* уже определять *struct Impl* и только там подключать пользовательский заголовочный файл для пользовательских типов. Также нужно будет своими руками писать деструктор.

В *Widget.cpp*:
```cpp
#include "widget.h"
#include "gadget.h"
#include <vector>
#include <string>

struct Widget::Impl {
    std::string name;
    std::vector <double> data;
    Gadget gadget1, gadget2, gadget3;
};

Widget::Widget () :
    pImpl (new Impl)
{}

Widget::~Widget () {
    delete Impl;
}
```

Этот пример можно улучшить, избавившись от обычных указателей:
```cpp
class Widget {
public:
    Widget ();

    //...

private:
    struct Impl;                  // Объявление структуры реализации
    std::unique_ptr <Impl> pImpl; // и интеллектуального указателя на неё
};
```

В *Widget.cpp*теперь (теперь без деструктора):
```cpp
#include "widget.h"
#include "gadget.h"
#include <vector>
#include <string>

struct Widget::Impl {
    std::string name;
    std::vector <double> data;
    Gadget gadget1, gadget2, gadget3;
};

Widget::Widget () :
    pImpl (std::make_unique <Impl> ())
{}
```

Вот только этот код не компилируется)
```cpp
#include "widget.h"

Widget widget;    // Ошибка!
```
Дело в том, что компилятор С++11 проверяет перед использованием *delete* при помощи *static_assert* является ли этот тип укзателем на неполный тип. Чтобы это исправить, нужно обеспечить полноту типа *Widget::Impl* в точке, где генерирует код, уничтожающий *std::unique_ptr <Widget::Impl>*. Нужно просто показать компилятору тело деструктора:
```cpp
class Widget {
public:
    Widget ();
    ~Widget ();    // Только объявление
    //...

private:
    struct Impl;
    std::unique_ptr <Impl> pImpl;
};
```

В *Widget.cpp*теперь:
```cpp
#include "widget.h"
#include "gadget.h"
#include <vector>
#include <string>

struct Widget::Impl {
    std::string name;
    std::vector <double> data;
    Gadget gadget1, gadget2, gadget3;
};

Widget::Widget () :
    pImpl (std::make_unique <Impl> ())
{}

Widget::~Widget () = default; // Определение ~Widget
```

Благодаря идиоме *Pimpl* класс *Widget* создан для того, чтобы его перемещали. Но если мы вспомним правила создания перемещающих операций, то поймём, что они не генерируются автоматически, поэтому пишем их сами:
```cpp
class Widget {
public:
    Widget ();
    ~Widget ();

    Widget (Widget&&) = default;                // Идея верна,
    Widget& operator= (Widget&&) = default;     // код - НЕТ

    //...

private:
    struct Impl;
    std::unique_ptr <Impl> pImpl;
};
```

В *Widget.cpp* как и ранее.

Опять же мы получаем ту же самую ошибку, потому что компилятор не знает в *Widget.h* полный размер *Impl*. Решение такое же, как и в прошлый раз:
```cpp
class Widget {
public:
    Widget ();
    ~Widget ();

    Widget (Widget&&);                // Только объявление
    Widget& operator= (Widget&&);     // Только объявление

    //...

private:
    struct Impl;
    std::unique_ptr <Impl> pImpl;
};
```

В *Widget.cpp*теперь:
```cpp
#include "widget.h"

// ... // Как и ранее

// Определения:
Widget::Widget (Widget&&) = default;
Widget::Widget& operator= (Widget&&) = default;
```

Теперь добавим копирование по тому же принципу, только не забудем, что нам нужно *глубокое копирование*:
```cpp
class Widget {
public:
    Widget ();
    ~Widget ();

    Widget (const Widget&);                // Только объявление
    Widget& operator= (cosnt Widget&);     // Только объявление

    //...

private:
    struct Impl;
    std::unique_ptr <Impl> pImpl;
};
```

В *Widget.cpp*теперь:
```cpp
#include "widget.h"

// ... // Как и ранее

// Определения:
Widget (const Widget& rhs) :
    pImpl (nullptr)
{
    if (rhs.pImpl != nullptr)
        pImpl = std::make_unique <Impl> (*rhs.pImpl);
}
Widget& operator= (cosnt Widget& rhs) {
    if (rhs.pImpl == nullptr)
        pImpl.reset ();

    else if (pImpl == nullptr)
        pImpl = std::make_unique <Impl> (rhs.pImpl);

    else
        pImpl = rhs.pImpl;

    return *this;
}
```

Если бы мы использовали *std::shared_ptr*, то ничего бы нам писать не пришлось и всё работало именно так, как нам нужно:
```cpp
class Widget {
public:
    Widget ();

    //...        // Нет дестрвктора и перемещающих операций

private:
    struct Impl;
    std::shared_ptr <Impl> pImpl;
};
```
```cpp
Widget w1;
auto w2 (std::move (w1);
w1 = std::move (w2);
```

Такое различие возникает из-за разной поддержки пользовательских указателей.

Но понятно, что для Pimpl нужно использовать исключительное владение, т.е. *std::unqiue_ptr*.

### <center>Следует запомнить</center>
* Идиома Pimpl уменьшает время компиляции приложения, снижая зависимости компиляции между клиентами и реализациями классов.
* Для указателей pImpl типа *std::unique_ptr* следует объявить *специальные функции-члены* в заголовочном файле, но реализовать их в файле реализации. Поступайте так, даже если реализации функций по умолчанию являются приемлемыми.
* Приведённый выше совет применим к интеллектуальному указателю *std::unique_ptr*, но не к *std::shared_ptr*.







































































