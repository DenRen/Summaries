1\. Вывод типов
============
## 1.1 Вывод типа шаблона

```C++
template <typename T>
void f (ParamType param);

f (expr);
```

*  ParamType - указатель или  ссылка (не универсальная)
*  ParamType - универсальная ссылка
*  ParamType - ни указатель, ни ссылка

### Случай 1 (ParamType - указатель или  ссылка (не универсальная))

Правила:
1. Если  expr - ссылка, то ссылочная часть отбрасывается (т. е. указательная тоже)
2. Затем сопоставляется тип expr с ParamType для определения T

Пример:
```cpp
template <typename T>
void f (T& param);

int x = 27;
const int cx = x;
const int& crx = x;

f (x);    // T - int,       param - int &
f (cx);   // T - const int, param - const int &
f (crx);  // T - const int, param - const int &

// ----------------------------------------------------

template <typename T>
void g (const T& param);

g (x);    // T - int,       param - const int&
g (cx);   // T - const int, param - const int& 
g (rcx);  // T - const int, param - const int&
```
#### Передача **константного** объекта в шаблон, получающий параметр T&, *безопасна*: константность объекта становится частью выведенного для Т типа.

```cpp
template <typename T>
void f (T* param);

int x = 27;
const int *px = &x;

f (x);    // T - int,       param - int *
f (px);   // T - const int, param - const int *
```

### Случай 2 (ParamType - универсальная ссылка)

Правила:
1. Если expr - lvalue, то T и ParamType - выводятся как lvalue-ссылки
2. Если expr - rvalue, применяются правила из Случая 1

```cpp
template <typename T>
void f (T&& param);

int x = 27;
const int cx = x;
const int& crx = &x;

f (x);    // x   - lvalue, T - int&,       param - int&
f (cx);   // cx  - lvalue, T - const int&, param - const int&
f (crx);  // crx - lvalue, T - const int&, param - const int&
f (27);   // 27  - rvalue, T - int,        param - int&& 
```
#### Когда используются универсальные ссылки, вывод типов различает аргументы, являющиеся lvalue, и аргументы, являющиеся rvalue. Этого никогда не происходит для неуниверсальных ссылок.

### Случай 3 (ParamType - не является ни указателем, ни ссылкой)

#### Это значит, что мы имеем дело с передачей по значению

```cpp
template <typename T>
void f (T param);
```

Правила:
1. Если expr - *ссылка*, то ссылочная часть **игнорируется**
2. Если expr - если после отбрасывания ссылочной части expr является *const*, это тоже **игнорируется**. **Игнорируется** и модификатор *volatile*.

```cpp
template <typename T>
void f (T param);

int x = 27;
const int cx = x;
const int& crx = &x;

f (x);    // T - int, param - int
f (cx);   // T - int, param - int
f (crx);  // T - int, param - int

const char* const ptr =   // ptr - константный указатель на
    "Fun with pointers";  // константный объект
    
f (ptr);  // T - const char*, param - const char*
```

### Аргументы-массивы
```cpp
const char name[] = "Briggs";  // name - const char[7]
const char* ptrToName = name;  // Массив становится указателем
```

Следующие два объявления функции эквивалентны:
```cpp
void myFunc (int param[]);
void myFunc (int param*);
```

```cpp
template <typename T>
void f (T param);

f (name); // name - массив, но T - const char*
```

Хотя функции не могут объявлять параметры как истинные массивы, они **могут** объявлять параметры, являющиеся **ссылками** на массивы. 

```cpp
template <typename T>
void f (T& param);

f (name); // Немного наркотического бреда:
          // name - массив,
          // T - тип массива (включает размер массива: const char[7])
```

Благодаря этому можно создать шаблон, который будет возвращать размер массива:
```cpp
template <typename T, std::size_t N>
constexpr std::size_t size_arr (T(&arr)[N]) noexcept {
    return N;
}

char name[] = "Mask";
std::array <int, size_arr (name)> keys;
```

### Аргументы-функции

Типы функций могут превращаться в указатели на функции, и всё, что говорилось про вывод типа для массивов, применимо к выводу типа для функций и их преобразованием в указатели на функцию.

```cpp
void someFunc (int, double);

template <typename T>
void f1 (T param);

template <typename T>
void f2 (T& param);

f1 (someFunc); // T - void (*) (int, double)
f2 (someFunc); // T - void (&) (int, double)
```
### <center>Следует запомнить </center>

*  В процессе вывода типа шаблона аргументы, являющиеся ссылками, рассматриваются как ссылками не являющиеся, т.е. их *"ссылочность" игнорируется*.
*  При выводе типов для параметров, являющихся универсальными ссылками, lvalue-аргументы рассматриваются специальным образом.
*  При выводе типов для параметров, передаваемых по значению, аргументы, объявленные как *const* и/или *volatile*, рассматриваются как не являющиеся ни *const*, ни *volatile*.
*  В процессе вывода типа шаблона аргументы, являющиеся именами массивов или функций, преобразуются в указатели, если только они не использованы для инициализации ссылок.

