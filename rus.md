[Недавно][1], TC39 определили финальную семантику классов в [ECMAScript 6][2].
Это статья поясняет как работает их реализация. Наиболее значимые из недавних
изменений связаны с тем, как реализована система наследования классов.

### 1. Обзор

    class Point {
        constructor(x, y) {
            this.x = x;
            this.y = y;
        }
        toString() {
            return '(' + this.x + ', ' + this.y + ')';
        }
    }

    class ColorPoint extends Point {
        constructor(x, y, color) {
            super(x, y);
            this.color = color;
        }
        toString() {
            return super.toString() + ' in ' + this.color;
        }
    }
    
    let cp = new ColorPoint(25, 8, 'green');
    cp.toString(); // '(25, 8) in green'
    
    console.log(cp instanceof ColorPoint); // true
    console.log(cp instanceof Point); // true
    

### 2. Основы

#### 2.1 Базовые классы

Классы определяются в ECMAScript 6 (ES6) следующим образом:

    class Point {
        constructor(x, y) {
            this.x = x;
            this.y = y;
        }
        toString() {
            return '(' + this.x + ', ' + this.y + ')';
        }
    }
    
Использовать этот класс можно просто вызвав конструктор функции, как в ES5:

    > var p = new Point(25, 8);
    > p.toString()
    '(25, 8)'
    
По факту, результатом создания такого класса будет функция:

    > typeof Point
    'function'
    
Однако, вы можете вызывать класс только через `new`, а не через вызов
функции ([Секция 9.2.2][2] в спецификации):

    > Point()
    TypeError: Classes can’t be function-called
    

##### Объявления классов не поднимаются

Объявления функций _поднимаются_: объявленные внутри общей области видимости,
функции сразу же доступны, независимо от того, где они были объявлены. Это 
означает, что вы можете вызвать функцию, которая будет объявлена позднее.

    foo(); // works, because `foo` is hoisted
    
    function foo() {}
    
В отличие от функций, определения классов не _поднимаются_. Таким образом, 
класс существует только после того, как его определение было достигнуто
и выполнено. Попытка создания класса до этого момента приведет к 
_«ReferenceError»_:

    new Foo(); // ReferenceError

    class Foo {}
    
Причина этого ограничения в том, что классы могут быть наследниками. Это 
поддерживается с помощью выражения _«extends»_, значение которого может быть
произвольным. Это выражение должно быть установлено в определенном месте,
которое не может быть _поднято_.

Отсутствие механизма _поднятия_ — это не такое большое ограничение, как вы
могли бы подумать. Например, функция, которая определена до определения 
класса, все еще может ссылаться на этот класс, но вы вынуждены ждать
выполнения определения класса до того, как сможете выполнить эту функцию.

    function functionThatUsesBar() {
        new Bar();
    }

    functionThatUsesBar(); // ReferenceError
    class Bar {}
    functionThatUsesBar(); // OK


##### Выражения класса

Так же, как и для функций, есть два способа определить класс: _объявление
класса_ и _выражение класса_.

По аналогии с функциями, идентификатор выражения класса доступен только 
внутри выражения:

    const MyClass = class Me {
        getClassName() {
            return Me.name;
        }
    };
    let inst = new MyClass();
    console.log(inst.getClassName()); // Me
    console.log(Me.name); // ReferenceError: Me is not defined
    

#### 2.2 Внутри тела определения класса

Тело класса может содержать только методы, но не свойства. 
Прототип, имеющий свойства обычно считается анти-паттерном.

##### _«Сonstructor»_, статические методы, прототипные методы

Давайте рассмотрим три вида методов, которые вы часто можете встретить 
в классах.

    class Foo {
        constructor(prop) {
            this.prop = prop;
        }
        static staticMethod() {
            return 'classy';
        }
        prototypeMethod() {
            return 'prototypical';
        }
    }
    let foo = new Foo(123);
    
Диаграмма объекта для это определения класса выглядит следующим образом:

(Совет для понимания: `[[Prototype]]` — это отношения наследования между 
объектами, в то время как `prototype` — обычное свойство, значением которого 
является объект. Значение свойства `prototype` оператор `new` использует 
как прототип для создаваемых объектов.)

![Схема наследования][Схема наследования]

**Для начала рассмотрим псевдо-метод _«constructor»_.** Этот метод является 
особенным, так как он определяет функцию, которая представляет собой класс:

    > Foo === Foo.prototype.constructor
    true
    > typeof Foo
    'function'

Иногда его называют _конструктором класса_. Он имеет особенности, которые 
обычный конструктор функции не имеет (главным образом, способность 
конструтора вызывать конструктор базового класса через _`super()`_, о котором 
я расскажу чуть позже).

**Далее, статические методы.** _Статичесие свойства_ (или _свойства класса_)
являются свойствами самого _`Foo`_. Если вы определили метод с помощью _`static`_,
значит вы реализовали метод класса:

    > typeof Foo.staticMethod
    'function'
    > Foo.staticMethod()
    'classy'
    

