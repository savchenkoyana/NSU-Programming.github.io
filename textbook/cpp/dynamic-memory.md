---
title: Динамическое выделение памяти
menu: textbook-cpp
raiilink: https://ru.wikipedia.org/wiki/%D0%9F%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5_%D1%80%D0%B5%D1%81%D1%83%D1%80%D1%81%D0%B0_%D0%B5%D1%81%D1%82%D1%8C_%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F
---

## Стек и куча

Программе на C++ доступны два вида памяти: стек и куча (heap). Автоматические переменные вроде

```cpp
int a;
double b;
```

хранятся на стеке. Как следует из названия, стек работает с переменными по схеме FILO (first in last out). Управление стеком происходит автоматически. При выходе переменной из области видимости, соответствующая ей в стеке память освобождается. Этот механизм позволяет разработчику не следить за удалением автоматических переменных. Стек работает очень быстро, но имеет ограниченный размер, который обычно не превосходит нескольких мегабайт.

Второй тип памяти - куча - устроен иначе. В куче объекты можно хранить в произвольном месте, создавать и удалять их в произвольном порядке, а размер кучи обычно значительно превосходит размер стека. Платить за эти преимущества приходится скоростью: работа с кучей происходит значительно медленнее, чем со стеком. Кроме того, объекты из кучи не удаляются автоматически.

Кучу имеет смысл использовать в двух случаях:

* Необходимо хранить большой объект. Хранение больших объектов на стеке может привести к его переполнению (stack overflow).
* Автоматическое управление памятью в стеке не соответствует логике программы. Чаще всего такая ситуация возникает, когда созданный объект должен продолжать свое существование после выхода из блока, в котором он был создан. Ниже мы рассмотрим пример.

Динамическое выделение памяти означает работу с кучей и является предметом данного раздела.

## Ручное управление памятью

Начнем с обзора низкоуровневых инструментов, которые обычно не используются при разработке на современном C++. Знание эти инструментов, однако, может пригодиться при чтении старого кода и при работе со старыми компиляторами.

Создать объект в куче можно с помощью оператора `new`:

```cpp
int *intptr = new int(7);
auto *vecptr = new std::vector<std::string>();
```

Оператор `new` возвращает указатель на область памяти в куче, в которой был создан объект. Создав объект с помощью оператора `new`, разработчик становится ответственными за его удаление. Освободить выделенную память и удалить объект можно с помощью оператора `delete`:

```cpp
delete intptr;
delete vecptr;
```

Вернемся к примеру из раздела про наследование, в котором мы строили модель символов в графическом текстовом редакторе. Напомним, что мы создали абстрактный базовый класс `Character` и два его наследника `Letter` и `Digit`. Допустим, нам надо реализовать функцию, которая возвращает полиморфный список символов (текст документа). Без динамического выделения нам будет сложно решить эту задачу. Например:

```cpp
std::list<Character*> create_document() {
    // Тут есть проблема
    Letter l1('a');
    Letter l2('b');
    Digit d1('1');
    Digit d2('2');
    return std::list<Character*>{&l1, &l2, &d1, &d2};
}
```

Объекты `l1`, `l2`, `d1` и `d2` в функции `create_document` созданы на стеке. При выходе из функции `create_document` для каждого объекта будет вызван деструктор и освобождена память на стеке. В этом виде функция возвращает список указателей на освобожденную память, что приводит к неопределенному поведению. Следующее изменение сделает код корректным:

```cpp
std::list<Character*> create_document() {
    auto* l1 = new Letter('a');
    auto* l2 = new Letter('b');
    auto* d1 = new Digit('1');
    auto* d2 = new Digit('2');
    return std::list<Character*>{l1, l2, d1, d2};
}
```

Теперь память для объектов выделяется в куче, объекты продолжают существовать после выхода из функции. Использовать функцию `create_document` необходимо с учетом особенностей работы с динамической памятью. Напишем функцию `print_document`, которая вызывает `create_document` и выводит символы в стандартный поток вывода:

```cpp
// здесь есть проблема
void print_document() {
    auto doc = create_document();
    for (auto item : doc) {
        cout << item;
    }
}
```

