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
*std::forward* - ничего не передаёт
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
2. либо *rvalue*-ссылка, либо *lvalue-*ссылка (универсальные ссылки)

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