**В третьих, прототипные методы.** _свойства прототипа_ _Foo_ являются и 
свойствами _Foo.prototype_. Это, как правило, методы, и наследуются 
экземплярами _Foo_.

    > typeof Foo.prototype.prototypeMethod
    'function'
    > foo.prototypeMethod()
    'prototypical'
    

##### Get'еры и Set'еры

Синтаксис для get'еров и set'еров такой же как и в 
[ECMAScript 5 литералы объекта][4]:

    class MyClass {
        get prop() {
            return 'getter';
        }
        set prop(value) {
            console.log('setter: '+value);
        }
    }

_MyClass_ используется следующим способом:

    > let inst = new MyClass();
    > inst.prop = 123;
    setter: 123
    > inst.prop
    'getter'
    

##### Вычисляемые имена методов

Вы можете определить имя метода с помощью выражения, если поместите его 
в квадратные скобки. Например, следующие определения класса _Foo_ 
эквивалентны:

    class Foo() {
        myMethod() {}
    }
    
    class Foo() {
        ['my'+'Method']() {}
    }
    
    const m = 'myMethod';
    class Foo() {
        [m]() {}
    }

Некоторые специальные методы в ECMAScript 6 имеют ключи, которые являются
символами [3].
Механизм вычисляемых имен методов позволяют вам определять такие методы. 
Например, если объект имеет метод с ключом _Symbol.iterator_, это — 
_итератор_ [4]. Это означает, что его содержимое может быть итерировано 
циклом _for-of_ или другими механизмами языка.

    class IterableClass {
        [Symbol.iterator]() {
            ···
        }
    }
    

##### Генераторы

Если вы определите метод с _*_ в начале, то получите _метод генератор_ [4].
Между прочим, генератор полезен для определения метода, ключом котороого 
является _Symbol.iterator_. Следующий код демонстрирует как это работает:

    class IterableArguments {
        constructor(...args) {
            this.args = args;
        }
        * [Symbol.iterator]() {
            for (let arg of this.args) {
                yield arg;
            }
        }
    }
    
    for (let x of new IterableArguments('hello', 'world')) {
        console.log(x);
    }
    
    // Output:
    // hello
    // world
    
#### 2.3 Классы наследники

Ключевое слово _extends_ позволяет создать класс-наследник существующего 
конструктора (который возможно был определен с помощью класса):

    class Point {
        constructor(x, y) {
            this.x = x;
            this.y = y;
        }
        toString() {
            return '(' + this.x + ', ' + this.y + ')';
        }
    }
    
    class ColorPoint extends Point {
        constructor(x, y, color) {
            super(x, y); // (A)
            this.color = color;
        }
        toString() {
            return super.toString() + ' in ' + this.color; // (B)
        }
    }
    
Этот класс используется как и ожидалось:

    > let cp = new ColorPoint(25, 8, 'green');
    > cp.toString()
    '(25, 8) in green'
    
    > cp instanceof ColorPoint
    true
    > cp instanceof Point
    true
    

В данном случае мы имеем два вида классов:

*   _Point_ — это _базовый класс_, потому что он не имеет выражения _extends_.
*   _ColorPoint_ — _производный класс_.

Есть два способа использовать ключевое слово _super_:

*   _Конструктор класса_ (псевдо-метод _constructor_ в теле класса)
    использует его как вызов функции (_super(···)_), для того чтобы  
    вызвать базовый конструктор (строка A).
*   Определения методов (в объектах, заданых через литерал, или классах,
    статических или нет) используют это для вызова свойства (_super.prop_)
    или вызова метода (_super.method(···)_), для ссылки на свойства базового
    класса (строка B).

##### Прототип класса наследника является базовым классом

Прототип класса наследника является базовым классом в ECMAScript 6:

    > Object.getPrototypeOf(ColorPoint) === Point
    true
    
Это означает, что статические свойства наследуются:

    class Foo {
        static classMethod() {
            return 'hello';
        }
    }
    
    class Bar extends Foo {
    }
    Bar.classMethod(); // 'hello'
    
Можно вызывать статические методы базового класса:

    class Foo {
        static classMethod() {
            return 'hello';
        }
    }
    
    class Bar extends Foo {
        static classMethod() {
            return super.classMethod() + ', too';
        }
    }
    Bar.classMethod(); // 'hello, too'
    

##### Вызов базового конструктора

В классе наследнике нужно вызвать _super()_ до того как будете обращаться к 
свойствам через _this_:

    class Foo {}
    
    class Bar extends Foo {
        constructor(num) {
            let tmp = num * 2; // OK
            this.num = num; // ReferenceError
            super();
            this.num = num; // OK
        }
    }

