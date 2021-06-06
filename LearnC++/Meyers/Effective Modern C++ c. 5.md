Rvalue-ссылки, семантика перемещения и прямая передача
===

Полезной эвристикой для выяснения, является ли выражение *lvalue*, является ответ на вопрос, можно ли получить его адрес.

Сам по себе параметр является *lvalue*:
```cpp
class Widget {
public:
    Widget (Widget&& rhs); // rhs является lvalue, хотя        !! Важно !!
    ...                    // и имеет ссылочный тип rvalue
};
```

* **Семантика перемещения** позволяет компиляторам заменять дорогостоящие операции копирования менее дорогими перемещениями. Семантика перемещения позволяет создавать типы, которые могут только перемещаться, такие как *std::unique_ptr*, *std::future* или *std::thread*.
*   **Прямая передача** делает возможным написание шаблонов функций, которые принимают произвольные аргументы и передают их другим функциям так, что целевые функции получают в точности те де аргументы, что и переданные исходным функциям.

*Rvalue-ссылки* представляет собой клей, который соединяет две эти довольно разные возможности. Это базовый механизм языка программирования, который делает возможным как семантику перемещения, так и прямую передачу.

*Rvalue* указывают объекты, которые могут быть перемещены, в то время как *lvalue* в общем случае не перемещены быть не могут.

### 5.1 Азы std::move и std::forward

*std::move*      - ничего не перемещает\
*std::forward* - ничего не передаёт\
Во время выполнения они не делают вообще ничего. Они не генерируют исполняемый код - ни одного байта.

Они являются шаблонами функций, которые выполняют приведения:
1. *std::move* - выполняет безусловное приведение своего аргумента к rvalue
2. *std::forward* - выполняет приведения при соблюдении определённых условий

В C++11 std::move выглядит примерно так:
```cpp
#include <type_traits>

template <typename T>
typename std::remove_reference <T>::type&&
move (T&& param)
{
    using ReturnType = typename std::remove_reference <T>::type&&;
    return static_cast <ReturnType> (param);
}
```

В C++14 можно сделать немного красивее:

```cpp
#include <type_traits>

template <typename T>
decltype (auto) move (T&& param)
{
    using ReturnType = typename std::remove_reference <T>::type&&;
    return static_cast <ReturnType> (param);
}
```

*std::move* имеет такое имя: чтобы легко распознавать объекты, которые могут быть перемещены. *rvalue* **обычно** являются всего лишь кандидатами на перемещение.

Интересный пример:
```cpp
class Annotation {
public:
    explicit Annotation (const std::string text) :
        value (std::move (text))
    { ... }
    
    ...
    
private:
    std::string value;
};
```

#### Здесть *text* не перемещается в *value*, а копируется! Потому что *text const std::string =>* результат приведения - это *rvalue* типа *const std::string*

### Два урока:
1. Не объявляйте объекты как константные, если хотите иметь возможность выполнять перемещение из них.
2.  *std::move* не только ничего не перемещает самостоятельно, но даже не  гарантирует, что приведённый этой функцией объект будет иметь право быть перемещённым. Единственное, что точно известно о результате применения *std::move*, - это то, что он является *rvalue*.

Пример работы с *std::forward*:
```cpp
void process (const Widget& lvalArg);
void process (Widget&& rvalArg);

template <typename T>
void logAndProcess (T&& param)
{
    auto now = std::chrono::system_clock::now ();
    makeLogEntry ("Вызов 'process'", now);
    process (std::forward <T> (param));
}

Widget w;

logAndProcess (w);             // Вызов с lvalue
logAndProcess (std::move (w)); // Вызов с rvalue

```

Иногда *std::forward* является излишним по сравнению с *std::move*.
```cpp
// Желательная реализация

class Widget () {
public:
    Widget (Widget&& rhs) :
        s (std::move (rhs.s))
    { ++moveCtorCalls; }

private:
    static std::size_t moveCtorCalls;
    std::string s;
};
``` 

