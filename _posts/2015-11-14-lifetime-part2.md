---
title: "Время жизни в Rust (Часть 2)"
categories: обучение
published: true
author: Александр Яшкин
excerpt:
    В первой части мы рассмотрели простой пример работы времени жизни в Rust. В этой
    части мы рассмотрим пример с более сложным использованием времени жизни.
    Попробуем в экземпляре структуры хранить ссылку на экземпляр другой структуры.

---

# Введение

В первой части мы рассмотрели простой пример работы времени жизни в Rust. В этой
части мы рассмотрим пример с более сложным использованием времени жизни.
Попробуем в экземпляре структуры хранить ссылку на экземпляр другой структуры.

# Постановка задачи

Мы создадим структуру `Customer`, описывающую покупателя, который должен владеть
экземпляром структуры `Car`. Покупатель будет иметь возможность покупать,
продавать и обмениваться с другими покупателями автомобилями.

# Пишем код

Тип `Car` будет лишь включать в себя название модели:

```rust
struct Car {
    model: String
}
```

При описании структуры `Customer` возникает вопрос о хранении в себе экземпляра
структуры `Car`. Для начала попробуем сделать так:

```rust
struct Customer {
    car: Option<Car>
}
```

Этот вариант самый простой. Но здесь есть большая проблема. Автомобиль не
является частью клиента. Покупка и продажа будет создавать копии автомобилей,
что приводит к большему увеличению памяти и  затраты процессора на копирование.
Хотелось бы хранить не экземпляр структуры `Car`, а ссылку на экземпляр. Вот тут
всё становится гораздо сложнее.

# Ссылка в структуре

В Rust ссылка должна всегда указывать на выделенный участок памяти, чтобы
избежать проблем. Это означает, что время жизни экземпляра структуры должно быть
не меньше чем у ссылки, указывающей на переменную внутри этого экземпляра. Это
означает, что ссылка, хранящаяся внутри структуры `Customer`, требует явного
указания параметров времени жизни. Добавим параметры времени жизни:

```rust
struct Customer<'a> {
    car: Option<&'a Car>
}
```

Здесь в **`Customer<'a>`** мы задаём имя `a` для времени жизни.

В **`Option<&'a Car>`** указываем, что экземпляр структуры `Car` имеет время
жизни `a`.

Теперь компилятор будет знать, что у экземпляра структуры `Customer` время жизни
должно быть таким же как и у экземпляра структуры `Car` или меньше. В противном
случае мы будем ссылаться на освобождённый участок памяти.

# Методы

После того, как мы объявили структуры, можем приступить к написанию методов для
структуры `Customer`:

```rust
impl <'a> Customer<'a> {
    fn new() -> Customer<'a> {
        Customer {
            car: None
        }
    }

    fn buy_car(&mut self, c: &'a Car) {
        self.car = Some(c);
    }

    fn sell_car(&mut self) {
        self.car = None;
    }
}
```

В этом коде мы реализовали покупку и продажу автомобиля. Обмен между
покупателями мы напишем чуть позже в этой статье.

