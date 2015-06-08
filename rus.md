[Недавно][1], TC39 определили финальную семантику классов в ECMAScript 6[2].
Это статья поясняет как работает их реализация. Наиболее значимые из недавних
изменений связаны с тем, как реализована система наследования классов.

### 1. Обзор {#overview}

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
    

### 2. Основы {#the_essentials}

#### 2.1 Базовые классы {#base_classes}

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
    
Использовать этот класс можно просто вызвав конструктор функции как в ES5:

    > var p = new Point(25, 8);
    > p.toString()
    '(25, 8)'
    
По факту, результатом создания такого класса будет функция:

    > typeof Point
    'function'
    
Однако, вы можете вызывать класс только через ```new```, а не через вызов
функции ([Секция 9.2.2][2] в спецификации):

    > Point()
    TypeError: Classes can’t be function-called
    

##### Объявления классов не поднимаются {#class_declarations_are_not_hoisted}

Объявления функций _поднимаются_: объявленные внутри общей области видимости
функции сразу же доступны независимо от того, где они были объявлены. Это 
означает, что вы можете вызвать функцию, которая будет объявлена позднее.

    foo(); // works, because `foo` is hoisted
    
    function foo() {}
    
В отличие от функций, определения классов не _поднимаются_. Таким образом, 
класс существует только после того, как его определение было достигнуто
и выполнено. Попытка создания класса до этого момента приведет к 
_ReferenceError_:

    new Foo(); // ReferenceError

    class Foo {}
    
Причина этого ограничения в том, что классы могут быть наследниками. Это 
поддерживается с помощью выражения _extends_, значение которого может быть
произвольным. Это выражение должно вычислено в правильном месте,
которое не может быть _поднято_.

Отсутствие механизма _поднятия_ - это не такое большое ограничение, чем вы
могли бы подумать. Например, функция, которая определена до определения 
класса, все еще может ссылаться на этот класс, но вы вынуждены ждать
выполнения определения класса до того, как сможете выполнить эту функцию.

    function functionThatUsesBar() {
        new Bar();
    }

    functionThatUsesBar(); // ReferenceError
    class Bar {}
    functionThatUsesBar(); // OK


##### Выражения класса {#class_expressions}

Так же как и функции, есть два вида _определения классов_, два способа 
определить класс: _объявление класса_ и _выражение класса_.

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
    

#### 2.2 Внутри тела определения класса {#inside_the_body_of_a_class_definition}

Тело класса может содержать только методы, но не свойства. 
Прототип, имеющий свойства обычно считается анти-паттерном.

##### _constructor_, статические методы, прототипные методы {#constructor_static_methods_prototype_methods}

Давайте рассмотрим три вида методов, которые вы часто можете найти в классах.

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
    
Диаграмма объекта для это определения класса выглядит следующим образом.
Совет для понимания:_[[Prototype]]_ это отношения наследования между объектами,
в то время как _prototype_  это обычное свойство, значением которого является
объект. _prototype_ это специальное свойство, значение которого оператор _new_
использует как прототип для создаваемых объектов.

![][3]

**Во первых, псевдо-метод _constructor_.** Этот метод является особенным, так 
как он определяет функцию, которая представляет собой класс:

    > Foo === Foo.prototype.constructor
    true
    > typeof Foo
    'function'

Иногда его называют _конструктор класса_. Он имеет особенности которые обычный 
конструктор функции не имеет (главным образом, способность конструтора вызывать
конструктор базового класса через _super()_, о котором я расскажу чуть позже).

**Во вторых, статические методы.** _Статичесие свойства_ (или _свойства класса)
являются свойствами самого _Foo_. Если вы определили метод с помощью _static_,
значит вы создали метод класса:

    > typeof Foo.staticMethod
    'function'
    > Foo.staticMethod()
    'classy'
    

**В третьих, прототипные методы.** _свойства прототипа_ _Foo_ являются и 
свойствами _Foo.prototype_. Это, как правило, методы и наследуются 
экземплярами _Foo_.

    > typeof Foo.prototype.prototypeMethod
    'function'
    > foo.prototypeMethod()
    'prototypical'
    

##### Get'еры и Set'еры {#getters_and_setters}

