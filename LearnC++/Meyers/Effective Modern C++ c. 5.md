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


### 5.4 Избегайте перегрузок для универсальных ссылок

Как мы поняли из предыдущего раздела, нужно использовать *std::forward*, если это действительно нужно:
```cpp

std::multiset <std::string> names;

template <typename T>
void logAndAdd (T&& name) {
    auto now = std::chrono::system_clock::now ();
    lof (now, "logAndAdd");
    name.emplace (std::forward <T> (name));
}

std::string petName ("Darla");
logAndAdd (petName);    // копирование lvalue в multiset

logAndAdd (std::string ("Persephone"); // Перемещение rvalue вместо копирования

logAndAdd ("Patty Dog"); // Создание std::string, вместо копирования временного объекта std::string
```

Теперь добавим получение имени по индексу:
```cpp
std::string nameFromIndex (int index);

void logAndAdd (int index) {
    auto now = std::chrono::system_clock::now ();
    lof (now, "logAndAdd");
    name.emplace (nameFromIndex (index));
}

std::string petName ("Darla");            // Как и ранее

logAndAdd (petName);                      // Как и ранее
logAndAdd (std::string ("Persephone");    //
logAndAdd ("Patty Dog");                  //

logAndAdd (22);                           // Вызов int перегрузки
```

Но если мы сделаем совсем маленький шаг влево, то всё упадёт:
```cpp
short nameIndex = 4;
// ...
logAndAdd (nameIndex); // Ошибка!
```

В данном случае int перегрузка не является полным совпадением, поэтому выбирается перегрузка с универсальной ссылкой и пытается создаться std::string от int, но это невозможно, поэтому получаем ошибку компиляции.

Аналогично в следующем примере:
```cpp
class Person {
public:
    template <typename T>
    explicit Person (T&& name) :
        name (std::forward <T> (name))
    {}
    
    explicit Person (int index);
    
    // Неявно генерируются компилятором следующие конструкторы
    Person (const Person& rhs);
    Perosn (Person&& rhs);
    
private:
    std::string name;
};

Person person ("Nancy");
auto cloneOfP (person);    // Не компилируется
```

Тут всё просто: в *cloneOfP*попадает *person* как *Person*, затем мы должны создать объект *Person* из *person*. Смотрим внимательно: *person* - *Person&*, а не *const Person&*, следовательно конструктор копирования не выигрывает перегрузку, поэтому компилятор переходит к *Person (T&& name)*, но конструктора *std::string* от *Person* нет, как следствие получаем ошибку компиляции.

Т.е. происходит инстанцирование шаблонного конструктора для получения неконстантного lvalue типа Person:
```cpp
class Person {
public:
    explicit Person (Person& name) :            // Инстанцирован из
        name (std::forward <Person&> (name))    // шаблона с прямой
    {}                                          // передачей
};
```

Становится очевидным, как заставить работать предыдущий пример, - сделать полное соответствие для копирующего конструктора:
```cpp
const Person person ("Nancy");
auto clonOfP (person); // Копирующий конструктор
```

Здесь всё равно инстанцируется шаблонный конструктор, но по правилам разрешения перегрузок С++ выбирается нормальный конструктор копирования.

Также стоит быть осторожным при наследовании, потому что при наличии конструктора с прямой передачей, часто будет выбираться именно он и мы снова получим аналогичные ошибки компиляции:
```cpp
class SpecialPerson : public Person {
public:
    SpecialPerson (const SpecialPerson& rhs) :  
        Person (rhs)    // Вызовестя конструктор базового класса с прямой передачей! 
    {}
    
    SpecialPerson (SpecialPerson&& rhs) :
        Person (rhs)    // Аналогично
    {}
};
```

Но понятно, что будет инстанцирован *Person::Person (SpecialPerson&& rhs)*, в котором будет создаваться *std::string* от *SpecialPerson*, но такого конструктора нет, как следствие ошибка компиляции!

### <center>Следеут запомнить</center>
* Перегрузка для универсальных ссылок почти всегда приводит к тому, что данная перегрузка вызывается чаще, чем вы ожидаете
* Особенно проблематичны конструкторы с прямой передачей, поскольку они обычно соответствуют неконстантным *lvalue* лучше, чем копирующие конструкторы, и могут перехватывать вызовы из производного класса копирующих и перемещающих конструкторов базового класса.

