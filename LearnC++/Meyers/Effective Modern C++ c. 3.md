3\. Переход к современному C++
===

## 3.1 Различие между {} и  () при создании объектов

С++ трактует следующие выражении одинаково:
```cpp
int z { 0 };    // То же
int z = { 0 };  // самое
```

Для устранения путаницы из-за нескольких синтаксисов инициализации и решения проблемы охвата всех сценариев инициализации C++11 вводит *унифицированную инициализацию (uniform initialization)*: единый синтаксис инициализации, который может, как минимум концептуально, использоваться везде и выражать всё. Он основан на фигурных скобках: *"фигурная инициализация" (braced initialization)*.

Примеры:
```cpp
std::vector <int> v { 1, 3, 5 };
```

```cpp
class Widget {
...
private:
    int x { 0 };  // Ок, значение по умолчанию равно 0
    int y = 0;    // Тоже ок
    int z (0);    // Ошибка!
};
```

```cpp
std::atomic <int> ail0 { 0 }; // Ок
std::atomic <int> ail1 (0);   // Ок
std::atomic <int> ail2 = 0;   // Ошибка!
```

Т.е. из трёх способов обозначения выражений инициализации только фигурные скобки могут использоваться везде.

Новая возможность фигурной инициализации заключается в том, что она запрещает неявные *сужающие преобразования* среди встроенных типов:
```cpp
double x = 0, y = 0, z = 0;

int sum { x + y + z }; // Ошибка! Сумма double не выражется с помощью int
```

Правда в данном случае gcc кинет Warning, но не более.

Инициализация с использование круглых скобок и знака равенства не выполняет проверку сужающего преобразования:

```cpp
int sum1 (x + y + z);
int sum2 = x + y + z;
```

Фигурные скобки решают проблему *наиболее неприятного анализа*:
```cpp
Widget w0 (10);   // Вызов конструктора
Widget w1 ();     // Объявление функции w1
Widget w2 {};     // Конструктор по умолчанию
```

Однако не забываем, про *std::initializer_list* при *auto {}*.

Без std::initializer_list:
```cpp
class Widget {
public:
    // Конструкторы без std::initializer_list
    Widget (int i, bool b);
    Widget (int i, double d);
};

Widget w0 (10, true);    // Вызов первого конструктора
Widget w1 {10, true};    // Вызов первого конструктора
Widget w2 (10, 3.14);    // Вызов второго конструктора
Widget w3 {10, 3.14};    // Вызов второго конструктора
```

С std::initializer_list:
```cpp
class Widget {
public:
    Widget (int i, bool b);    // Как и ранее
    Widget (int i, double d);  // Как и ранее

    Widget (std::initializer_list <long double> i1); // Добавлен
};

Widget w0 (10, true);    // Вызов первого конструктора
Widget w1 {10, true};    // Вызов третьего конструктора (10, true -> long double)
Widget w2 (10, 3.14);    // Вызов второго конструктора
Widget w3 {10, 3.14};    // Вызов третьего конструктора (10, 3.14 -> long double)
```

Пример с копированием и перемещением:
```cpp
class Widget {
public:
    Widget (int i, bool b);    // Как и ранее
    Widget (int i, double d);  // Как и ранее

    operator float () const;   // Преобразование во float
};

Widget w4 (w3);                // Копирующий конструктор
Widget w5 {w3};                // Конструктора с std::initializer_list
                               // (w3 преобр. в float, а float преобр. в long double)
Widget w6 (std::move (w3));    // Перемещающий конструктор
Widget w7 {std::move (w3)};    // Аналогично w5
```

Мощный пример. Присмотритесь к *bool*. Тут происходит сужение в фигурных скобках, что запрещено C++:
```cpp
class Widget {
public:
    Widget (int i, bool b);    // Как и ранее
    Widget (int i, double d);  // Как и ранее

    Widget (std::initializer_list <bool> (i)); // Теперь тип элемента - bool

    // Нет функций неявного преобразования
};

Widget w (10, 3.14);    // Ошибка! Требуется сужающее преобразование
```

Тут int и double *можно было* неявно преобразовать в bool, но тогда это было бы сужающее преобразование.

Но если мы сделаем так, что не будет никакой возможности преобразовать int и double в новый тип, то тогда вызовутся нужные нам конструкторы:
```cpp
class Widget {
public:
    Widget (int i, bool b);    // Как и ранее
    Widget (int i, double d);  // Как и ранее

    Widget (std::initializer_list <std::string> (i)); // Теперь тип элемента - std::string

    // Нет функций неявного преобразования
};

Widget w0 (10, true);    // Вызов первого конструктора
Widget w1 {10, true};    // Вызов первого конструктора
Widget w2 (10, 3.14);    // Вызов второго конструктора
Widget w3 {10, 3.14};    // Вызов второго конструктора
```

