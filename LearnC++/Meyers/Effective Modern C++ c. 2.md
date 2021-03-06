2\. Объявление auto
===

## 2.1 Предпочитайте auto явному объявлению типа

При написании кода возникают определённые проблемы:
1) Можно объявить, но забыть определить какую-то переменную:
    ```cpp
    int x;
    ```
2) Иногда просто неудобно записывать громоздкий тип:
    ```cpp
    template <typename T>
    void dwim (It b, It e) {
        while (b != e) {
            typename std::iterator_traits <It>::value_type
                currValue = *b;
            ...
        }
    }
    ```
3) Попытка объявить локальную переменную с типом лямбда-выражения. Но этот тип известен только **компилятору**!

Все эти проблемы решаются введением ключевого слова *auto*:
```cpp
// Решение 1-ой проблемы
auto x;     // Ошибка! Требуется инициализатор
auto y = 4; // Отлчино!
```

```cpp
// Решение 2-ой проблемы
template <typename T>
void dwim (It b, It e) {
    while (b != e) {
        auto currValue = *b; // ЗАмечательно!
        ...
    }
}
```

```cpp
// Первое решение 3-ей проблемы
auto derefUPLess =
[] (const std::unique_ptr <Widget>& p1,
    const std::unique_ptr <Widget>& p2)
    { return *p1 < *p2; }

// Второе решение 3-ей проблемы
auto derefUPLess =
[] (const auto& p1,
    const auto& p2)
    { return *p1 < *p2; }
```

Может прийти идея, что последнее можно записать через *std::function*, и да, это возможно, правда у этого есть много минусов.

```cpp
std::function <bool (const std::unique_ptr <Widget>&,
                     const std::unique_ptr <Widget>&)>

derefUPLess =    [] (const std::unique_ptr <Widget>& p1,
                     const std::unique_ptr <Widget>& p2)
                        { return *p1 < *p2; }
```
* Во-первых, *std::function* хранит много памяти и может вызывать выделение динамической памяти
* Во-вторых, *std::function* работает медленно, т.к. есть косвенные вызовы фукнций
* В-третьих, это неудобно и некрасиво писать

Поэтому для хранения *замыкания* лучше использовать *auto*

Применение *auto* помогает вам избежать ошибок сокращения типа:
```cpp
std::vector <int> v;

unsigned size_0 = v.size (); // Для x32 всё нормально
                             // Для x64 4-ёх байтовый тип приравнивается 8 байтному
                             // Неоректное поведение

auto size = v.size ();       // Тип автоматически определится: std::vector <int>::size_type
```

Ещё один интересный пример. Здесь очень неявная ошибка, которая сильно тормозит код:
```cpp
std::unordered_map <std::string, int> m;
...
for (const std::pair <std::string, int>& p : m) {
    ...
}
```
Дело в том, что часть *unordered_map*, содержащая ключ, является константной, поэтому, чтобы получить *std::string&* из *const std::string&* создаётся копия, приравнивается, используется и удаляется. Это крайне неэффективно!

Чтобы избежать такой ошибки, достаточно пользоваться *auto*:
```cpp
for (const auto& p : m) {
    ...
}
```

### <center>Следует запомнить</center>
* Переменные, объявленные как *auto*, должны быть инициализированы; в общем случае они не восприимчивы к несоответствиям типов, которые могут привести к проблемам переносимости или эффективности; могут облегчить процесс рефакторинга; и обычно требуют куда  меньшего количества ударов по клавишам, чем переменные с явно указанными типами.
* Переменные, объявленные как *auto*, могут быть подвержены неприятностям, описанным в разделах 1.2  и 2.2.


## Если auto выводит нежелательный тип, используйте явно типизированный инициализатор

Рассмотрим следующий код:
```cpp
std::vector <bool> featires (const Widget&);

Widget w;
bool highPriority = features (w)[5];

processWidget (w, highPriority);
```

Но после добавления "безобидного" *auto*, мы получим неопределённое поведение:
```cpp
std::vector <bool> featires (const Widget&);

Widget w;
auto highPriority = features (w)[5];

processWidget (w, highPriority);    // Неопределённое поведение
```

Дело в том, что при явной инициализации результат *features (w)[5]* неявно приводился к *bool*, тогда как в случае использования *auto* происходил вывод типа из *features (w)[5]*. Но этот тип **не** *bool*, а *std::vector\<bool\>::reference*, который содержит указатель на элемент во временно объекте *features (w)[5]*. После завершения строчки, в *highPriority* остаётся только *висячий указатель*!!! Поэтому в *processWidget (w, highPriority)* будет неопределённое поведение.

Следовательно, нужно избегать кода следующего типа:
```cpp
auto someVar = выражение с типом "невидимого" прокси-класса;
```
Чтобы его распознать, нужно читать либо документацию, либо заголовочные файлы.

*Идиома явной инициализации инициализатора*:
```cpp
auto highPriority = static_cast <bool> (features(w)[]5);
auto sum = static_cast <Matrix> (m1 + m2 + m3 + m4);
```

Эта идиома также может более точно говорить, что вы делаете. Пусть есть функция, которая возвращает *double*, но точности в *flaot* вам вполне достаточно. Тогда:
```cpp
float ep = calcEpsilon (); // Неявное преобразование float -> double
```

Но это вряд ли выражает мысль "я намеренно уменьшаю точность значения, возвращённого функцией". Зато это делает идиома явной типизацией инициализатора:
```cpp
auto ep = static_cast <float> (calcEpsilon ();
```
```cpp
auto index = static_cast <int> (d * (c.size () - 1));
```

### <center>Следует запомнить</center>
* "Невидимые" прокси-типы могут привести *auto* к выводу неверного типа инициализирующего выражения.
* Идиома явно типизированного инициализатора заставляет *auto* выводить тот тип, который вам нужен.