Синтаксис для get'еров и set'еров такой же как и в 
[в ECMAScript 5 литералах объекта][4]:

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
    

##### Вычисляемые имена методов {#computed_method_names}

Вы можете определить имя метода с помощью выражения, если вы поместите его 
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
Например, если объект имеет метод с ключом _Symbol.iterator_, это - 
_итератор_ [4]. Это означает, что его содержимое может быть итерировано 
циклом _for-of_ или другими механизмами языка.

    class IterableClass {
        [Symbol.iterator]() {
            ···
        }
    }
    

##### Генераторы {#generator_methods}

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
    
#### 2.3 Классы наследники {#subclassing}

Ключевое слово _extends_ позволяет создать класс-наследник существующего 
конструктора (который был или, возможно, не был определен с помощью класса):

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

*   _Point_ - это _базовый класс_, потому что он не имеет выражения _extends_.
*   _ColorPoint_ - _производный класс_.

Есть два способа использовать ключевое слово _super_:

*   _Конструктор класса_ (псевдо-метод _constructor_ в теле класса)
    использует его как вызов функции (_super(···)_), для того чтобы  
    вызвать базовый конструктор (строка A).
*   Определения методов (в объектах, заданых через литерал, или классах,
    статических или нет) используют это для вызова свойства (_super.prop_)
    или вызова метода (_super.method(···)_), для ссылки на свойства базового
    класса (строка B).

##### Прототип класса наследника является базовым классом {#the_prototype_of_a_subclass_is_the_superclass}

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
    

##### Вызов базового конструктора {#super-constructor_calls}

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

Не вызвав `super()` в производном классе, вы получите ошибку:

    class Foo {}
    
    class Bar extends Foo {
        constructor() {
        }
    }
    
    let bar = new Bar(); // ReferenceError
    

##### Переопределение результата конструктора {#overriding_the_result_of_a_constructor}

Так же, как в ES5, можно переопределить результат конструктора, явно возвращая 
объект:

    class Foo {
        constructor() {
            return Object.create(null);
        }
    }
    console.log(new Foo() instanceof Foo); // false

Если вы так сделаете, то не имеет значения, инициализирован ли _this_ или нет.
Другими словами: вы не обязаны вызывать _super()_ в производном конструкторе, 
если переопределите результат таким образом.

##### Конструкторы по умолчанию для классов {#default_constructors_for_classes}

Если вы не указываете _constructor_  для базового класса, тогда испольузется
слудающая конструкция:

    constructor() {}

Для дочерних классов, используется конструктор по умолчанию:

    constructor(...args) {
        super(...args);
    }

##### Наследования встроенных конструкторов {#subclassing_built-in_constructors}   

В ECMAScript 6, вы наконец-то можете наследоваться от всех встроенных 
конструкторов ([обходные пути в ES5][5], но здесь есть значительные ограничения).

Например, теперь вы можете создавать свои собственные классы исключений 
(которые будут наследовать такие особенности, как стек вызовов для 
большинства движков):

    class MyError extends Error {    
    }
    throw new MyError('Something happened!');

Вы также можете наследоваться от _Array_, экзмепляры которого правильно работают 
с _length_:

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

Заметьте, что наследование от встроенных конструкторов это то, что движок 
должен поддерживать изначально, вы не сможете получить эту функциональность 
с помощью транспайлеров.

### 3. Детали классов {#the_details_of_classes}

То, что мы до сих пор рассматривали является основой классов. Если вам 
интересно узнать подробнее механизм классов, то вам нужно читать дальше. 
Давайте начнем с синтаксиса классов. Ниже приводится немного модифицированная 
верcия синтаксиса предложенного в [Секции. A.4 специфакации ECMAScript 6][6].

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

*   Расширяемое значение может быть произвольным выражением. Это значит, что 
    вы можете написать следующий код: 
    
        class Foo extends combine(MyMixin, MySuperClass) {}

*   Между методами допускается точка с запятой.

#### 3.1 Различные проверки {#various_checks}

*   Проверки ошибок: имя класса не может быть _eval_ or _arguments_; дубликаты
    имен классов не допускаются; название _constructor_ может использоваться
    только для обычных методов, для get'еров, set'еров и генераторов - 
    не допускается.   

