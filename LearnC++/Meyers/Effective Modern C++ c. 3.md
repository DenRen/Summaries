Переход к современному C++
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

### 3.5 Предпочитайте удалённые (deleted) функции закрытым неопределённым

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
void processPointer <void> (T* ptr);

template <>
void processPointer <const void> (T* ptr);

template <>
void processPointer <volatile void> (T* ptr);
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

### 3.6 Объявляйте перекрывающие функции как *override*

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


### 3.7 Предпочитайте итераторы *const_iterator* итераторам *iterator*

 


