Обратите внимание на объявление `impl <'a> Customer<'a>`. Как вы могли заметить,
объявления параметров времени жизни очень похоже на [обобщённое
программирование](https://rurust.gitbooks.io/rust_book_ru/content/src/generics.html).
В этом объявлении после ключевого слова `impl` задаём имена для параметров
времён жизни. Их может быть несколько. В нашем примере используется один
параметр времени жизни `a`.

# Использование

Отлично! У нас есть две структуры `Customer` и `Car`. Давайте попробуем
использовать эти структуры в программе:

```rust
fn main() {
    let car = Car{ model: "Skoda Fabia".to_string() };
    let mut bob = Customer::new();

    // Боб покупает машину
    bob.buy_car(&car);
}
```

Этот код рабочий и компилируется. Но если мы поменяем две строчки местами, то
код перестанет компилироваться:

```rust
fn main() {
    let mut bob = Customer::new();
    let car = Car{ model: "Skoda Fabia".to_string() };

    bob.buy_car(&car); // Ошибка!
}
```

Причина ошибки в том, что время жизни у `bob` (экземпляр структуры `Customer`)
больше чем у `car` (экземпляр структуры `Car`). В Rust выделение и освобождение
объектов происходит в порядке, обратном порядку объявления. Таким образом, на
непродолжительное время `bob` будет хранить в себе ссылку на уже освобождённый
`car`. Rust не может нам такое позволить.

Система времени жизни в Rust не может знать, когда ссылка будет не нужна. Давайте
посмотрим на ещё один пример:

```rust
fn main() {
    let logan = Car{ model: "Renault Logan".to_string() };
    let mut bob = Customer::new();

    {
        // Внутренняя область видимости

        let fabia = Car{ model: "Skoda Fabia".to_string() };

        bob.buy_car(&fabia);    // ОШИБКА!
        bob.buy_car(&logan);
    }
}
```

Этот код не компилируется, т.к. время жизни ссылки `fabia` меньше, чем у `bob`.
С точки зрения разработчика код безопасен, т.к. после внутренней области
видимости `bob` больше не ссылается на `fabia`. Компилятор строго следует нашему
объявлению метода `buy_car(&mut self, c: &'a Car)`, где мы указали, что
экземпляр структуры `Car` имеет время жизни `a`, и экземпляр `Customer` не может
иметь время жизни больше, чем у экземпляра `Car`.

# Торговля между покупателями

Торговля в нашей задаче подразумевает обмен машинами между экземплярами структур
`Customer`. Добавим метод для `Customer`:

```rust
fn trade_with(&mut self, other: &mut Customer<'a>) {
    let tmp = other.car;
    
    other.car = self.car;
    self.car = tmp;
}
```

Здесь нет ничего особенного. Хочется лишь отметить, что параметр времени жизни
является частью типа данных. Переменная `other` имеет тип данных `&mut
Customer<'a>`.

Теперь попробуем воспользоваться этим методом:

```rust
fn main() {
    let fabia = Car{ model: "Skoda Fabia".to_string(); };
    let logan = Car{ model: "Renault Logan".to_string(); };

    let mut bob = Customer::new();
    let mut alice = Customer::new();

    bob.buy_car(&fabia);
    alice.buy_car(&logan);

    bob.trade_with(&mut alice);
}
```

Экземпляры структуры `Car` требуется создавать перед созданием экземпляров
структур `Customer`. Но не имеет значение очерёдность объявлений `bob` и
`alice`.

# Применение правил заимствования

Когда экземпляр хранит в себе ссылку на другой экземпляр, то он _заимствует_
ссылку и применяются правила заимствования (мы перечисляли их в первой части
статьи).

Можно **позаимствовать множество неизменяемых ссылок** на экземпляр, если
не создаются ссылки с правом изменения.

```rust
fn main() {
    let fabia = Car{ model: "Skoda Fabia".to_string() };
    let mut bob = Customer::new();

    bob.buy_car(&fabia);    // bob заимствует неизменяемую ссылку на fabia 

    let p1 = &fabia;        // Можем заимствовать дополнительные неизменяемые ссылки
    let p2 = &fabia;
}
```

Можно иметь лишь **одно заимствование изменяемой ссылки** на экземпляр при, этом
на этот экземпляр не должно быть каких-либо других заимствований.

```rust
fn main() {
    let mut fabia = Car{ model: "Skoda Fabia".to_string() };
    let mut bob = Customer::new();

    bob.buy_car(&fabia);    // bob заимствует изменяемую ссылку на fabia

    let p1 = &mut fabia;    // Ошибка!
}
```

Вы не можете **передать владение** экземпляром пока существует заимствование на
данный экземпляр.

```rust
fn main() {
    let mut fabia = Car{ model: "Skoda Fabia".to_string() };
    let mut bob = Customer::new();

    bob.buy_car(&fabia);    // bob заимствует ссылку на fabia

    let f = fabia;          // Ошибка! Нельзя передать владение.
}
```

Этот код, неожиданно для программиста, не будет компилироваться:

```rust
fn main() {
    let fabia = Car{ model: "Skoda Fabia".to_string() };
    let mut logan = Car{ model: "Renault Logan".to_string() };
    let mut bob = Customer::new();

    bob.buy_car(&fabia);
    bob.buy_car(&logan);

    let p1 = &mut fabia;    // Ошибка!
}
```

С точки зрения программиста код является безопасным. `bob` больше не нуждается в
заимствовании ссылки на `fabia` и заимствует ссылку на `logan`, но компилятор не
может догадаться об этом.

# Заключение

Отлично! Мы рассмотрели модель хранения ссылок в структурах. Плохо то, что не
всегда эта модель работает. Добавим функцию в наш код:

```rust
fn shop_for_car(c: &mut Customer) {
    let car = Car{ model: "BMW M4 Coupe".to_string() };

    c.buy_car(&car);    // Ошибка!
}
```

Здесь ошибка заключается в том, что созданный экземпляр структуры `Car` будет
освобождён по завершению локальной области видимости функции. Здесь требуется
выделить память в куче (heap) и разместить в ней экземпляр `car`. В Rust это
можно сделать с помощью контейнера
[`Box<T>`](https://doc.rust-lang.org/stable/std/boxed/). Но об этом мы поговорим
в третьей части серии статей о времени жизни в Rust.