Пропустив вызов _super()_ в производном классе, вы получите ошибку:

    class Foo {}
    
    class Bar extends Foo {
        constructor() {
        }
    }
    
    let bar = new Bar(); // ReferenceError
    

##### Переопределение результата конструктора

Так же, как в ES5, можно переопределить результат конструктора, явно 
возвращая объект:

    class Foo {
        constructor() {
            return Object.create(null);
        }
    }
    console.log(new Foo() instanceof Foo); // false

Если вы так сделаете, то не имеет значения, инициализирован ли _this_ или нет.
Другими словами: вы не обязаны вызывать _super()_ в производном конструкторе, 
если переопределите результат таким образом.

##### Конструкторы по умолчанию для классов

Если не указать _constructor_  для базового класса, тогда используется
следующая конструкция:

    constructor() {}

Для дочерних классов, используется конструктор по умолчанию:

    constructor(...args) {
        super(...args);
    }

##### Наследования встроенных конструкторов

В ECMAScript 6 наконец-то можно наследоваться от всех встроенных 
конструкторов ([обходные пути в ES5][5], но здесь накладываются 
значительные ограничения).

Например, теперь вы можете создавать свои собственные классы исключений 
(которые будут наследовать такие особенности, как стек вызовов для 
большинства браузерных движков):

    class MyError extends Error {    
    }
    throw new MyError('Something happened!');

Вы также можете наследоваться от _Array_, экзмепляры которого правильно 
работают с _length_:

    class MyArray extends Array {
        constructor(len) {
            super(len);
        }
    }
    
    // Instances of of `MyArray` work like real arrays:
    let myArr = new MyArray(0);
    console.log(myArr.length); // 0
    myArr[0] = 'foo';
    console.log(myArr.length); // 1

Заметьте, что наследование от встроенных конструкторов — это то, что движок 
должен поддерживать изначально, вы не сможете получить эту функциональность 
с помощью транспайлеров.

### 3. Детали классов

То, что мы до сих пор рассматривали является основой классов. Если вам 
интересно узнать подробнее механизм классов, то вам нужно читать дальше. 
Давайте начнем с синтаксиса классов. Ниже приводится немного модифицированная 
верcия синтаксиса предложенного в [Секции A.4 специфакации ECMAScript 6][6].

    ClassDeclaration:
        "class" BindingIdentifier ClassTail
    ClassExpression:
        "class" BindingIdentifier? ClassTail
    
    ClassTail:
        ClassHeritage? "{" ClassBody? "}"
    ClassHeritage:
        "extends" AssignmentExpression
    ClassBody:
        ClassElement+
    ClassElement:
        MethodDefinition
        "static" MethodDefinition
        ";"
    
    MethodDefinition:
        PropName "(" FormalParams ")" "{" FuncBody "}"
        "*" PropName "(" FormalParams ")" "{" GeneratorBody "}"
        "get" PropName "(" ")" "{" FuncBody "}"
        "set" PropName "(" PropSetParams ")" "{" FuncBody "}"
    
    PropertyName:
        LiteralPropertyName
        ComputedPropertyName
    LiteralPropertyName:
        IdentifierName  /* foo */
        StringLiteral   /* "foo" */
        NumericLiteral  /* 123.45, 0xFF */
    ComputedPropertyName:
        "[" Expression "]"
    

Два наблюдения:

*   Значение расширения может быть произвольным выражением. Это значит, что 
    вы можете написать следующий код: 
    
        class Foo extends combine(MyMixin, MySuperClass) {}

*   Между методами допускается точка с запятой.

#### 3.1 Различные проверки

*   Проверки ошибок: имя класса не может быть _eval_ или _arguments_; 
    одинаковые имена классов не допускаются; название _constructor_ может 
    использоваться только для обычных методов, для get'еров, set'еров и 
    генераторов — не допускается.   

*   Классы не могут быть вызываемой функцией. Иначе они бросают исключение 
    _TypeException_

*   Методы прототипа не могут использоваться как конструкторы:
    
        class C {
            m() {}
        }
        new C.prototype.m(); // TypeError        

#### 3.2 Атрибуты свойств

Определения класса создают (изменяемые) разрешаемые связи. Для данного 
класса _Foo_:

*   Статические методы _Foo.*_ доступны для записи и настройки, но
    не для перечисления. Доступность для записи позволяет динамически вносить
    изменения в них.
   
*   Конструктор и объект в свойстве _prototype_ имеют неизменяемые ссылки:
   
    *   _Foo.prototype_ не доступен для записи, перечисления и настройки.
    *   _Foo.prototype.constructor_ не доступен для записи, перечисления 
    и настройки.
       
*   Прототипные методы _Foo.prototype.*_ доступны для записи и настройки, 
    но не для перечисления.
   
Заметьте, что определения методов в литералах объекта создают перечисляемые 
свойства.

### 4. Детали наследования классов