```cpp
// Нежелательная реализация

class Widget () {
public:
    Widget (Widget&& rhs) :
        s (std::forward <std::string> (rhs.s))
    { ++moveCtorCalls; }

private:
    static std::size_t moveCtorCalls;
    std::string s;
};
``` 

### <center>Следует запомнить</center>
* *std::move* выполняет безусловное приведение к *rvalue*. Сама по себе эта функция ничего не перемещает.
* *std::forward* приводит аргумент к *rvalue* только тогда, когда этот аргумент связан с *rvalue*
* Ни *std::move*, ни *std::forward* не выполняют никаких действий  времени выполнения


### 5.2 Отличие универсальных ссылок от rvalue-ссылок

*T&&* имеет два разных значения:
1. *rvalu*e-ссылка
2. либо *rvalue*-ссылка, либо *lvalue-* ссылка (универсальные ссылки)

Универсальные ссылки также могут быть связаны с *const*, *volatile* или *const volatile* объектами.

Универсальные ссылки возникают в двух контекстаз:
```cpp
// Параметры шаблона функций
template <typename T>
void f (T&& param);

// Объявление auto
auto&& var2 = var1;
```

Их связывает *вывод типа*.

```cpp
template <typename T>
void f (T&& param);

Widget w;
f (w);                // В f передаётся lvalue; param - Widget& (lvalue-ссылка)

f (std::move w));    // В f передаётся rvalue; param - Widget&& (rvalue-ссылка)

template <typename T>
void g (std::vector <T>&& param); // param - rvalue-ссылка

template <typename T>
void w (const T&& param); // param - rvalue-ссылка
```

Интересный пример:
```cpp
template <class T, class Allocator = allocator <T>>
class vector {
public:
    void push_back (T&& x);
};
```

Хотя и похоже, но здесь универсальной  ссылки нет, потому что нет вывода типа для x. Дело в том, что push_back не может существовать без конкретного инстанцированного вектора, частью которого он является; а тип этого инстанцирования полностью определяет объявление push_back.

Т.е. код:
```cpp
    std::vector <Widget> v;
```
Приводит к следующему инстанцированию шаблона:
```cpp
class vector <Widget, allocator <Widget>> {
public:
    void push_back (Widget&& x); // x - rvalue-ссылка
};
```

А вот здесь уже будет вывод типа:
```cpp
template <class T, class Allocator = allocator <T>>
class vectr {
public:
    template <class... Args>
    void emplace_back (Args&&... args); // Универсальный указатель
};
```

Пример с замером времени работы функции:
```cpp
auto timeFuncInvocation = 
[] (auto&& func, auto&&... params)    // func - унив. ссылка, params - нуль или несколько унив. ссылок
{
    // Запуск таймера
    std::forward <decltype (func)> (func) (
        std::forward <decltype (params)> (params)...;
    );
    // Остановка таймера
}
```
Весь этот раздел  - основы универсальных ссылок - абстракция. Лежащая в основе истина - это свёртывание ссылок (reference collapsing).

### <center>Следует запомнить</center>
* Если параметр шаблона функции имеет тип *T&&* для выводимого типа *T* или если объект объявлен с использование *auto&&*, то параметр или объект объявлен универсальной ссылкой.
* Если вид объекта типа не является в точности *type&&* или если вывод типа не имеет места, то *type&&* означает *rvalue*-ссылку.
* Универсальные ссылки соответствуют *rvalue*-ссылкам, если они инициализируются значением *rvalue*. Они соответствуют *lvalue*-ссылкам, если они инициализируются значениями *lvalue*.

### 5.3 Используйте std::move для rvalue-ссылок, а std::forward - для универсальных

Понятно, что при использовании *rvalue*-ссылки, мы хотим использовать преимущества "правосторонности". Поэтому мы приводим параметры, связанные с такими объектами, к *rvalue*. Для этой задачи создана функция *std::move*:
```cpp
class Widget {
public:
    Widget (Widget&& rhs) :            // rhs - rvalue-ссылка
        name (std::move (rhs.name)),
        data (std::move (rhs.data))
    {
        // ...
    }
    
private:
    std::string name;
    std::shared_ptr <SomeStructData> data;
    
};
```