## 1.2 Вывод типа auto

За одним любопытным исключением, вывод типа *auto* представляет собой вывод типа *шаблона*.

```cpp
auto x = 27;            // спецификатор типа - auto
const auto cx = x;      // спецификатор типа - const auto
const auto& rcx = x;    // спецификатор типа - const auto &
```

Компилятор действует так, как если бы для каждого объявления имелся шаблон, а также вызов этого шаблона с соответствующим инициализирующим выражением:

```cpp
template <typename T>        // Концептуальный шаблон для вывода типа x
void func_for_x (T param);

func_for_x (7);              // Концептуальный вызов: выведенный тип param - x

template <typename T>        // Концептуальный шаблон для вывода типа x
void func_for_cx (const T param);

func_for_cx (x);             // Концептуальный вызов: выведенный тип param - cx

template <typename T>        // Концептуальный шаблон для вывода типа x
void func_for_crx (const T& param);

func_for_crx (x);            // Концептуальный вызов: выведенный тип param - crx
```

В объявлении переменной с типом *auto* спецификатор типа занимает место *ParamType*, так что опять имеем три случая:
1. Указатель или  ссылка (не универсальная)
2. Универсальная ссылка
3. Ни указатель, ни ссылка

```cpp
// Случай 3 и 1
auto x = 27;          // x   - не указатель и не ссылка
const auto cx = x;    // cx  - не указатель и не ссылка 
const auto& crx = x;  // crx - неуниверсальная ссылка

// Случай 2
auto&& uref1 = x;     // x - int и lvalue, поэтому uref1 - int&
auto&& uref2 = cx;    // cx - const int и lvalue, поэтому uref2 - const int&
auto&& uref3 = 27;    // 27 - int и rvalue, поэтмоу uref3 - int&&
```
Аналогично для массивов и функций:

```cpp
const char name[] = "R. N. Briggs"; // name - const char[13]

auto  arr1 = name; // arr1 - const char *
auto& arr2 = name; // arr2 - const char &[13] 

void someFunc (int, double);

auto  func1 = someFunc; // func1 - void (*) (int, double)
auto& func2 = someFunc; // func2 - void (&) (int, double)
```

### Важная особенность auto

С появлением C++11 у нас есть 4 способа объявить int с начальным значением 27:
```cpp
int x1 = 27;
int x2(27);
int x3 = { 27 };
int x4{27};
``` 

Но уже для *auto* всё не так тривиально

```cpp
auto x1 = 27;        // x1 - int
auto x2(27);         // x2 - int
auto x3 = { 27 };    // x3 - initilizer_list <int>, занчение 27
auto x4{27};         // x4 - initilizer_list <int>, значение 27

```
Когда инициализатор для переменной, объявленной как *auto*, заключён в фигурные скобки, то выведенный тип - std::initializer_list.

```cpp
auto x5 = { 1, 2, 3.0 }; // Ошибка! Невозможно вывести тип T для std::initializer_list <T>
```

Это как раз и есть то самое единственное отличие вывода типа *auto* от вывода типа шаблона.

Если тот же инициализатор передаётся шаблону, то вывод типа оказывается неудачным.
```cpp
auto x = { 11, 23, 9 };  // x - std::initializer_list <int>

template <typename T>
void f (T param);

f ({ 11, 23, 9 });       // Ошибка вывода типа для T
```
Но если указать, что param представляет собой std::initializer_list <T>, то вывод типа шаблона определит тип Т:
```cpp
template <typename T>
void f (std::initializer_list <T> initList);

f ({ 11, 23, 9 });    // Вывод int в качестве T, а тип initList - std::initializer_list <int>
```

Таким образом, единственное реальное различие между выводом типа *auto* и выводом типа шаблоны заключается в том, что *auto* **предполагает**, что инициализатор в фигурных скобках представляет собой *std::initializer_list*, в то время как *вывод шаблона* **этого не делает**. 

*auto* может применяться для возвращающего типа и для объявления своих параметров в *лямбда-выражениях*. 
#### Однако такое применение *auto* использует вывод типа шаблона, а не вывод типа auto:

```cpp
auto createInitList () {
    return { 1, 2, 3 };    // Ошибка: невозможно вывести тип для { 1, 2, 3 }
}

// ...

std::vector <int> v;

auto resetV =
    [&v](const auto& newValue) { v = newValue; }; // C++14

resetV ({ 1, 2, 3 }); // Ошибка: невозможно вывести тип для { 1, 2, 3 }
```

### <center>Следует запомнить</center>

* Вывод типа *auto* обычно такой же, как и вывод шаблона, но вывод типа *auto*, в отличие от вывода типа шаблона, предполагает, что инициализатор в фигурных скобках представляет *std::initializer_list*.
* *auto* в возвращаемом типе функции или параметре *лямбда-выражения* влечёт за собой влечёт применение вывода типа шаблона, а не вывод типа *auto*.

## 1.3 Знакомство с decltype