В ECMAScript 6, наследование классов выглядит следующим образом:

    class Point {
        constructor(x, y) {
            this.x = x;
            this.y = y;
        }
        ···
    }
    
    class ColorPoint extends Point {
        constructor(x, y, color) {
            super(x, y);
            this.color = color;
        }
        ···
    }
    
    let cp = new ColorPoint(25, 8, 'green');    

Этот код создает следующие объекты:

![][7] 

Следующий подраздел расматривает цепочки прототипов (в две колонки), 
далее рассматривает как _cp_ выделяется в памяти и инициализируется.

#### 4.1 Цепочки прототипов

На диаграмме видно что есть 2 _цепочки прототипов_ (объекты связаны через 
отношения _[[Prototype]]_, которые наследуются):

*   Левая колонка: классы (функции). Прототипом производного класса является 
    расширяемый класс. Прототип базового класса является _Function.prototype_, 
    которая также есть прототип функции:
    
        > const getProto = Object.getPrototypeOf.bind(Object);
        
        > getProto(Point) === Function.prototype
        true
        > getProto(function () {}) === Function.prototype
        true

*   Правая колонка: цепочки прототипов экземпляров. Цель класса — 
    установить эту цепочку прототипов. Цепочка протипов заканчивается на
    _Object.prototype_ (чей прототип является _null_), который также прототип 
    объектов, созданных через литералы объекта:

        > const getProto = Object.getPrototypeOf.bind(Object);
        
        > getProto(Point.prototype) === Object.prototype
        true
        > getProto({}) === Object.prototype
        true

Из цепочки прототипов в левой колонке следует, что статические свойства 
наследуются. 

#### 4.2 Выделение памяти и инициализация экземпляров объектов

Потоки данных между конструкторами классов отличается от канонического пути 
наследования в ES5. Под капотом это выглядит примерно так:

    // Instance is allocated here
    function Point(x, y) {
        // Performed before entering this constructor:
        this = Object.create(new.target.prototype);
    
        this.x = x;
        this.y = y;
    }
    ···
    
    function ColorPoint(x, y, color) {
        // Performed before entering this constructor:
        this = uninitialized;
    
        this = Reflect.construct(Point, [x, y], new.target); // (A)
            // super(x, y);
    
        this.color = color;
    }
    Object.setPrototypeOf(ColorPoint, Point);
    ···
    
    let cp = Reflect.construct( // (B)
             ColorPoint, [25, 8, 'green'],
             ColorPoint);
    // let cp = new ColorPoint(25, 8, 'green');
    
В ES5 и ES6 экземпляр объекта создается в разных местах:

*   В ES6 он создается базовым конструктором, последним в цепочке вызовов 
    конструкторов.
*   В ES5 он создается оператором _new_, первым в цепочке вызовов 
    конструкторов.
   
Предыдущий код использует две новые возможности ES6:

*   _new.target_ является неявным параметром, который имеют все функции. Это
    вызов конструктора, где _this_ является вызовом метода.
    
    *   Если конструктор напрямую вызывается через _new_, его значение это
        и есть этот конструктор (строка B).
    *   Если конструктор был вызван через _super()_, его значение это
        _new.target_ конструктора, который был вызван (строка A).
    *   Вызвав функцию обычным способом, значение будет _undefined_. Это 
        значит, что вы можете использовать _new.target_ чтобы определить была 
        ли функция функцией вызова или вызовом конструктора (через _new_).
    *   Внутри стрелочной функции _new.target_ ссылается на _new.target_
        окружающей не стрелочной функции.

*   _Reflect.construct()_ [5] позволяет вызвать конструктор при задании
    _new.target_ в качестве последнего параметра.

Преимуществом этой реализации наследования является то, что это позволяет 
писать нормальный код для наследования встроенных констукторов (такие как 
_Error_ и _Array_). Последний раздел объясняет, почему иной подход был 
необходим.

##### Проверки безопасности

*   _this_ инициализируется в производных конструкторах и значит, что если
    к _this_ обращаются до того как вызвали _super()_, то будет 
    брошено исключение.
*   Вызов _super()_ после инициализации _this_ приведет к _ReferenceError_.
    Это защита от двойного вызова _super()_.
*   Если конструктор возвращается неявно (без _return_), тогда результат будет
    _this_. Если _this_ инициализирован, тогда бросится исключение 
    _ReferenceError_ . Это защита от невызова _super()_.
*   Если конструктор явно возвращает не объект (включая _undefined_ и _null_),
    результатом будет _this_ (это поведение оставляет совместимость с ES5 
    и ранее). Если _this_ инициализирован, тогда бросится исключение 
    _TypeError_.
*   Если конструктор явно возвращает объект, тогда он и будет результатом.
    Тогда не имеет значение инициализирован _this_ или нет.

##### Выражение _extends_

Давайте рассмотрим как выражение _extends_ влияет на работу класса
([Секция. 14.5.14 спецификации][8]).