*   Классы не могут быть вызываемой функцией. Иначе они бросают исключение 
    _TypeException_

*   Методы прототипа не могут использоваться как конструкторы:
    
        class C {
            m() {}
        }
        new C.prototype.m(); // TypeError
        

#### 3.2 Атрибуты свойств {#attributes_of_properties}

Class declarations create (mutable) let bindings. For a given class `Foo`:

*   Static methods `Foo.*` are writable and configurable, but not enumerable.
    Making them writable allows for dynamic patching.
   
*   A constructor and the object in its property `prototype` have an immutable
    link:
   
    *   `Foo.prototype` is non-writeable, non-enumerable, non-configurable.
    *   `Foo.prototype.constructor` is non-writeable, non-enumerable, non-
        configurable.
       
*   Prototype methods `Foo.prototype.*` are writable and configurable, but not
    enumerable.
   

Note that method definitions in object literals produce enumerable properties.

### 4. Детали наследования классов {#the_details_of_subclassing}

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

#### 4.1 Цепочки прототипов {#prototype_chains}

На диаграмме видно что есть 2 _цепочки прототипов_ (объекты связаны через 
отношения `[[Prototype]]`, которые наследуются):

*   Левая колонка: классы (функции). Прототипом производного класса является 
    расширенный класс. Прототип базового класса является _Function.prototype_, 
    которая также есть прототип функции:
    
        > const getProto = Object.getPrototypeOf.bind(Object);
        
        > getProto(Point) === Function.prototype
        true
        > getProto(function () {}) === Function.prototype
        true
        

*   Правая колонка: цепочки прототипов экземпляров. Вся цель класса - 
    установить эту цепочку прототипов. Цепочка протипов заканчивается с
    _Object.prototype_ (чей прототип является _null_), который также прототип 
    объектов, созданных через литералы объекта:
    
        > const getProto = Object.getPrototypeOf.bind(Object);
        
        > getProto(Point.prototype) === Object.prototype
        true
        > getProto({}) === Object.prototype
        true
        
Из цепочки прототипов в левой колонке следует, что статические свойства наследуются.

Цепочка прототипов в левой колонке приводит к наследованию статических свойств.

#### 4.2 Выделение памяти и инициализация экземпляров объектов {#allocating_and_initializing_the_instance_object}

Потоки данных между конструкторами классов отличается от канонического пути 
наследования в ES5. Под капотом, это выглядит примерно так:

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

*   _new.target_ является неявным параметром, который имеют все функции.  Это
    вызов конструктора, где _this_ является вызовом метода.
    
    *   Если конструктор напрямую вызывается через _new_, его значение это
        и есть этот конструктор (строка B).
    *   Если конструктор был вызван через _super()_, его значение это
        _new.target_ конструктора который был вызван (строка A).
    *   Вызвав функцию обычным способом, значение будет _undefined_. Это 
        значит, что вы можете использовать _new.target_ чтобы определить была 
        ли функция функцией вызова или вызовом конструктора (через _new_).
    *   Внутри стрелочной функции _new.target_ ссылается на _new.target_
        окружающие не стрелочные функции.

*   _Reflect.construct()_ [5] позволяет вызвать конструктор при задании
    _new.target_ в качестве последнего параметра.

Преимуществом этой реализации наследования является то, что это позволяет 
писать нормальный код для наследования встроенных констукторов (такие как 
_Error_ и _Array_). Последний раздел объясняет, почему иной подход был 
необходим.

##### Безопасные проверки {#safety_checks}

*   _this_ инициалированый в производных конструкторах значит что,
    исключение будет бросаться если к _this_ обращаются до того как 
    вызвали _super()_.
*   После инициализации `this`, вызова _super()_ приведет к _ReferenceError_.
    Это защита от двойного вызова _super()_.
*   Если конструктор возвращается неявно (без _return_), тогда результат будет
    _this_. Если _this_ инициализирован, тогда бросится исключение 
    _ReferenceError_ . Это защита от невызова _super()_.