Использование функции `print_document` приводит к утечке памяти: при каждом ее вызове в куче выделяется память, которая никогда не освобождается. Более аккуратная реализация выглядит так:

```cpp
void print_document() {
    auto doc = create_document();
    for (auto item : doc) {
        cout << item;
    }
    // здесь может быть проблема, которую мы обсудим позже
    for (auto item : doc) {
        delete item;
    }
}
```

Эта логика применима в любой ситуации с динамическим выделением памяти. Например, динамическое выделение памяти может происходить в конструкторе класса, а ее освобождение - в деструкторе.

Существуют версии операторов `new` и `delete` для создания и удаления массивов объектов:

```cpp
int* p = new int[10];  // выделяем массив из 10 переменных типа int
delete[] p;
```

При освобождении памяти важно использовать правильную версию оператора `delete`, что дополнительно усложняет разработку программ с ручным управлением памятью. Хорошей новостью является то, что оператор `delete[]` сам определяет размер удаляемого массива.

Далее мы рассмотрим более удобные и безопасные инструменты для работы с динамической памятью, которые доступны в современном C++.

## Владение ресурсами и идиома RAII

Динамическое выделение памяти тесно связано с концепцией владения ресурсами. Ресурсом может быть не только память, но и, например, файловый дескриптор или сокет-соединение. В хорошо спроектированной программе структура владения ресурсами устроена ясно: каждым ресурсом владеет определенный объект, который отвечает за освобождение ресурса. Владение ресурсом можно передавать другому объекту, который вместе с ресурсом берет на себя ответственность за его освобождение.

Ясно организовать владение ресурсами практически в любой программе можно, следую идиоме [RAII]({{ page.raiilink }}) (resource acquisition is initialization, получение ресурса есть инициализация), которая (в несколько упрощенном виде) состоит в следующем:

* Каждый ресурс следует инкапсулировать в класс, при этом
  * Конструктор выполняет выделение ресурса
  * Деструктор выполняет освобождение ресурса
* Взаимодействие с ресурсом происходит только через объект RAII-класса.

Мы уже видели пример RAII-объекта в C++, когда говорили про работу с файлами. Объект `fstream` владеет ресурсом - файловым дескриптором - и отвечает за его освобождение, а вся работа с файлом происходит через этот объект.

## Умные указатели

В рамках идиомы RAII в современном C++ решены сложности работы с динамическим выделением памяти. Логика работы с динамической памятью инкапсулирована в специальных классах `std::unique_ptr` и `std::shared_ptr`, которые называют умными указателями. При конструировании такого объекта происходит выделение памяти, а при вызове деструктора - освобождение. Например:

```cpp
#include <memory>

int main() {
    auto luptr = std::make_unique<Letter>('l');
    auto dsptr = std::make_shared<Digit>('7');
    return 0;
}
```

При выходе из функции `main` выделенная в куче память корректно будет освобождена. Объекты `std::unique_ptr` и `std::shared_ptr` различаются с точки зрения владения объектом. Уникальный указатель `std::unique_ptr` единолично владеет ресурсом. Это означает, что не может быть два разных объекта `std::unique_ptr` не могут быть связаны с одним и тем же ресурсом. Это, например, означает, что объект `std::unique_ptr` не имеет копирующего конструктора и копирующего оператора присваивания. Вместо этого возможно использование перемещающего конструктор и перемещающего оператора присваивания. Например:

```cpp
auto luptr = std::make_unique<Letter>('l');
// std::unique_ptr<Letter> luptr2 = luptr;  // ошибка, уникальное владение
auto luptr3 = std::move(luptr);  // перемещение возможно. luptr передал владение и потерял связь с объектом
```

Объекты `std::shared_ptr` можно копировать. При этом несколько объектов `std::shared_ptr` оказываются связанными с одним ресурсом (динамически выделенной памятью). Освобождение памяти происходит в момент, когда последний ссылающийся на эту память объект `std::shared_ptr` вышел из области видимости. Необходимость подсчета ссылок в объектах `std::shared_ptr` приводит к определенным накладным расходам. Например объекты `std::shared_ptr` занимают больше памяти, чем объекты `std::unique_ptr`. Объекты `std::unique_ptr` при этом не уступают в производительности простым указателям.

