6\. Лямба-выражения (no end)
===
```cpp
std::find (containter.begin (), container.end (),
           [] (int val) { return 0 < val && val < 10; });    // Лямбда-выражеине
```
* *Замыкание* (*closure*) представляет собой объект времени выполнения, создаваемый лямба-выражением.
* *Класс замыкания* (*closure class*) представляет собой класс, из которого инстанцируется замыкание.

```cpp
{
    int x = 0;
    
    // ...
    
    auto c1 = [x] (int y) {return x * y > 55; }; // Копия замыкания
    auto c2 = c1;                                // Копия замыкания
    auto c3 = c2;                                // Копия замыкания
}
```

## 6.1 Избегайте режимов захвата по умолчанию

Захват по умолчанию - это захват по ссылке. Нужно быть осторожным и писать лямбды так, чтобы ссылки не провисали. Вместо *[&]*, лучше писать явно название захватываемой переменной: *[&divisor]*.

Пример (не может провиснуть, но опасно):
```cpp
template <typename C>
void workWithContainer (const C& cont)
{
    auto calc1 = computeValue1 ();
    auto calc2 = computeValue2 ();
    auto divisor = computeDivisor (calc1, calc2);
    
    using ContElemT = C::value_type_t; // typenmae C::value_type
    
    if (std::all_of (
            std::begin (cont),
            std::end (cont),
            [&divisor] (const auto& value)
            { return value % divisor == 0; }        
        ))
    {
        // Да
    } else {
        // Как минимум одно - нет
    }
};
```

Можно применить захват по значению и для данного примера этого будет достаточно:
```cpp
using FilterContainer = std::vector <std::function <bool (int)>>;

FilterContainer filters;

// ...

filters.emplace_back (                // Теперь divisor
    [=] (int value)                   // не может 
    { return value % divisor == 0; }  // повиснуть
);
```

Но что если у нас есть следующий класс:
```cpp
class Widget {
public:
    void addFilter () const;

private:
    int divisor;
}; 

void Widget::addFilter () const
{
    filters.emplace_back (
        [=] (int value) { return value % divisor == 0; }
    );
}
```

Это сверх плохой код! Если мы заменим *[=]* на *[]* или *[divisor]*, то компилироваться вообще не будет. Дело в том, что компилятор видит предыдущий код как: 
```cpp
class Widget {
public:
    void addFilter () const;

private:
    int divisor;
}; 

void Widget::addFilter () const
{
    auto currentObjectPtr = this;
    filters.emplace_back (
        [currentObjectPtr] (int value)
        { return value % currentObjectPtr->divisor == 0; }
    );
}
```

Тогда мы с лёгкостью можем получить висячий указатель!
```cpp
void doSomeWork {
    auto pw = std::make_unique <Widget> ();
    
    pw->addFilter ();
} // Уничтожение Widget; В filter висячий указатель!
```

Чтобы решить эту проблему, нужно вно создать локальную копию члена-данных:
```cpp
void Widget::addFilter () const
{
    auto divisorCopy = divisor;
    filters.emplace_back (
        [divisorCopy] (int value)            // Просто [=] тоже будет работать
        { return value % divisorCopy == 0; }
    );
}
```

Но зачем испытывать удачу, если в С++14 имеется *лучший способ захвата члена-данных*, заключающийся в использовании обобщённого захвата лямбда-выражения (см. разде 6.2):
```cpp
void Widget::addFilter () const
{
    filters.emplace_back (
        [divisor = divisor] (int value)    // Копирование divisor в замыкание
        { return value % divisor == 0; }
    );
}
```

Режим захвата по умолчанию для обобщённого захвата лямбда-выражения не существует даже в С++14, так что *нужно избегать режимов захвата по умолчанию*.

Рассмотри наш пример, но уже со статическими объектами хранения:
```cpp
template <typename C>
void workWithContainer (const C& cont)
{
    static auto calc1 = computeValue1 ();
    static auto calc2 = computeValue2 ();
    static auto divisor = computeDivisor (calc1, calc2);
    
    filters.emplace_back (
        [=] (int value)                    // Ничего не захватывает
        { return value % divisor == 0; }   // Ссылка на статическую переменную
    );
    
    ++divisor;
};
```

Здесь хоть и написано *[=]*, но ни о каком копировании речи нет. Дело в том, что лямбда-выражения не использует никакие нестатические локальные переменные, поэтому ничего не захватывается. Поэтому после каждого вызова этой функции, они будут демонстрировать разное поведение.

### <center>Следует запомнить</center>
* Захват по умолчанию по ссылке может привести к висячим ссылкам.
* Захват по умолчанию по значению восприимчив к висячим указателям (особенно к *this*) и приводит к ошибочному предположению о самодостаточности лямбда-выражений.

## 6.2 Используйте инициализирующий захват для перемещения объектов в замыкания

Чтобы перемещать объекты в замыкание нужно использовать специальный механизм. Очень гибкий механизм, часть которого позволяет перемещать объекты в замыкание, называется *инициализирующий захват*.

Применение *инициализирующего захвата* делает возможным узнать:
1. *Имя члена данных* в классе замыкания, сгенерированном из лямбда-выражения
2. *Выражение инициализации* этого члена-данных

Вот так можно передать, например, *std::unique_ptr*:
```cpp
class Widget {
public:
    bool isValidated () const;
    bool isProcessed () const;
    bool isArchived  () const;
private:
    // ...
};

auto pw = std::make_unique <Widget> ();

// Настройка *Widget

auto func = [pw = std::move (pw)]         // Инициализирующий захват
    {  
        return pw->isValidated () &&
               pw->isArchived ();
    };
``` 

Если нам не нужен пункт "Настройка *Widget", то можем непосредственно инициализировать член-данные класса замыкания (С++14):
```cpp
auto func = [pw = std::make_unique <Widget> ()]         // Инициализирующий захват
    {  
        return pw->isValidated () &&
               pw->isArchived ();
    };
```

Ещё одним названием *инициализирующего захвата* является *обобщённый захват лямбда-выражения* (*generalized lambda capture*).

Также нужно понимать, что всё, что мы можем сделать с классами, мы можем сделать и с лямбда-выражениями. Поэтому если у вас С++11 или компилятор не поддерживает инициализирующий захват, то можно написать всё это своими руками:
```cpp
class IsValAndArch {
public:
    using DataType = std::unique_ptr <Widget>;
    
    explicit IsValAndArch (DataType&& ptr) :
        pw (std::move (ptr))
    {}
    
    bool operator () () const
    {
        return pw->isValidated () &&
               pw->isArchived ();
    }
    
private:
    DataType pw;
};
```

В С++11 перемещающий захват можно эмулировать с помощью:
1. Перемещения захватываемого объекта в функциональный объект с помощью *std::bind*
2. Передачи лямбда выражению ссылки на захватываемый объект













