Теперь рассмотрим вызовы без аргументов:
```cpp
class Widget {
public:
    Widget (int i, bool b);    // Как и ранее
    Widget (int i, double d);  // Как и ранее

    Widget (std::initializer_list <int> (i));

    // Нет функций неявного преобразования
};

Widget w0;    // Вызов конструктора по умолчанию
Widget w1 {}; // Вызов конструктора по умолчанию
Widget w2 (); // Трактуется как объявление функции!!!
```

#### Пустые фигурные скобки означают отсутствие аргументов, а не пустой std::initializer_list

Если *хотите* вызвать конструктор с пустым std::initializer_list, то:
```cpp
Widget w3 ({}); // Вызов конструктора с пустым std::initializer_list
Widget w4 {{}}; // То же самое
```

И это достаточно повседневная практика. Давайте рассмотрим *std::vector <int>*:
```cpp
std::vector <int> v1 (10, 20); // Size: 10, каждый элемент имеет значение 20
std::vector <int> v2 {10, 20}; // Size:  2, первый равен 10, второй 20
```

А теперь крайне интересный пример. Предположим, что мы хотим создать объект произвольного типа с произвольным количеством аргументов:
```cpp
template <typename T,        // Тип объекта
          typename... Ts>    // Типы аргументов
void doSomeWork (Ts&&... params) {
    // Создание локального объекта
}
```

Рассмотрим два способа создания локального объекта:
```cpp
T localObject (std::forward <Ts> (params)...);    // Круглые скобки
T localObject {std::forward <Ts> (params)...};    // Фигурные скобки
```

А вот самое интересное:
```cpp
std::vector <int> v;

doSomeWork <std::vector <int>> (10, 20);
```

Если локальный объект создавался при помощи круглых скобок, то создаться вектор из 10 элементов, если из фигурных, что из элементов 10 и 20.

### <center>Следует запомнить</center>
* Фигурная инициализация является наиболее широко используемым синтаксисом инициализации, предотвращающим сужающие преобразование и нечувствительным к особенностям синтаксического анализа C++.
* В процессе разрешения перегрузки конструкторов фигурные инициализаторы соответствуют параметрам *std:initializer_list*, если это возможно, даже если другие конструкторы обеспечивают лучшее соответствие.
* Пример, в котором выбор между круглыми и фигурными скобками приводит  к значительно отличающимся результатам, является создание *std::vector <числовой тип>* с двумя аргументами.
* Выбор между фигурными или круглыми скобками для создания объектов внутри шаблонов может быть очень сложным.

## 3.2 Предпочитайте *nullptr* значениям *0* и *NULL*

Тип *nullptr* - *std::nullptr_t*. *std::nullptr_t* неявно преобразуется во все типы указателей.

Важно понимать, что у *0* тип *int*, а у *NULL* либо *int*, либо *void**.  Поэтому иногда мы хотим вызвать функцию с аргументом NULL, которая принимает void*, а в действительности срабатывает перегрузка и вызывается функция с таким же именем, но принимающая int. Жуть.

Рассмотрим примеры, которые объяснят, почему лучше пользоваться *nullptr*, вместо *0* и *NULL*
```cpp
int func_sp (std::shared_ptr <Widget> spw);
int func_up (std::unique_ptr <Widget> upw);
int func_np (Widget* pw); // np - normal pointer

auto res_sp = func_sp (0);        // Это сработает, грустно
auto res_up = func_up (NULL);     // И это тоже сработает
auto res_np = func_np (nullptr);  // А это понятно, что сработает
```

Как видим это всё работает, но давайте напишем это на шаблонах:
```cpp
template <typename FuncType, typename PtrType>
decltype (auto) func (FuncType func, PtrType ptr) {    // C++14
    return func (ptr);
}
```

Теперь если мы запустим также, как в предыдущий раз, то получим что-то гораздо лучше:
```cpp
auto res_sp = func (func_sp, 0);        // Ошибка!
auto res_up = func (func_up, NULL);     // Ошибка!
auto res_np = func (func_np, nullptr);  // Ок
```

Это происходит потому, что *int* не совместим с типом *std::shared_ptr*. А *std::nullptr_t* совместим с любым указателем.

### <center>Следует запомнить</center>
* Предпочитайте применение *nullptr* использованию *0* или *NULL*
* Избегайте перегрузок с использованием целочисленных типов и типов к=указателей.

## 3.3 Предпочитайте объявление псевдонимов применению typedef

Сокращения имён типов в C++ можно сделать двумя способами:
```cpp
typedef
    std::unique_ptr <std::unordered_map <std::string, std:string>>
    UPtrMapSS;
```
```cpp
using UPtrMapSS = std::unique_ptr <std::unordered_map <std::string, std:string>>;
```