Важно, что умные указатели сохраняют свойство полиморфности. Это позволяет нам модифицировать функцию `create_document` следующим образом:

```cpp
std::list<std::unique_ptr<Character>> create_document() {
    std::list<std::unique_ptr<Character>> doc;
    doc.push_back(std::make_unique<Letter>('a'));
    doc.push_back(std::make_unique<Letter>('b'));
    doc.push_back(std::make_unique<Digit>('1'));
    doc.push_back(std::make_unique<Digit>('2'));
    return doc;
}
```

и не заботится больше о ручном освобождении ресурсов. Несмотря на некоторую громоздкость синтаксиса умные указатели значительно упрощают разработку на C++. Мы рекомендуем использовать умные указатели вместо низкоуровневых операторов `new` и `delete` для работы с динамической памятью.

Сложность обращения с длинными названиями типов в C++ вроде `std::list<std::unique_ptr<Character>>` (и это не самый плохой случай) может быть преодолена с помощью псевдонимов. Например:

```cpp
using CharPtr = std::unique_ptr<Character>;  // определили псевдоним для уникальных указателей на Character
using Document = std::list<CharPtr>;
Document create_document() {
    Document doc;
    doc.push_back(std::make_unique<Letter>('a'));
    doc.push_back(std::make_unique<Letter>('b'));
    doc.push_back(std::make_unique<Digit>('1'));
    doc.push_back(std::make_unique<Digit>('2'));
    return doc;
}
```

## Виртуальный деструктор

В заключение этого раздела обсудим один тонкий момент, связанный с полиморфизмом и освобождением ресурсов в C++. Функция `create_document` корректно работает с динамической памятью. Однако, если использовать классы `Character`, `Letter` и `Digit` в том виде, в каком мы их оставили в [разделе про наследование](inheritance), то освобождение памяти при удалении объекта `Document` будет выполнено неверно. Контейнер `std::list` работает с (умными) указателями на объекты абстрактного класса `Character`. При удалении объекта `std::list` происходит удаление всех объектов типа `std::unique_ptr<Character>`, которые в свою очередь вызывают деструкторы объектов `Character`. Вместо этого мы хотим, чтобы для каждого объекта вызывался деструктор нужного класса-наследника. Вызов только деструктора базового класса снова может привести к утечке памяти.

Как для любого другого метода полиморфизм вызова деструктора реализуется через механизм виртуальных методов, в данном случае нам нужен виртуальный деструктор:

```cpp
class Character {
    // ...
    virtual ~Character() = default;
};
```

Поскольку ничего особенного, кроме собственно виртуальности, нам не нужно, мы доверили генерирование деструктора компилятору. Теперь наша программа, динамически выделяющая память для полиморфных объектов с помощью умных указателей, будет работать так, как нам нужно.

В любой иерархии классов деструктор базового класса рекомендуется объявлять виртуальным, чтобы избегать проблем с освобождением ресурсов объектами классов-наследников.

## Резюме

В этом разделе мы обсудили основы работы с динамической памятью в C++. Рекомендуемыми инструментами работы с динамической памятью являются умные указатели `std::unique_ptr` и `std::shared_ptr`. Не забывайте объявлять деструктор базового класса виртуальным, если возможна работа с объектами классов-потомков через указатель на объект базового класса (а такая возможность есть всегда).

## Документация и ссылки

* [https://en.cppreference.com/w/cpp/memory/new/operator_new](https://en.cppreference.com/w/cpp/memory/new/operator_new)
* [https://en.cppreference.com/w/cpp/memory/new/operator_delete](https://en.cppreference.com/w/cpp/memory/new/operator_delete)
* [https://en.cppreference.com/w/cpp/language/raii](https://en.cppreference.com/w/cpp/language/raii)
* [Идиома RAII (wikipedia)](https://ru.wikipedia.org/wiki/%D0%9F%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5_%D1%80%D0%B5%D1%81%D1%83%D1%80%D1%81%D0%B0_%D0%B5%D1%81%D1%82%D1%8C_%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F)
* [https://en.cppreference.com/w/cpp/keyword/using](https://en.cppreference.com/w/cpp/keyword/using)
* [https://en.cppreference.com/w/cpp/language/destructor](https://en.cppreference.com/w/cpp/language/destructor)