*   Если конструктор явно возвращает не объект (включая _undefined_ и _null_),
    результатом будет _this_ (это поведение остававляет совместимость с ES5 
    и ранее). Если _this_ инициализирован, тогда бросится исключение 
    _TypeError_.
*   Если конструктор явно возвращает объект, тогда он и будет результатом.
    Тогда не имеет значение инициализирован _this_ или нет.

##### Выражение _extends_ {#the_extends_clause}

Давайте рассмотрим как выражение _extends_ влияет на работу класса
([Секция. 14.5.14 спецификации][8]).

Значение _extends_ должно быть "конструктивно" (ссылаться через _new_) 
хотя _null_ тоже поддерживается.

    class C {
    }
    

*   Тип конструктора: базовый
*   Прототип _C_: _Function.prototype_ (как обычная функция)
*   Прототип _C.prototype_: _Object.prototype_ (который также прототип объекта
    созданный через литералы объекта)

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

Такой класс не является полезным: вызов через _new_ приведет к ошибке, потому 
что конструктор по умолчанию сделает вызов базового конструктора и  
_Function.prototype_ (базовый конструктор) не может быть конструктором вызова.
Единственный способ избежать ошибки - это добавить конструктор который 
возвратит объект.

#### 4.3 Почему мы не можем наследовать встроенные конструкторы в ЕС5? {#why_can’t_you_subclass_built-in_constructors_in_ES5}

В ECMAScript 5, большинство встроенных конструкторов не могут быть унаследованы
([несколько обходных путей][5]).

Чтобы понять почему, давайте использовать канонический ES5 шаблон наследования
_Array_. Как мы вскоре узнаем, это не работает.

    function MyArray(len) {
        Array.call(this, len); // (A)
    }
    MyArray.prototype = Object.create(Array.prototype);
    
К сожалению, если мы создадим _MyArray_, мы поймем, что он не работает должным
образом: экземпляр свойства _length_ не изменится в ответ на наше добавление
элементов в массив:

    > var myArr = new MyArray(0);
    > myArr.length
    0
    > myArr[0] = 'foo';
    > myArr.length
    0

Есть два препятствия, которые мешают _myArr_ быть правильным массивом.

**Первое припятствие: инициализация.** _this_ ручной конструктор _Array_ 
(в строке A) полностью игнорируется. Это значит что вы не можете использовать
_Array_ чтобы установить экземпляр который бы создавал _MyArray_.

    > var a = [];
    > var b = Array.call(a, 3);
    > a !== b  // a is ignored, b is a new object
    true
    > b.length // set up correctly
    3
    > a.length // unchanged
    0
    

**Второе припятствие: выделение памяти.** Экзепляры объектов созданные через 
_Array_ являются *экзотичными* (термин, используемый в спецификации ECMAScript 
для объектов, которые имеют особенности, которые нормальные объекты не имеют): 
их свойства _length_ отслеживают и влияют на управление элементами массива. 
В общем, экзотичные объекты могут быть созданы с нуля, но вы не можете 
преобразовать существующий обычный объект в экзотический. К сожалению, 
это то, что делает _Array_, когда вызывается на строке A: 
Он должен был превратить обычный объект, созданный  из _MyArray_ в 
экзотический объект массива.

##### Решение: ES6 наследование {#the_solution_ES6_subclassing}

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
    что _Array_ может выделить в памяти экзотический объект. В то время как 
    большая часть нового подхода связано с тем, как полученные конструкторы 
    ведут себя, этот шаг требует, чтобы базовый конструктор понимает 
    _new.target_ и делал _new.target.prototype_ в прототипе выделенного 
    экземпляра.

*   Инициализация также происходит в базовом конструкторе, конструктор 
    класса-наследника получает инициализированный объект и работает с ним, 
    вместо того, чтобы создавать собственный объект, отдавать его конструктору 
    базового класса, чтобы тот его создавал.


#### 4.4 Отсылка к базовым свойствам в методах {#referring_to_super_properties_in_methods}

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

_ColorPoint.prototype.toString_ делает базовый вызов (строка B) метода 
(начиная со строки A) который переопределен. Давайте вызовем объект, в котором 
хранится *главный объект* этот метод. Например, 
_ColorPoint.prototype_ главный объект для _ColorPoint.prototype.toString()_.