Это в точности одно и тоже.
Различия начинаются, когда дело доходит до шаблонов:
```cpp
template <typename T>
using MyAllocList = std::list <T, MyAlloc <T>>;

MyAllocList <Widget> lw; // Кратко понятно
```
```cpp
template <typename T>
struct MyAllocList {
    typedef std::list <T, MyAlloc <T>> type;
};

MyAllocList <Widget>::type lw; // Добавляется лишнее слово ::type
```

Но следующий пример становится действительно расточительным для *typedef*:
```cpp
template <typename T>
class Widget {
private:
    typename MyAllocList <T>::type list;
};
```

Здесь *MyAllocList \<T\>::type* ссылается на тип, который зависит от параметра типа шаблона (T). Тем самым MyAllocList \<T\>::type* является *зависимым типом (depended type)*, а одно из милых правил C++ требует, чтобы имена зависимых типов представлялись ключевым словом *typename*.

Использую псевдоним, всё становится куда лучше:
```cpp
template <typename T>
using MyAllocList = std::list <T, MyAlloc <T>>;

template <typename T>
class Widget {
private:
    MyAllocList <T> list; // Ни typename, ни ::type
};
```

#### TMP - template metaprogramming
#### Свойства типов - type traits

Набор шаблонов в заголовочном файле <type_traits>:
```cpp
// C++11
std::remove_const <T>::type;
std::remove_reference <T>::type;
std::add_lvalue_reference <T>::type;

// C++14
std::remove_const_t <T>;
std::remove_reference_t <T>;
std::add_lvalue_reference_t <T>;
```

Также для C++14 не нужно будет писать *typename* для каждого использования подобных конструкций, т.к. это всё псевдонимы, а *std::\*\*\* \<T\>::type* - *typdef*.

Вообще это всё пишется самому очень просто:
```cpp
template <class T>
using remove_const_t = typename remove_const <T>::type;
```

### <center>Следует запомнить</center>
* В отличие от псевдонимов, *typedef* не поддерживает шаблонизацию
* Шаблоны псевдонимов не требуют суффикса *"::type"*, а в шаблонах префикса *typename*, часто требуемого при обращении к *typedef*
* C++14 предлагает шаблоны псевдонимов для всех  преобразований свойств типов C++11.

## 3.4 Предпочитайте перечисления с областью видимости перечислениям без таковой

```cpp
enum Color {black, white, red}; // Без области видимости (Unscoped)
enum class Color {
    black, white, red           // С областью вилимостью (Scoped enum)
};
```

У *enum class* нет неявных преобразований в числа. Они существенно строже типизированы, нежели *enum*.

Можно использовать явное приведение типа для исполнения своих грязных желаний:
```cpp
Color c = Color::red;
auto num_red = static_cast <double> (x);
```

Если *enum* объявлен в каком-то важно заголовочном файле и вы добавили в *enum* новое поле, то вам придутся перекомпилировать всю систему полностью. Это очень плохо!

Но если же вы используете *enum class*, то будет работать даже следующий пример:
```cpp
enum class Status;
void continueProcessing (Status s);
```

Но как компилятор знает размер *enum class*? Для *enum* всё понятно, его можно только определить, но ведь *enum class* можно ведь и просто объявить. Ответ прост: он заранее известен. По умолчанию для *enum class* он *int*, но мы можем его перекрыть:
```cpp
enum class Status : std::uint32_t; // Базовый тип лоя Status - std::uint32_t
```

Но иногда *enum* полезен. Мы можем использовать его для адекватного (не нужно хранить числа в голове программиста) и немногословного доступа к элементам кортежа:
```cpp
using UserInfo =
    std::tuple <std::string,    // Имя
                std::string,    // Адрес
                std::size_t>;   // Репутация

UserInfo uInfo;

auto val = std::get <1> (uinfo); // Получение значения поля 1
```

Добавим *enum* для удобства:
```cpp
enum UserInfoFields { uiName, uiEmail, uiReputation };

UserInfo uInfo;

auto val = std::get <uiEmail> (uInfo);
```

С *enum class* будет будет громоздко, но зато глобальное пространство имён не будет засорено.

Можно написать в лоб, но это так себе вариант. Он громоздок в *использовании* и при это кастует в тип std::size_t, а не в тип, от которого наследован *enum class*:

```cpp
enum class UserInfoFields { uiName, uiEmail, uiReputation };

UserInfo uInfo;

auto val = std::get <static_cast <std::size_t> (UserInfoFields::uiEmail)> (uInfo);
```

Давайте лучше напишем общую функцию из *enum class* в целочисленный тип, от которого он унаследован (для этого нужно использовать *std::underlyinf_type_t*). Во-первых, std::get, принимает аргумент шаблона, т.е. значение должно должно быть известно на *этапе компиляции*. Поэтому функция должна быть *constexpr*. В итоге:
```cpp
// C++ 11
template <typename E>
constexpr typename std::underlying_type <E>::type
toUType (E enumerator) noexcept
{
    return static_cast <typename std::underlying_type <E>::type> (enumerator);
}
```