Когда мы используем *универсальные* ссылки, мы подразумеваем, что объект *может* быть перемещён, поэтому мы неявно обрабатываем сразу два случая при помощи *std::forward*:
```cpp
class Widget {
public:
    template <typename T>
    void SetName (T&& newName) {            // newName - универсальная ссылка
        name = std::forward <T> (newName);
    }
private:
    // Аналогично
};
```

Короче говоря, *rvalue*-ссылки при их передаче в другие функции должны быть безусловно приведены к *rvalue* (с помощью *std::move*), т.к. они всегда связываются с *rvalue*, а *универсальные* ссылки должны приводиться к *rvalue* при из передаче условно (с помощью *std::forward*), поскольку они только иногда связаны с *rvalue*.

Не нужно использовать *std::forward* для *rvalue*-ссылок, потому что код становится многословным, подверженным ошибкам и неидиоматичным! **И ни в коем случаем** нельзя применять *std::move* для универсальных ссылок, т.к. это может привести у неожиданному изменению значений lvalue:  
```cpp
class Widget {
public:
    template <typename T>
    void SetName (T&& newName) {    // newName - универсальная ссылка
        name = std::move (newName); // Плохо!   
    }
private:
    // Аналогично
};

std::string getWidgetName (); // Фабричная функция
Widget w;

auto name = getWidgetName ();
w.SetName (name);    // Перемещение name в w!
                     // Значение name теперь неизвестно
```

Конечно, можно написать две функции: одна принимает *const std::string& name*, другая *std::string&& name*, но
1. Это в два раза больше строк кода!!!
2. В случае передачи литерала *"Adela Novak"* будет создан временный объект! При использовании универсальных ссылок, временного объекта создаваться не будет

Получается, что универсальные ссылки безопаснее, удобнее и эффективнее!

Но самая большая проблема с перегрузкой rvalue и lvalue - это плохая масштабируемость!

В некоторых случаях без такой подход *невозможен*. Следующая функция может принимать любое количество как lvalue параметров, так и rvalue. Поэтому единственный выход здесь применять только универсальные ссылки:
```cpp
template <class T, class... Args>
unique_ptr <T> make_unique (Args&&... args);
```

Нужно пользоваться возможностью более эффективного перемещения:
```cpp
Matrix operator* (Matrix&& lhs, const Matrix& rhs) { // Возврат по значению
    lhs += rhs;
    return std::move (lhs);
}
```

Аналогично для *std::forward*:
```cpp
template <typename T>
Fraction reduceAndCopy (T&& frac) {
    frac.reduce ();
    return std::forward <T> (frac); // Перемещение с случае rvalue, копирование в случае lvalue
}
```

Но не стоит допускать грубых ошибок и возвращать ссылки на локальные объекты функций:
```cpp
Widget makeWidget () {
    Widget w;
    
    // ...
    
    return std::move (w);    // Не делайте этого!
}
```

В любом хоть немного приличном компиляторе C++ уже есть *RVO* - return value optimization.
RVO выполняется, если:
1. Тип локального объекта совпадает с возвращаемым функцией
2. Локальный объект представляет собой возвращаемое значение

Предыдущий пример нужно написать вот так:
```cpp
Widget makeWidget () {
    Widget w;
    
    // ...
    
    return w;    // "Копирование" w в возвращаемое значение
}
```

### <center>Следует запомнить</center>
* Применяйте *std::move* к *rvalue*-ссылкам, а *std::forward* - к *универсальным* ссылкам, когда вы их используете в последний раз
* Делайте то же для rvalue- и универсальных ссылок, возвращаемых из функций по значению.
* Никогда не применяйте *std::move* и *std::forward* к локальным объектам, которые могут быть объектом оптимизации возвращаемого значения

















