Базовый вызов на строке B включает три этапа:

1.  Начинается поиск в прототипе главного объекта текущего метода.

2.  Look for a method whose name is `toString`. That method may be found in the
    object where the search started or later in the prototype chain.
   

3.  Call that method with the current `this`. The reason for doing so is: the
    super-called method must be able to access the same instance properties (in our 
    example, the properties of
   `cp`).

Note that even if you are only getting or setting a property (not calling a
method), you still have to consider`this` in step #3, because the property may
be implemented via a getter or a setter.

Let’s express these steps in three different, but equivalent, ways:

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
    

Variation 3 is how ECMAScript 6 handles super-calls. This approach is supported
by[two internal *bindings*][10] that the *environments* of functions have (*
environments* provide storage space, so-called *bindings*, for the variables in
a scope):

*   `[[thisValue]]`: This internal binding also exists in ECMAScript 5 and
    stores the value of
   `this`.
*   `[[HomeObject]]`: Refers to the home object of the environment’s function
    . Filled in via an internal property
   `[[HomeObject]]` that all methods have that use `super`. Both the binding
    and the property are new in ECMAScript 6.
   

A method definition in a class literal that uses `super` is now special: Its
value is still a function, but it has the internal property`[[HomeObject]]`.
That property is set up by the method definition and can’t be changed in 
JavaScript. Therefore, you can’t meaningfully move such a method to a different 
object.

Using `super` to refer to a property is not allowed in function declarations,
function expressions and generator functions.

Referring to super-properties is handy whenever prototype chains are involved,
which is why you can use it in method definitions inside object literals and 
class literals (the class can be derived or not, the method can be static or not
).

### 5. Constructor calls explained via JavaScript code {#constructor_calls_explained_via_javascript_code}

The JavaScript code in this section is a much simplified version of how the
specification describes constructor calls and super-constructor calls. It may be
interesting to you if you prefer code to explanations in human language. Before 
we can delve into the actual functionality, we need to understand a few other 
mechanisms.

#### Internal variables and properties {#internal_variables_and_properties}

The specification writes internal variables and properties in double brackets
(`[[Foo]]`). In the code, I use double underscores, instead (`__Foo__`).

Internal variables used in the code:

*   `[[NewTarget]]`: The operand of the `new` operator that triggered the
    current constructor call (passed on if
   `[[Construct]]` is called recursively via `super()`).
*   `[[thisValue]]`: Stores the value of `this`.
*   `[[FunctionObject]]`: Refers to the function that is currently executed.

Internal properties used in the code:

*   `[[Construct]]`: All constructor functions (including those created by
    classes) have this own (non-inherited) method. It implements constructor calls 
    and is invoked by
   `new`.
*   `[[ConstructorKind]]`: A property of constructor functions whose value is
    either
   `'base'` or `'derived'`.

#### Environments {#environments}

*Environments* provide storage space for variables, there is one environment
per scope. Environments are managed as a stack. The environment on top of that 
stack is considered active. The following code is a sketch of how environments 
are handled.

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
    

#### Constructor calls {#constructor_calls}

Let’s start with the default way ([ES6 spec Sect. 9.2.3][11]) in which
constructor calls are handled for functions:

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
    

#### Super-constructor calls {#super-constructor_calls_2}