В C++14, конечно, это будет выглядеть изящней:
```cpp
template <typename E>
constexpr auto
toUType (E enumerator) noexcept
{
    return static_cast <std::underlying_type_t <E>> (enumerator);
}
```

Теперь мы можем обращаться к полю кортежа:
```cpp
auto val = std::get <toUType (UserInfoFields::uiEmail)> (uInfo);
```

### <center>Следует запомнить</center>
* Перечисление в стиле C++98 в настоящее время известны как перечисления без области видимости
* Перечислители перечислений с областями видимости видимы только внутри перечислений. Они преобразуются в другие типы только с помощью явных приведений.
* Как перечисления с областями видимости, так и без таковых поддерживают указание базового типа. Базовым типом по умолчанию с областью видимости является *int*. Перечисления без области видимости базового типа не имеют.
* Перечисления с областями видимости могут быть предварительно объявлены. Перечисления без областей видимости могут быть предварительно объявлены,  только если их объявление указывает на базовый тип.

## 3.5 Предпочитайте удалённые (deleted) функции закрытым неопределённым

Иногда нужно, чтобы некоторые автоматически создаваемые функции-члены класса или просто функции нельзя было использовать. Например, конструктор копирования или оператор присваивания. Вот два способа в стилях C++98 и C++11:
```cpp
// C++98
template <class charT, class traits = char_traits <charT>>
class basic_ios : public ios_base {
public:
    // ...

private:
    basic_ios (const basic_ios&);               // Не определён
    basic_ios& operator= (const basic_ios&);    // Не определён
};
```

```cpp
// C++11
template <typename charT, class traits = char_traits <charT>>
class basic_ios : public ios_base {
public:
    // ...

private:
    basic_ios (const basic_ios&) = delete;
    basic_ios& operator= (const basic_ios&) = delete;
};
```

Здесь можно отметить одну деталь: *delete* функции лучше выносить в *public* часть, чтобы при ошибке компилятор говорил, что функция специально удалена, а не то, что она недоступна, потому что находится в *private* части.

Также если мы не хотим, чтобы были какие-то *обычные* функции, то в C++11 мы можем просто их удалить:
```cpp
bool isLucky (int number);
bool isLucky (char) = delete;
bool isLucky (bool) = delete;
bool isLucky (double) = delete; // double захватит и float функцию
```

Также можно удалять специализации:
```cpp
template <typename T>
void processPointer (T* ptr);

template <>
void processPointer <void> (void* ptr);

template <>
void processPointer <const void> (const void* ptr);

template <>
void processPointer <volatile void> (volatile void* ptr);
```

И в отличии от C++98 в C++11 можно удалять специализации  функций, объявленных как public:
```cpp
class Widget {
public:
    template <typename T>
    void processPointer (T* ptr)
    {
        //...
    }
};

template <>
void Widget::processPointer <void> (void*) = delete;
```

### <center>Следует запомнить</center>
* Предпочитайте удалённые функции закрытым функциям без определений.
* Удалённой может быть любая функция, включая функции, не являющиеся членами, и инстанцирования шаблонов.

## 3.6 Объявляйте перекрывающие функции как *override*

Для осуществления перекрытия требуется выполнение нескольких условий:
* Функция базового класса должна быть виртуальной
* Имена функций в базовом и производном классах должны быть одинаковыми (за исключением деструктора)
* Типы параметров функций в базовом и производном классах должны быть одинаковыми
* Константность функций в базовом и производном классах должна совпадать
* Возвращаемые типы и спецификации исключений функций в базовом и производном классах должны быть совместимыми
* (С++11) *Ссылочные квалификаторы* функций должны быть идентичными (void func () *&*;)

Получается, что маленькие ошибки могут привести к большим последствиям.

C++11 даёт возможность явно указать, что функция производного класса предназначена для того, чтобы перекрывать функцию из базового класса: её надо объявить как *override*.

```cpp
class Base {
    public:

    virtual void doWork () const;
};

class Derived : public Base {
public:
    // void doWork (); // Не скомпилируется

    void doWork () const override
    { //... }
};
```

*override* может помочь оценить последствия изменения сигнатуры виртуальной функции базового класса. После изменения и компиляции можно просто посмотреть сколько классов перестали компилироваться. Без *override* нужно было было надеяться на достаточное количество тестов.

*final* не даёт наследоваться от класса или перекрывать функцию в производном классе.

Для того, чтобы различать, является ли <em>*this<em> lvalue или rvalue объектом и вызывать соответствующие функции, нужно использовать *ссылочный квалификатор*:
```cpp
class Widget {
public:
    using DataType = std::vector <double>;
    // ...

    DataType& data () &        // Для lvalue Widget возвращает lvalue
    {
        return values;
    }

    DataType&& data () &&      // Для rvalue Widget возвращает rvalue
    {
        return std::move (values);
    }

    // ...

private:
    DataType values;
};
```