Значение _extends_ должно быть «конструктивно» (вызываться через _new_) 
хотя _null_ тоже поддерживается.

    class C {
    }

*   Тип конструктора: базовый
*   Прототип _C_: _Function.prototype_ (как обычная функция)
*   Прототип _C.prototype_: _Object.prototype_ (который также прототип 
    объекта, созданный через литералы объекта)

    class C extends B {
    }

*   Тип конструктора: наследник
*   Прототип _C_: _B_ 
*   Прототип _C.prototype_: _B.prototype_

    class C extends Object {
    }

*   Тип конструктора: наследник
*   Прототип _C_: _Object_
*   Прототип _C.prototype_: _Object.prototype_

Обратите внимание на следующее различие с первым случаем: 
Если нет _extends_, класс является базовым и выделяет в памяти экземпляры. 
Если класс расширяет _Object_, это производный класс объекта и выделяет 
экземпляры. Полученные экземпляры (в том числе их цепочки прототипов)
одинаковы, только получены разными способами.

    class C extends null {
    }

*   Тип конструктора: наследник
*   Прототип _C_: _Function.prototype_
*   Прототип _C.prototype_: _null_ 

Такой класс бесполезный: вызов через _new_ приведет к ошибке, потому 
что конструктор по умолчанию сделает вызов базового конструктора и  
_Function.prototype_ (базовый конструктор) не может быть конструктором вызова.
Единственный способ избежать ошибки — это добавить конструктор, который 
возвратит объект.

#### 4.3 Почему мы не можем наследовать встроенные конструкторы в ЕS5?

В ECMAScript 5, большинство встроенных конструкторов не могут быть 
унаследованы ([несколько обходных путей][5]).

Чтобы понять почему, давайте используем канонический ES5 шаблон наследования
_Array_. Как мы вскоре узнаем, это не работает.

    function MyArray(len) {
        Array.call(this, len); // (A)
    }
    MyArray.prototype = Object.create(Array.prototype);
    
К сожалению, если мы создадим _MyArray_, мы поймем, что он не работает 
должным образом: экземпляр свойства _length_ не изменится в ответ на наше 
добавление элементов в массив:

    > var myArr = new MyArray(0);
    > myArr.length
    0
    > myArr[0] = 'foo';
    > myArr.length
    0

Есть два препятствия, которые мешают _myArr_ быть правильным массивом.

**Первое припятствие: инициализация.** _this_,который вы передаете в 
конструктор _Array_ (в строке A) полностью игнорируется. Это значит, что вы 
не можете использовать _Array_ чтобы настроить экземпляр который создал 
_MyArray_.

    > var a = [];
    > var b = Array.call(a, 3);
    > a !== b  // a is ignored, b is a new object
    true
    > b.length // set up correctly
    3
    > a.length // unchanged
    0
    

**Второе припятствие: выделение памяти.** Экзепляры объектов, созданные через 
_Array_ являются _экзотичными_ (термин, используемый в спецификации ECMAScript 
для объектов, которые имеют особенности, которые нормальные объекты не имеют): 
их свойства _length_ отслеживают и влияют на управление элементами массива. 
В общем, экзотические объекты могут быть созданы с нуля, но вы не можете 
преобразовать существующий обычный объект в экзотический. К сожалению, 
это то, что делает _Array_, когда вызывается на строке A: 
Он должен был превратить обычный объект, созданный  из _MyArray_ в 
экзотический объект массива.

##### Решение: ES6 наследование

В ECMAScript 6, наследование _Array_ выглядит следующим образом:

    class MyArray extends Array {
        constructor(len) {
            super(len);
        }
    }
    

Это работает (но это не то, что ES6 транспайлеры могут поддерживать, это 
зависит от того, поддерживает ли движок JavaScript это изначально):

    > let myArr = new MyArray(0);
    > myArr.length
    0
    > myArr[0] = 'foo';
    > myArr.length
    1
    

Сейчас рассмотрим, как подход к наследованию в ES6, позволяет обойти 
припятствия:

*   Выделение пямяти происходит в базовом конструкторе. Это значит, что
    _Array_ может выделить в памяти экзотический объект. В то время как 
    большая часть нового подхода связано с тем, как полученные конструкторы 
    ведут себя, этот шаг требует, чтобы базовый конструктор понимал 
    _new.target_ и делал _new.target.prototype_ прототипом выделенного 
    экземпляра.

*   Инициализация также происходит в базовом конструкторе, конструктор 
    класса-наследника получает инициализированный объект и работает с ним, 
    вместо того, чтобы создавать собственный объект и отдавать его 
    конструктору базового класса, чтобы тот его создавал.


#### 4.4 Ссылка на базовые свойства в методах