### 5.5 Знакомство с альтернативами перегрузки для универсальных ссылок

Будут использоваться примеры из предыдущего раздела.

#### Отказ от перегрузки
Можно использовать разные имена: *logAndAddName и logAndAddNameIndex*. Но это не будет работать для конструктора *Person*.

#### Передача const T&
Может быть отказ от некоторой неэффективности в пользу простоты может оказаться более привлекательным компромиссом, чем казалось изначально.

#### Передача по значению
```cpp
class Person {
public:
    explicit Person (std::string name) :     // Замена конструктора с T&&;
        name (std::move (name))              // О применении std::move см. 8.1
    {}
    
    explicit Person (int index) :            // Как и ранее
        name (nameFromIndex (index))
    {}
};
```
По сути с C++11 lvalue объект будет копироваться, а rvalue создаваться конструктором перемещения

#### Диспетчеризация дескрипторов (tag dispatch)

Что если мы не можем отказаться от универсальных ссылок? Тогда попробуем переписать функцию *logAndAdd*, которая принимает как строки, так и *целочисленный* тип, используя *logAndAddImpl*:
```cpp
template <typename T>
void logAndAdd (T&& name) {
    logAndAddImpl (std::forward <T> (name),
                   std::is_integral <T> ()); // Не совсем корректно
}
```

Но если передать временный объект *int*, то *name* будет *int&*, то *std::is_integral <int&>* всегда будет выдать *false*, потому что это ссылка, а не целочисленный объект :
```cpp
// C++14
template <typename T>
void logAndAdd (T&& name) {
    logAndAddImpl (std::forward <T> (name),
                   std::is_integral <T> (
                       std::remove_reference_t <T>
                   ));
}
```

Теперь делаем перегрузку для *logAndAddImpl*:
```cpp
template <typename T>
void logAndAddImpl (T&& name, std::false_type) {
    auto now = std::chrono::system_clock::now ();
    log (now, "logAndAdd");
    names.emplace (std::forward <T> (name));
}

template <typename T>
void logAndAddImpl (int index, std::true_type) {
    logAndAddImpl (nameFromIndex (index));
}
```

В таком решении *std::true_type* и *std::false_type* являются *дескрипторами*. Вызов промежуточных функций реализации в logAndAdd "*диспетчеризирует*" передачу работы правильной перегрузке путём создания нужного объекта дескриптора. Отсюда и название этого метода проектирования: "*диспетчеризация дескрипторов*".

#### Ограничения шаблонов, получающих универсальные ссылки

Когда мы пользуемся диспетчеризацией дескрипторов, мы хотим, чтобы генерированные компилятором функции *всегда* обходили диспетчеризацию дескрипторов (например, например в конструкторе копирования, базовый класс выбирал тоже конструктор копирования).

Но в некоторых ситуациях, в которых перегруженная функция, принимающая универсальную ссылку, оказывается более "жадной", чем мы хотели, но недостаточно жадной, чтобы действовать как единственная функция диспетчеризации, метод диспетчеризации дескрипторов оказывается не тем, что требуется. Нам нужна другая технология, и это технология - *std::enable_if*.

**std::enable_if** даёт нам возможность заставить компиляторы вести себя так, как если бы определённого шаблона не существовало. Такие шаблоны называются *отключенными (disabled)*.

Простое использование:
```cpp
class Person {
public:
    template <typename T,
              typename = typename std::enable_if <условие>::type>
    explicit Person (T&& name);
    // ...
};
```

Наша задача в том, чтобы вызывать конструкторы копирования и перемещения *Person*, когда на входе тип *Person*, и конструктор с прямой передачей, если типы отличны. Но ведь на входе может быть и *const Person* и *volatile Person*. Да и вообще *T* будет lvalue, т.е. нужно убрать ссылку.

В итоге нам нужно избавиться от ссылки, const и volatile. Для этого нужно использовать *std::decay*. Тогда наше условие будет выглядеть так:
```cpp
!std::is_same <Person, typename std::decay <T>::type>::value
```

Добавим это в наш класс:
```cpp
class Person {
public:
    template <
        typename T,
        typename = typename std::enable_if <
                       !std::is_same <
                           Person, 
                           std::decay <T>::type
                       >::value
                   >::type
    >
    explicit Person (T&& name);
          
};
```