### <center>Следует запомнить</center>
* Объявляйте перекрывающие функции как *override*.
* Ссылочные квалификаторы функции-члена позволяют по-разному рассматривать lvalue- и rvalue-объекты (*this).


## 3.7 Предпочитайте итераторы *const_iterator* итераторам *iterator*

В C++98 крайне непрактично сделаны функции, принимающие итераторы. Так что сразу перейдём к практичному C++11 и "доведённому до ума" в некоторых мелочах C++14.

Нужно в местах, где мы не собираемся изменять объект, использовать *const_iterator*:
```cpp
std::vector <int> values;

// ...

auto it = std::find (values.cbegin (),    // cbeing, вместо begin
                     values.cend (),      // cend,   вместо end
                     1983);

values.insert it, 1983);                  // Принимает const_iterator
```

*Максимально обобщённый код использует функции, не являющиеся членами, а не предполагает наличие функций-членов.* Обобщим предыдущий код:
```cpp
template <class C, typename V>
void FindAndInsert (C container,
                    const V& targetValue,
                    const V& insertValue)
{
    auto it = std::find (std::cbegin (container),    // Не член cbegin
                         std::cend   (container),    // Не член cend
                         targetValue);

    container.insert (it, insertValue);
}
```

Этот код прекрасно работает в C++14, но не работает в C++11, потому что в C++11 были добавлены только *begin*, *end*, но не были добавлены *cbegin*, *cend*, *rbegin*, *rend*, *crbegin*, *crend*.

Но мы с лёгкостью можем написать сами недостающие функции в С++11, имея только std::begin:
```cpp
template <class C>
auto cbegin (const C& container) -> decltype (std::begin (container))
{
    return std::begin container;
}
```

Этот пример работает для всего, даже для встроенных массивов.

### <center>Следует запомнить</center>
* Предпочитайте использовать *const_iterator* вместо *iterator* там, где это можно.
* В максимально обобщённом коде предпочтительно использовать версии функции *begin*, *end*, *rbegin* и прочих, не являющиеся членами.

## 3.8 Если функция не генерирует исключений, то объявляйте их как *noexcept*

Отсутствие объявления функции как *noexcept*, когда вы точно знаете, что она не в состоянии генерировать исключения, - не более чем просто *плохая спецификация интерфейса*.

```cpp
int func (int num) throw (); // func не генерирует исключений: C++98
int func (int num) noexcept; // func не генерирует исключений: C++11
```

В C++98 при исключении произойдёт сворачивание стека до вызывающего func кода, и после некоторых действий, выполнение программы прекращается.

В С++11 поведение времени выполнение несколько иное: стек только, *возможно*, сворачивается перед завершением программы.

Разница между точным и возможным сворачиванием стека даёт удивительную разницу в генерации кода: при возможном не нужно держать стек в сворачиваемом состоянии, не нужно в правильно уничтожать объекты. Т.е. *noexcept* легче оптимизировать:
```cpp
RetType function (params) noexcept; // Наиболее оптимизируетма
RetType function (params) throw (); // Менее оптимизируема
RetType function (params);          // Менее оптимизируема
```

Используя семантику перемещения, иногда можно существенно оптимизировать программу, но это возможно только в том случае, когда выполнена строгая гарантия безопасности исключений. Т.е перемещающие операции не должны генерировать исключений. Единственный нормальный способ сказать, что функция не будет генерировать исключений, это *noexcept*. (Проверка обычно выполняется окольным путём: std::move_if_noexcept -> std::is_nothrow_move_constructible)

Также можно создавать функции, являющиеся *условно noexcept*:
```cpp
template <class C, std::size_t N>
void swap (C (&a)[N],
           C (&b)[N]) noexcept (noexcept (swap (*a, *b));

template <typename T>
struct pair {
    void swap (pair& p) noexcept (noexcept (swap (first,  p.first)) &&
                                    noexcept (swap (second, p.second)));
};
```

Нужно иметь в виду, что большинство функций *нейтральны по отношению к исключениям*: они могут вызывать такие функции, которые генерируют исключения, а исключения проходят дальше из этой функции. Такие функции не могут быть *noexcept*.

В С++11 стало правилом языка, что нельзя генерировать исключения функциям освобождения памяти (т.е. операторам *operator delete* и *operator delete[]*) и деструкторам.

Можно сурово пошутить над коллегами, объявив деструктор как *noexcept (false)* и кинуть исключение из него. Тогда будет неопределённое поведение))

При проектировании различают функции с *широким контрактом* от функций с *узким контрактом*. Функция с *широким контрактом* не имеет предусловий. Она может быть вызвана в любом месте, в любой ситуации (кроме уже неопределённого) и в любом состоянии. На её аргументы не накладываются никаких ограничений. Функции с *широким контрактом* никогда не демонстрируют неопределённого поведения. Если предусловия *узких контрактов нарушены*, их результаты являются неопределёнными.