Следующий ES6 код делает вызов базового метода на строке B.

    class Point {
        constructor(x, y) {
            this.x = x;
            this.y = y;
        }
        toString() { // (A)
            return '(' + this.x + ', ' + this.y + ')';
        }
    }
    
    class ColorPoint extends Point {
        constructor(x, y, color) {
            super(x, y);
            this.color = color;
        }
        toString() {
            return super.toString() // (B)
                   + ' in ' + this.color;
        }
    }
    
    let cp = new ColorPoint(25, 8, 'green');
    console.log(cp.toString()); // (25, 8) in green
    

Чтобы понять как работает базовые вызовы, давайте взглянем на диаграмму 
объекта _cp_:

![][9] 

_ColorPoint.prototype.toString_ делает вызов метода базового класса (строка B)
(начиная со строки A), который переопределен. Давайте вызовем объект, в 
котором хранится этот метод, _домашний объект_. Например, 
_ColorPoint.prototype_ - это _домашний объект_ для 
_ColorPoint.prototype.toString()_.

Вызов базового класса на строке B состоит из трёх этапов:

1.  Начинается поиск в прототипе домашнего объекта текущего метода.

2.  Поиск метода с названием _toString_. Этот метод должен быть найден в 
    объекте, где начался поиск, или позже по цепочке прототипов.   

3.  Вызвать этот метод с текущим _this_. Причина почему это происходит: 
    метод вызываемый как базовый должен иметь возможность доступа к тем же 
    свойствам экземпляра (в нашем примере, к свойствам _cp_).

Обратите внимание, что даже если вы только получаете или установливаете 
свойство (без вызова метода), вам все равно придется учитывать _this_ 
в шаге 3, потому что свойство может быть реализовано через get'ер или set'ер.

Давайте реализуем эти шаги в трех различных, но эквивалентных способах:

    // Variation 1: super-method calls in ES5
    var result = Point.prototype.toString.call(this) // steps 1,2,3
    
    // Variation 2: ES5, refactored
    var superObject = Point.prototype; // step 1
    var superMethod = superObject.toString; // step 2
    var result = superMethod.call(this) // step 3
    
    // Variation 3: ES6
    var homeObject = ColorPoint.prototype;
    var superObject = Object.getPrototypeOf(homeObject); // step 1
    var superMethod = superObject.toString; // step 2
    var result = superMethod.call(this) // step 3
    
Способ 3 показывает как в ECMAScript 6 обрабатываются вызовы базового класса. 
Этот подход поддерживается [двумя внутренними *привязками*][10], которые имеют 
_состояния_ функций (_состояние_ обеспечивает хранилище, так называемые 
_привязки_, для переменных окружения):

*   _[[thisValue]]_: Эта внутрення привязка также есть и в ECMAScript 5 и
    хранит значение переменной _this_.
*   `[[HomeObject]]`: Относится к домашнему объекту состояния функции. 
    Заполняется через внутреннее свойство _[[HomeObject]]_ которое имеют все 
    функции, использововашие _super_.  И привязка и свойство являются новыми 
    в ECMAScript 6.
   
Определение метода в литерале класса, который использует _super_, теперь имеет
особенность: это значение все еще функция, но имеет внутреннее свойство 
_[[HomeObject]]_. Это свойство устанавливается определением метода и не может
быть изменено в JavaScript. Таким образом, вы не можете перенести этот метод
в другой объект.

Использование _super_ не допускается для обращения к свойству в определениях 
функций, выражениях функций и генераторах.

Ссылаться на базовые свойства удобно, когда используются прототипы цепочек, 
поэтому вы можете использовать их в определениях методов, внутри литералов 
объектов и литералах классов (класс при этом может быть унаследованным или 
нет, метод может быть статическим или нет).

### 5. Пояснение вызовов конструктора через JavaScript код

Код JavaScript в этом разделе достаточно упрощен по сравнению с тем, как 
спецификация описывает вызовы конструктора и вызовы базового конструктора. 
Это может быть интересно для вас, если вы предпочитаете объяснения кода 
человеческим языком. Прежде чем мы углубимся в функциональность, мы должны 
понимать несколько других механизмов.

#### 5.1 Внутренние переменные и свойства

Спецификация описывает внутренние переменные и свойства в двойных скобках 
(_[[Foo]]_). В коде я использую двойные подчеркивания вместо этого 
(`__Foo__`).

Внутренние переменные используемые в коде:

*   _[[NewTarget]]_: Операнд оператора _new_, который запускает текущий вызов 
    конструктора (передается, если [[Construct]] вызывается рекурсивно через
    _super()_).
*   _[[thisValue]]_: Хранит значение _this_.
*   _[[FunctionObject]]_: Ссылается на функцию, которая в настоящее 
    время выполняется.

Внутренние свойства используемые в коде:

*   _[[Construct]]_: Все функции конструткора (включая также созданные 
    классом) имеют этот собственный (не наследуемый) метод. Он реализует 
    вызов конструктора и вызывается через _new_.