Super-constructor calls are handled as follows ([ES6 spec Sect. 12.3.5.1][12]

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
    

### The species pattern {#the_species_pattern}

One more mechanism of built-in constructors has been made extensible in
ECMAScript 6: If a method such as`Array.prototype.map()` returns a fresh
instance, what constructor should it use to create that instance? The default is
to use the same constructor that created`this`, but some subclasses may want it
to remain a direct instance of`Array`. ES6 lets subclasses override the default
, via the so-called*species pattern*:

*   When creating a new instance of `Array`, methods such as `map()` use the
    constructor stored in
   `this.constructor[Symbol.species]`.
*   If a sub-constructor of `Array` does nothing, it inherits 
    `Array[Symbol.species]`. That property is a getter that returns `this`.

You can override the default, via a static getter (line A):

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
    

An alternative is to use `Object.defineProperty()` (you can’t use assignment
, as that would trigger a setter, which doesn’t exist):

    Object.defineProperty(
        MyArray2, Symbol.species, {
            value: Array
        });
    

The following getters all return `this`, which means that methods such as 
`Array.prototype.map()` use the constructor that created the current instance
for their results.

*   `Array.get [Symbol.species]()`
*   `ArrayBuffer.get [Symbol.species]()`
*   `Map.get [Symbol.species]()`
*   `Promise.get [Symbol.species]()`
*   `RegExp.get [Symbol.species]()`
*   `Set.get [Symbol.species]()`
*   `%TypedArray%.get [Symbol.species]()`

### Conclusion {#conclusion}

#### The specialization of functions {#the_specialization_of_functions}

There is an interesting trend in ECMAScript 6: Previously, a single kind of
function took on three roles: real function, method and constructor. In ES6, 
there is specialization:

*   Arrow functions are specialized for non-method callbacks, where them
    picking up the
   `this` of their surrounding method or constructor is an advantage. Without
   `this`, they don’t make much sense as methods and they throw an exception
    when invoked via
   `new`.

*   Method definitions enable the use of `super`, by setting up the property 
    `[[HomeObject]]`. The functions they produce can’t be constructor-called.

*   Class definitions are the only way to create derived constructors (enabling
    ES6-style subclassing that works for built-in constructors). Class definitions 
    produce functions that can only be constructor-called.
   

#### The future of classes {#the_future_of_classes}

The design maxim for classes was “maximally minimal”. Several advanced
features were discussed, but ultimately discarded in order to get a design that 
would be unanimously accepted by TC39.

Upcoming versions of ECMAScript can now extend this minimal design – classes
will provide a foundation for features such as traits (or mixins), value objects
(where different objects are equal if they have the same content) and const 
classes (that produce immutable instances
).

#### Does JavaScript need classes? {#does_javascript_need_classes%3F}

Classes are controversial within the JavaScript community. On one hand, people
coming from class-based languages are happy that they don’t have to deal with 
JavaScript’s unorthodox inheritance mechanisms, anymore. On the other hand, 
there are many JavaScript programmers who argue that what’s complicated about 
JavaScript is not prototypal inheritance, but constructors[6].

ES6 classes provide a few clear benefits:

*   They are backwards compatible with much of the current code.

*   Compared to constructors and constructor inheritance, classes make it
    easier for beginners to get started.
   

*   Subclassing is supported within the language.

*   Built-in constructors are subclassable.

*   No library for inheritance is needed, anymore; code will become more
    portable between frameworks.
   

*   They provide a foundation for advanced features in the future (mixins and
    more
    ).

*   They help tools that statically analyze code (IDEs, type checkers, style
    checkers, etc.
    ).

I have made my peace with classes and am glad that they are in ES6. I would
have preferred them to be prototypal (based on constructor objects[6], not
constructor functions), but I also understand that backwards compatibility is 
important.

### Further reading {#further_reading}

Acknowledgement: #1 was an important source of this blog post.


 [1]: https://github.com/rwaldron/tc39-notes/blob/master/es6/2015-01/jan-27.md#44-subclass-instantiation-reformation-status-and-open-issues

 [2]: https://people.mozilla.org/~jorendorff/es6-draft.html#sec-ecmascript-function-objects-call-thisargument-argumentslist
 [3]: img/methods.jpg
 [4]: http://speakingjs.com/es5/ch17.html#getters_setters
 [5]: http://speakingjs.com/es5/ch28.html

 [6]: https://people.mozilla.org/~jorendorff/es6-draft.html#sec-functions-and-classes
 [7]: img/subclassing_es6.jpg

 [8]: https://people.mozilla.org/~jorendorff/es6-draft.html#sec-runtime-semantics-classdefinitionevaluation
 [9]: img/supercalls.jpg

 [10]: https://people.mozilla.org/~jorendorff/es6-draft.html#sec-function-environment-records

 [11]: https://people.mozilla.org/~jorendorff/es6-draft.html#sec-ecmascript-function-objects-construct-argumentslist-newtarget

 [12]: https://people.mozilla.org/~jorendorff/es6-draft.html#sec-super-keyword-runtime-semantics-evaluation