И если всё понятно с функциям с *широкими контрактами*, где мы просто пишем *noexcept*, то с функциями с *узкими контрактами не всё так однозначно*. Мы можем, например, принимать std::string только длины до 32 символов. Если длина будет больше, то это нарушение предусловия приводит *по определению* к неопределённому поведению. Но раз мы точно уверены, что будут строки короче 32 символов, и при этом имеются тесты на это предусловие, то мы спокойно можем писать *noexcept*, хотя в общем случае она таковой не является:
```cpp
void func const std::string& str) noexcept; // Предусловие: str.length () <= 32
```

Также компилятор не обязан выдавать ошибку в случае, если функция объявленная как *noexcept*, вызывает не *noexcept* функции, потому что они могут быть строго задокументированы как не генерирующие исключений, либо эти функции могут быть частью С интерфейса, либо частью стандартной библиотеки С++98.

### <center>Следует запомнить</center>
* *noexcept* является частью интерфейса функции, а это означает, что вызывающий код может зависеть от данного модификатора.
* Функции, объявленные как *noexcept*, предоставляют б*о*льшие возможности оптимизации, чем без такой спецификации.
* Спецификация *noexcept* имеет особое значение для операций перемещения, обмена, функций освобождения памяти и деструкторов.
* Большинство функций *нейтральны по отношению к исключениям*, а не являются *noexcept*-функциями.

## 3.9 Используйте, где это возможно, *constexpr*

constexpr объекты являются константными и *известными во время компиляции*. (Технически их значения определяются во время *трансляции*, а трансляция состоит не только из компиляции, но и из компоновки).
```cpp
int size = 10; // Неконстантная переменная!!

// ...

constexpr auto arraySize1 = size;     // Ошибка! Значение size не известно во время компиляции
std::array <int, size> data1;         // Ошибка! Значение size не извеснто во время компиляции

constexpr auto arraySize2 = 10;       // OK, 10 - константа времени компиляции
std::array <int, arraySize2> data2;   // OK, arraySize2 - constexpr
```

У *const* нет гарантии того, что он должен быть инициализирован во время компиляции:
```cpp
int size = 10; // Неконстантная переменная!!

// ...

const auto arraySize1 = size;     /* Ошибка! Значение size не известно во время компиляции
                                     arraySize1 является константной копией size */
std::array <int, size> data1;     // Ошибка! Значение size не извеснто во время компиляции
```

*Функции constexpr* производят константы времени компиляции, *когда они вызываются с константами времени компиляции*. Иначе они производят значения времени выполнения.

То, что *constexpr функция* может выполняться не только на этапе компиляции, но и в ран-тайме, делает *constexpr функции* кране практичными и удобными: не нужно писать две версии одной и той же функции.

Попробуем написать функцию *pow* для целочисленных аргументов используя *constexpr*:
```cpp
// C++11
constexpr int pow (int base, int exp) noexcept
{
    return (exp == 0) ? 1 : base * pow (base, exp - 1);
}
```

Это версия, написанная на С++11. Достаточно функциональный код.

Теперь попробуем уже написать на С++14:
```cpp
// C++14
constexpr pow (int base, int exp) noexcept
{
    auto result = 1;
    for (int i = 1; i < base; ++i)
        result *= base;

    return result;
}
```

Этот гораздо лучше. Здесь уже видно огромное преимущество *constexpr* в С++14 над С++11.

Но есть ограничения в С++14: функции *constexpr* ограничены приёмом и возвратом только *литеральных типов* (*literal types*).

Также можно создавать *constexpr* функции-члены, конструкторы в классе:
```cpp
class Point {
public:
    constexpr Point (double xVal = 0, yVal = 0) noexcept :
        x (xVal), y (yVal)
    {}

    constexpr double xValue () const noexcept { return x; }
    constexpr double yValue () const noexcept { return y; }

    void setX (double newX) noexcept { x = newX; }
    void setY (double newY) noexcept { y = newY; }

private:
    double x, y;
};
```

Это очень сильно, ведь теперь можно многое выполнять во время компиляции. Например, можно написать функцию, которая использует объекты этого класса и вызывает только *constexpr* методы:
```cpp
constexpr Point midpoint (const Point& point1, const Point& point2) noexcept
{
    return Point ((point1.xValue () + point2.xValue ()) / 2,
                  (point1.yValue () + point2.yValue ()) / 2);
}

constexpr Point point1 (9.4,  27.7);    // Во время
constexpr Point point2 (28.8, 5.3);     // компиляции

constexpr auto mid = midpoint (p1, p2); // Тоже во время компиляции
```

В C++11 есть два ограничения (например, которые не дают создать методы setX, setY):
* *constexpr* функция-член неявно является *const*, поэтому методы не могут модифицировать класс
* Возвращают только литеральные типы, а *void* таким не является