*   _[[ConstructorKind]]_: Свойство функций конструктора значение которого
    либо _'base'_ либо _'derived'_.

#### 5.2 Состояния

_Состояния_ обеспечивают хранилище для переменных, одно состояние на 
окружение. Состояния управляются как стек. Состояние на вершине стека 
считается активным. Следующий код демонстриурет как состояния обрабатываются.

    /**
     * Function environments are special, they have a few more
     * internal variables than other environments.
     * (`Environment` is not shown here)
     */
    class FunctionEnvironment extends Environment {
        constructor(Func) {
            // [[FunctionObject]] is a function-specific
            // internal variable
            this.__FunctionObject__ = Func;
        }
    }
    
    /**
     * Push an environment onto the stack
     */
    function PushEnvironment(env) { ··· }
    
    /**
     * Pop the topmost environment from the stack
     */
    function PopEnvironment() { ··· }
    
    /**
     * Find topmost function environment on stack
     */
    function GetThisEnvironment() { ··· }
    

#### 5.3 Вызов конструктора

Давайте начнем с основ ([ES6 спецификация, Секция. 9.2.3][11]), где вызовы 
конструктора обрабатываются для функций:

    /**
     * All constructible functions have this own method,
     * it is called by the `new` operator
     */
    AnyFunction.__Construct__ = function (args, newTarget) {
        let Constr = this;
        let kind = Constr.__ConstructorKind__;
    
        let env = new FunctionEnvironment(Constr);
        env.__NewTarget__ = newTarget;
        if (kind === 'base') {
            env.__thisValue__ = Object.create(newTarget.prototype);
        } else {
            // While `this` is uninitialized, getting or setting it
            // throws a `ReferenceError`
            env.__thisValue__ = uninitialized;
        }
    
        PushEnvironment(env);
        let result = Constr(...args);
        PopEnvironment();
    
        // Let’s pretend there is a way to tell whether `result`
        // was explicitly returned or not
        if (WasExplicitlyReturned(result)) {
            if (isObject(result)) {
                return result;
            }
            // Explicit return of a primitive
            if (kind === 'base') {
                // Base constructors must be backwards compatible
                return env.__thisValue__; // always initialized!
            }
            throw new TypeError();
        }
        // Implicit return
        if (env.__thisValue__ === uninitialized) {
            throw new ReferenceError();
        }
        return env.__thisValue__;
    }
    

#### 5.4 Вызов базового конструктора

Вызов базового конструктора обрабатывается следующим образом
([ES6 спецификация, Секция. 12.3.5.1][12]).

    /**
     * Handle super-constructor calls
     */
    function super(...args) {
        let env = GetThisEnvironment();
        let newTarget = env.__NewTarget__;
        let activeFunc = env.__FunctionObject__;
        let superConstructor = Object.getPrototypeOf(activeFunc);
    
        env.__thisValue__ = superConstructor
                            .__Construct__(args, newTarget);
    }
    

### 6. Шаблон разновидностей

Еще один механизм встроенных конструкторов была расширен в ECMAScript 6: 
если метод, такой как _Array.prototype.map()_, возвращает экземпляр, то какой 
конструктор следует использовать для создания этого экземпляра? По умолчанию,
используется тот же конструктор, который создал _this_, но некоторые 
наследники могут оставаться прямым экземпляром _Array_. ES6 позволяет 
классам-наследникам переопределить значение по умолчанию с помощью так
называемого _шаблона разновидности_:

*   При создании нового экземпляра _Array_, методы, такие как _map()_ 
    используют конструктор, хранящийся в _this.constructor[Symbol.species]_.
*   Если конструктор наследника _Array_ ничего не делает, он наследует
    _Array[Symbol.species]_. Это свойство является get'ером, который 
    возвращает _this_.

Вы можете изменить настройки по умолчанию, с помощью статического get'ера 
(строка A):

    class MyArray1 extends Array {
    }
    let result1 = new MyArray1().map(x => x);
    console.log(result1 instanceof MyArray1); // true
    
    class MyArray2 extends Array {
        static get [Symbol.species]() { // (A)
            return Array;
        }
    }
    let result2 = new MyArray2().map(x => x);
    console.log(result2 instanceof MyArray2); // false
    