Примеры без подводных камней:
```cpp
const int i = 0; // decltype (i) - const int

template <typename T>
class vector {
public:
    ...
    T& operator[] (std::size_t index);
    ...
};

vector <int> v; // decltype (v) - vector <int>

if (v[2] == 0) {} // decltype (v[2]) - int&
```

C++11 (требует уточнения)
```cpp
template <typename Container, typename Index>
auto authndAccess (Container& c, Index i)
    -> decltype (c[i])    // Завершающий ворзращаемый тип
{
    authenticateUser ();
    return c[i];
}
```

C++14 (не совсем корректно)
```cpp
template <typename Container, typename Index>
auto authndAccess (Container& c, Index i)
{
    authenticateUser ();
    return c[i];
}
```

Но если мы используем этот код, то получим ошибку компиляции:
```cpp
std::deque <int> d;
authAndAccess (d, 5) = 10;    // int rvalue присваевается 10 - не компилируется
```

C++14 (работает, но ещё требует уточнения)
```cpp
template <typename Container, typename Index>
decltype (auto) authndAccess (Container& c, Index i)
{
    authenticateUser ();
    return c[i];
}
```

Дело в том, что в предпоследнем примере используется вывод типа для auto, т.е. вывод типа шаблона. Тем самым c[i], который int&, превращался в int.
В последнем же примере используются правила вывода типа decltype. Т.е. decltype (auto) выведет int&.

*decltype (auto)* можно использовать для инициализирующего выражения:
```cpp
Widget w;
const Widget& crw = w;

auto myWidget1 = crw;            // myWidget1 - Widget        (вывод типа auto)

decltype (auto) myWidget2 = crw; // myWidget2 - const Widget& (вывод типа decltype)

```

Тогда возникает вопрос, что делать, если я хочу передать *c* как *rvalue*?
```cpp
std::deque <std::string> makeStringDeque (); // Фабричная функция
auto s = authAndAccess (makeStringDeque (), 5);
```

C++14 можно сделать так:
```cpp
template <typename Container, typename Index>
decltype (auto) authndAccess (Container&& c, Index i)
{
    authenticateUser ();
    return std::forward <Container> (c)[i];
}
```

Любопытный факт (для C++11):
```cpp
int x = 0;
decltype (x);  // int
decltype ((x); // int&
```

Но для C++14 всё становится серьёзнее:
```cpp
decltype (auto) f1 () {
    int x = 0;
    ...
    return x;    // decltype (x) - int
}

decltype (auto) f2 () {
    int x = 0;
    ...
    return (x);  // decltype ((x)) - int& - ссылка на локальный объект!!! Неопределённое поведение
}
```

Применение decltype к имени даёт тип этого имени. **Однако** для lvalue-выражений, более сложных, чем имена, decltype гарантирует, что возвращаемый тип всегда будет lvalue-ссылкой.

### <center>Следует запомнить</center>

* *decltype* почти всегда даёт тип переменной или выражения без каких-либо изменений.
* Для *lvalue*-выражений типа T, отличных от имени, decltype всегда даёт тип T&.
* C++14 поддерживает конструкцию *decltype (auto)*, которая, подобно *auto*, выводит тип из его инициализатора, но выполняет вывод типа с использованием правил *decltype*.

## 1.4 Как посмотреть выведенные типы

Мы рассмотрим получение информации и выводе типа:
* при редактировании кода
* во время компиляции
* во время выполнения

### Редакторы IDE
 В редакторах можно просто навести мышь на имя переменной
 
 ### Диагностика компилятора
 
Можно намеренно совершить ошибку, чтобы компилятор вывел сообщение об ошибке с типом переменной:
```cpp
const int theAnswer = 42;
auto x = theAnswer;
auto y = &theAnswer;

template <typename T>    // Только объявление TD
class TD;

TD <decltype (x)> xType; // Сообщение об ошибке
TD <decltype (y)> yType; // будет содержать типы x и y
```

### Вывод во времени исполнения

Правда это не особо удобно, нужно знать обозначения, да и ещё всё происходит во времени исполнения:
```cpp
std::cout << typeid (x).name () << std::endl;    // 'i'
std::cout << typeid (y).name () << std::endl;    // 'PKi'
```

Но, к сожалению, *std::type::info* не надёжен и может выдать неверный *name*.

Отличным решением будет использование библиотеки *Boost.TypeIndex*
```cpp
#include <boost/type_index.hpp>

template <typename T>
void f (const T& param)
{
    using std::cout;
    using boost::typeindex::type_id_with_cvr;
    
    // Вывод информации о T
    cout << "T =  "
         << type_id_with_cvr <T> ).pretty_name ()
         << std::endl;
         
     // Вывод информации о param
     cout << "param =  "
          << type_id_with_cvr <decltype param)>.pretty_name ()
          << std::endl;
     ...
}
```

### <center>Следует запомнить</center>
* Выводы типа часто модно просмотреть с помощью редакторов IDE, сообщений об ошибках компиляции и с использованием библиотеки Boost.TypeIndex.
* Результат, которые выдают некоторые инструменты, могут оказаться как неточными, так и бесполезными, так что понимание правил вывода типов в C++ является **совершенно необходимым**.





