Оба эти ограничения сняты в С++14. Даже функции с установкой полей могут быть объявлены как *constexpr*! Это даёт возможность писать функции, которые модифицируют объект:
```cpp
constexpr Point reflection (const Point& point) noexcept
{
    Point result;
    result.setX (-point.xValue ());
    result.setY (-point.yValue ());
    return result;
}

constexpr auto reflectionMid = reflection (mid); // (-19.1, -16.5) - известно во время компиляции
```

### <center>Следует запомнить</center>
* Объекты *constexpr* являются константными и инициализируются объектами, значения которых известны во время компиляции.
* Функции *constexpr* могут производить результаты времени компиляции при вызове с аргументами, значения которых известны во время компиляции.
* Объекты и функции *constexpr* могут использоваться в более широком диапазоне контекстов по сравнению с объектами и функциями, не являющимися *constexpr*.
* *constexpr* является частью интерфейса объектов и функций

## 3.10 Делайте константные функции-членов безопасными в смысле потоков

Рассмотрим класс полинома, в котором для эффективности сделано кэширование значений вычисления корней полинома:
```cpp
class Polynomial {
public:
    using RootsType = std::vector <double>;

    RootsType roots () const
    {
        if (!rootsAreValid) {
            // Вычисляемм корни

            rootsAreValid = true;
        }

        return rootVals;
    }

private:
    mutable bool rootsAreValid {false};
    mutable RootsType rootVals {};
};
```

Но если этот *const* метод *root* вызовут одновременно  два потока, то возникнет состояние гонки. Поэтому стоит модифицировать этот класс *мьютексом*:
```cpp
class Polynomial {
public:
    using RootsType = std::vector <double>;

    RootsType roots () const
    {
        std::lock_guard <std::mutex> guard (mutex);    // Блокировка
        if (!rootsAreValid) {
            // Вычисляемм корни

            rootsAreValid = true;
        }

        return rootVals;                               // Разблокирование
    }

private:
    mutable std::mutex mutex;
    mutable bool rootsAreValid {false};
    mutable RootsType rootVals {};
};
```

Этот код работает, но что если мы бы хотели использовать *std::atomic* для кэша и индикатора кэширования?
```cpp
class Widget {
public:
    int magicValue const
    {
        if (cacheValid)
            return cachedValue;
        else {
            auto value1 = expensiveComputation1 ();
            auto value2 = expensiveComputation2 ();

            cachedValue = value1 + value2;    // Сложение
            cacheValide = true;               // Установка индикатора

            return cachedValue;
        }
    }

private:
    mutable std::atomic <bool> cacheValid {false};
    mutable std::atomic <int> cachedValue;
};
```

Но здесь может произойти даже замедление! Представим, что много потоков выполняют эту функцию, но у всех *cacheValid == false*, потому что ещё никто не выполнил вычисления. Тогда все потоки будут вычислять бесполезные значения и никакого ускорения мы не добьёмся! Может прийти идея, что нужно поменять местами строчки сложения и установки индикатора, но это критически плохая идея! Ведь если один поток поставил флаг, но ещё не успел записать значения, а другой проверил флаг и вернул ещё не вычисленные значение, то работа программы некорректна!

Поэтому здесь лучше использовать мьютекс:
```cpp
class Widget {
public:
    int magicValue const
    {
        std::lock_guard <std::mutex> guard (mutex);    // Ставим блокировку
        if (cacheValid)
            return cachedValue;
        else {
            auto value1 = expensiveComputation1 ();
            auto value2 = expensiveComputation2 ();

            cachedValue = value1 + value2;    // Сложение
            cacheValide = true;               // Установка индикатора

            return cachedValue;                        // Разблокирование mutex
        }
    }

private:
    mutable std::mutex mutex;
    mutable bool cacheValid {false};
    mutable int cachedValue;
};
```

Это наводит на мысль, что если у вас есть одна переменная, требующая синхронизации, то адекватно применять *std::atomic*. Если их две и более, то нужен *std::mutex*.

**Важно помнить, что *std::atomic* и *std::mutex*невозможно копировать и перемещать, так что наличие этих элементов в классе запрещает этот класс копировать и перемещать!**

Если ваши методы-члены класса не будут использоваться в многопоточной реализации, то можно не усложнять свои классы, так как это ресурсоёмко.

### <center></center>
* Делайте константные функции-члены безопасными с точки зрения потоков, если только вы не можете быть уверены, что они *гарантированно* не будут использоваться в контексте параллельных вычислений.
* Использование переменных *std::atomic* может обеспечить более высокую по сравнению с мьютексами производительность, но они годятся только для работы с единственной переменной или ячейкой памяти.

## 3.11 Генерация специальных функций-членов

*Специальные функции-члены* - это те функции-члены, которые С++ готов генерировать сам.

В С++ 98 это:
* Конструктор по умолчанию
* Деструктор
* Копирующий конструктор
* Оператор копирующего присваивания