Альтернативой является использование _Object.defineProperty()_ (вы не можете 
использовать присвоение, т.к. вызываете set'ер, который не существует):

    Object.defineProperty(
        MyArray2, Symbol.species, {
            value: Array
        });
    

Следующие get'еры возвращают _this_, это означает, что такие методы 
как _Array.prototype.map()_ используют конструктор который создал текущий
экземпляр их результатов.

*   `Array.get [Symbol.species]()`
*   `ArrayBuffer.get [Symbol.species]()`
*   `Map.get [Symbol.species]()`
*   `Promise.get [Symbol.species]()`
*   `RegExp.get [Symbol.species]()`
*   `Set.get [Symbol.species]()`
*   `%TypedArray%.get [Symbol.species]()`

### 7. Заключение

#### 7.1 Специализация функций

Существует интересная тенденция в ECMAScript 6: ранее единственный вид 
функции был на трех ролях: функция, метод и конструктор. 
В ES6, есть еще специализация:

*   Стрелочные функции специализируются на функциях обратного вызова, где их
    использует _this_ в окружающем методе, или конструктор как преимущество. 
    Без _this_ они не имеют смысла как методы и они бросают исключение, 
    если вызываеются через _new_.

*   Определения метода позволяют использовать _super_, определив свойство 
    _[[HomeObject]]_. Функции, которые они производят, не могут быть 
    вызываемыми конструкторами.

*   Определения классов являются единственным способом создания производных 
    конструкторов (включающий в ES6 наследование, которое работает на 
    встроенных конструкторах). Определения классов создают функции, которые 
    могут быть только вызываемыми конструкторами.   

#### 7.2 Будущее классов

Дизайн классов был «максимально минимальным». Обсуждались несколько 
расширяющих функциональностей, но в конечном итоге от них отказались, 
чтобы получить вариант, который был принят единогласно TC39.

Будущие версии ECMAScript теперь могут расширять этот минималистичный дизайн 
- классы обеспечивают основу для такой функциональности как признаки (или 
миксины),
значения объектов (где различные объекты равны, если они имеют одинаковое 
содержание) и константные классы (которые создают неизменяемые экземпляры).

#### 7.3 Нужны ли классы JavaScript'у?

Классы являются спорными в сообществе JavaScript. С одной стороны, люди 
которые пришли из языков, основанных на классах — счастливы, что им больше не 
придется иметь дело с необычными механизмами наследования в JavaScript.
С другой стороны, существует множество JavaScript программистов, которые 
утверждают, что в JavaScript прототипное наследование проще, чем наследование 
с помощью конструкторов. [6]

Классы ES6 обеспечивают несколько очевидных преимуществ:

*   Они обратно совместимы с большинством текущего кода.

*   По сравнению с конструкторами и наследованием конструкторов, классы 
    реализуют это проще для начинающих.

*   Наследование поддерживается в языке.

*   Встроенные конструкторы наследуемы.

*   Теперь не нужны библиотеки, реализующие наследование; код станет более 
    переносимым между фреймворками.   

*   Они обеспечивают основу для расширенной функциональности в будущем 
    (миксины и т.п).

*   Они помогают инструментам, которые статически анализируют код (IDE, 
    проверки типов, проверики стилей, и т.д.).

Я закончил эту статью с классами и я рад, что они есть в ES6. Я бы
предпочел, чтобы они были прототипными (на основе конструктора объектов [6], 
а не конструктора функций), но я также понимаю, что обратная совместимость 
является важной.

### 8. Для дополнительного чтения

Подтверждение: #1 был важным источником этой статьи.

1. [Instantiation Reform: One last time](https://github.com/rwaldron/tc39-notes/blob/master/es6/2015-01/jan2015-allen-slides.pdf), slides by Allen Wirfs-Brock.
2. [Exploring ES6: Upgrade to the next version of JavaScript](http://exploringjs.com/), book by Axel
3. [Symbols in ECMAScript 6](http://www.2ality.com/2014/12/es6-symbols.html)
4. [Iterators and generators in ECMAScript 6](http://www.2ality.com/2013/06/iterators-generators.html)
5. [Meta programming with ECMAScript 6 proxies](http://www.2ality.com/2014/12/es6-proxies.html)
6. [Prototypes as classes – an introduction to JavaScript inheritance](http://www.2ality.com/2011/06/prototypes-as-classes.html)

 [1]: https://github.com/rwaldron/tc39-notes/blob/master/es6/2015-01/jan-27.md#44-subclass-instantiation-reformation-status-and-open-issues
 [2]: https://people.mozilla.org/~jorendorff/es6-draft.html#sec-ecmascript-function-objects-call-thisargument-argumentslist
 [4]: http://speakingjs.com/es5/ch17.html#getters_setters
 [5]: http://speakingjs.com/es5/ch28.html
 [6]: https://people.mozilla.org/~jorendorff/es6-draft.html#sec-functions-and-classes
 [7]: img/subclassing_es6.jpg
 [8]: https://people.mozilla.org/~jorendorff/es6-draft.html#sec-runtime-semantics-classdefinitionevaluation
 [9]: img/supercalls.jpg
 [10]: https://people.mozilla.org/~jorendorff/es6-draft.html#sec-function-environment-records
 [11]: https://people.mozilla.org/~jorendorff/es6-draft.html#sec-ecmascript-function-objects-construct-argumentslist-newtarget
 [12]: https://people.mozilla.org/~jorendorff/es6-draft.html#sec-super-keyword-runtime-semantics-evaluation
 
 [Схема наследования]: img/methods.jpg