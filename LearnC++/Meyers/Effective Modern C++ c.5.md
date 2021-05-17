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




