С++11 добавляет ещё:
* Перемещающий конструктор
* Оператор перемещающего присваивания
```cpp
class Widget {
public:
    Widget (Widget&& rhs);            // Перемещающий конструктор
    Widget& operator= (Widget&& rhs); // Оператор перемещающего присваивания
};
```

Операции копирования и перемещения копируют и перемещают только нестатические члены-данных

Деструктор производного класса является виртуальным, если этот класс унаследован от базового класса с виртуальным деструктором.

Сердцем каждого почленного перемещения является применение *std::move* к объекту.

Две операции *копирования* независимы: если вы определите одну, то вторая будет создана по стандартным правилам.
Две операции *перемещения* зависимы: если вы определили одну, то другая не будет создана.

Также, если есть копирующие операции в классе, то не будут генерироваться перемещающие и наоборот. Это объясняется тем, что компилятор считает, что раз копирование является не просто почленным копированием, то и перемещение будет особым, поэтому не будет создавать перемещение, потому что не знает как это сделать правильно.

**Рекомендация о *большой тройке***: если вы объявили хотя бы одну из трёх операций - копирующий конструктор, копирующий оператор присваивания или деструктор, - то вы должны объявить все три операции.

Перемещающие операции генерируются (при необходимости) только если в классе:
* не объявлены никакие копирующие операции
* не объявлены никакие перемещающие операции
* не объявлен деструктор

Если стандартное поведение определённых операций нас устраивает, то можно использовать *= default*.

```cpp
class Base {
public:
    virtual ~Base () = default;                // Делает конструктор виртуальным

    Base (Base&&) = default;                   // Поддержка перемещения
    Base& operator= (Base&&) = default;

    Base (const Base&) = default;              // Поддержка копирования
    Base& operator= (const Base&) = default;

    // ...
};
```

Фактически, даже если у нас есть класс, в котором компиляторы могут генерировать копирующие и перемещающие операции, *то лучше всего в них явно объявлять и применять конструкцию **default****. Это необходимо потому, что если мы захотим добавить, например, логирование в деструктор, т.е. определим нетривиальный деструктор, и забудем указать перемещающим операциям *= default*, то в классе не будут генерироваться перемещающие операции, потому что не выполняются соответствующие правила!

Пример:
```cpp
class StringTable {
public:
    StringTable () {}

private:
    std::map <int, std::string> values;
};
```

Добавим логирование:
```cpp
class StringTable {
public:
    StringTable () {
        makeLogEntry ("Создание StringTable");    // Добавлено
    }

    ~StringTable () {
        makeLogEntry ("Удаление StringTable");    // Добавлено, но удалились перемещающие операции!!!
    }

private:
    std::map <int, std::string> values;
};
```

Простое добавление деструктора может *на порядки замедлить систему*!

Правила С++11, управляющие специальными функциями-членами:
* *Конструктор по умолчанию*. Генерируется только если класс не содержит пользовательских конструкторов.
* *Деструктор*. Аналогично С++98, только ещё автоматически объявляется *noexcept*. Является виртуальным, если виртуальным является деструктор базового класса.
* *Копирующий конструктор*.Генерируется, только если класс не содержит пользовательского копирующего конструктора. Удаляется, если класс объявляет перемещающую операцию.
* *Оператор копирующего присваивания*. Генерируется, только если класс не содержит пользовательского оператора копирующего присваивания. Удаляется, если класс объявляет перемещающую операцию.
* *Перемещающий конструктор* и *оператор перемещающего присваивания*. Генерируется, только если класс не содержит пользовательских копирующих операций, пользовательских перемещающих операций или пользовательский деструктор.

Обратите внимание, что наличие *шаблона* функции-члена, не препятствует компиляторам генерировать специальные функции-члены, если они удовлетворяют обычным правилам:
```cpp
class Widget {
public:
    template <typename T>
    Widget (const T& rhs);

    template <typename T>
    Widget& operator= (const T& rhs);
};
```

В этом примере всё равно будут генерироваться операции копирования и перемещения. (В разделе 5.4 это имеет важные последствия)

### <center>Следует запомнить</center>
* *Специальыне функции-члены* - это те функции-члены, которые компиляторы могут генерировать самостоятельно: конструктор по умолчанию, деструктор, копирующие и перемещающие операции.
* Перемещающие операции генерируется только для классов, в которых нет явно объявленных перемещающих операций, копирующих операций и деструкторов.
* Копирующий конструктор генерируется только для классов, в которых нет явно объявленного копирующего конструктора, и удаляется, если объявляется перемещающая операция. Копирующий оператор присваивания генерируется только для классов, в которых нет явно объявленного копирующего оператора присваивания, и удаляется, если объявляется перемещающая операция.Генерация копирующих операций в классах с явно объявленным деструктором является устаревшей операцией и может быть отменена в будущем.
* Шаблоны функций-членов не подавляют генерацию *специальных функций-членов*.