Это уже лучше, но что если от нашего класс отнаследуются и получат класс SpecialPerson : public Person (см. предыдущий раздел)? Тогда в копирующем конструкторе производного класса будет вызываться конструктор Person с прямой передачей, потому что проверка при помощи *std::is_same <Person, SpecialPerson>* даст ложь. Нужно сказать компилятору, чтобы перегрузка срабатывала и для производных классов. Для этого есть *std::is_base_of*.

**std::is_base_of <T1, T2>::value** истинно, если *T2* производный от *T1*. Пользовательские типы рассматриваются как производные от самих себя, т.е. *std::is_base_of <T, T>::value* будет *true*, если *T*-пользовательский тип, и *false*-если встроенный. Это удобно и позволяет заменить *is_same* в некоторых случаях.

Вот так будет выглядеть наш класс: 
```cpp
// C++11
class Person {
public:
    template <
        typename T,
        typename = typename std::enable_if <
                       !std::is_base_of <
                           Person, 
                           std::decay <T>::type
                       >::value
                   >::type
    >
    explicit Person (T&& name);
          
};
```

В C++14 это будет выглядеть элегантнее (без лишних typename и ::type):
```cpp
class Person {
public:
    template <
        typename T,
        typename = std::enable_if_t <
                       !std::is_base_of <
                           Person, 
                           std::decay_t <T>
                       >::value
                   >
    >
    explicit Person (T&& name);
          
};
```

Это уже почти работает. Почти - потому что мы немного отвлеклись от нашей первоначальной задачи: нам ведь нужно было принимать в перегрузке на int только целочисленные типы, так добавим этот ингредиент в наш суп:
```cpp
class Person {
public:
    template <
        typename T,
        typename = std::enable_if_t <
           !std::is_base_of <Person, std::decay_t <T>>::value
           &&
           !std::is_integral <std::remove_reference_t <T>>::value
       >
    >
    explicit Person (T&& name);
    
    explicit Person (T&& name) :  
        name (std::forward <T> (name))
    {}
    
    explicit Person (int index) :
        name (nameFromIndex (index))
    {}
    
    // ... Копирующий и перемещающий конструктор

private:
    std::string name;
};
```

Получилось очень круто! Мы использовали все преимущества универсальных ссылок (прямая передача => эффективность) и перегрузок, вместо их запрета.

#### Компромиссы

Есть определённая проблема в предыдущем подходе. Дело в том, что если случайно допустить ошибку при использовании *Person*, например создать его не от char, а от char16_t, *Person person (u"Konrad Zuse");*, то получим от компилятора сообщение об ошибке более чем на *160 строк*! В других предыдущих вариантах мы получим довольно простое сообщение об ошибке. Учтём, что наш класс *Person* ещё очень прост, а что если прямая передача проходила бы через несколько слоёв? Тогда ошибки совершенно неадекватные. На помощь может прийти использование *static_assert* и *std::is_constructible*:
```cpp
class Person {
public:
    template <
        typename T,
        typename = std::enable_if_t <
           !std::is_base_of <Person, std::decay_t <T>>::value
           &&
           !std::is_integral <std::remove_reference_t <T>>::value
       >
    >
    explicit Person (T&& name);
    
    explicit Person (T&& name) :  
        name (std::forward <T> (name))
    {
        static_assert (
            std::is_constuctible <std::string, T>::value,
            "Параметр name не может быть использован для"
            "конструирования std::string"
        );
    
        // ... Здесь идёт код обычного конструктора
    }
    
    // ... Остальная часть класса
};
```

В случае ошибки, мы получим наше сообщение сразу *после* обычных сообщениях об ошибках (более 160 строк).

### <center>Следует запомнить</center>
* Альтернативы комбинации универсальных ссылок и перегрузки включают использование различных имён, передачу параметров как lvalue-ссылок на const, передачу параметров по значению и использование диспетчеризации дескрипторов.
* Ограничение шаблонов с помощью *std::enable_if* позволяет использовать универсальные ссылки и перегрузки совместно, но управляет условиями, при которых компиляторы могут использовать перегрузки с универсальными ссылками.
* Параметры, представляющие собой универсальные ссылки, часто имеют преимущества высокой эффективности, но обычно их недостатком является сложность использования.



























