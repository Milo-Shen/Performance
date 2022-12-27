# V8引擎优化机制之隐藏类和内联缓存

我们知道 Javascript 作为一种动态语言，性能方面与 c#, Java 之类的静态语言相比存在着一定的差距。而随着 Web 技术的发展，对 Javascript 的执行效率提出越来越高的要求。为了追求更好的性能，V8 引擎借鉴了大量的静态语言编译技术来优化引擎的执行效率。比如 V8 引擎放弃生成中间字节码，而是直接从 AST (抽象语法树)生成机器语言。与静态语言不同, Javascript 的程序在执行期间需要反复检查数据类型。因此，V8引擎中存在两种机制来优化这个过程。

## hidden class 隐藏类
于动态类型语言来说，由于类型的不确定性，在方法调用过程中，语言引擎每次都需要进行动态查询，这就造成大量的性能消耗，从而降低程序运行的速度。大多数的 Javascript 引擎会采用哈希表的方式来存取属性和寻找方法。而为了加快对象属性和方法在内存中的查找速度，V8 引擎引入了隐藏类 (Hidden Class) 的机制，起到给对象分组的作用。在初始化对象的时候，V8 引擎会创建一个隐藏类，随后在程序运行过程中每次增减属性，就会创建一个新的隐藏类或者查找之前已经创建好的隐藏类。每个隐藏类都会记录对应属性在内存中的偏移量，从而在后续再次调用的时候能更快地定位到其位置。

```javascript
function Person(name, age) {
    this.name = name;
    this.age = age;
}

var user_1 = new Person("Jack", 32);
var user_2 = new Person("Marry", 20);

user_1.email = "xiaoming@qq.com";
user_1.job = "teacher";

user_2.job = "chef";
user_2.email = "lisi@qq.com";
```

观察以上代码，当初始化Person对象的时候， 最开始会创建一个C0的隐藏类, 该类不带有任何属性。随后在调用构造器函数的时候，随着属性的增加，引擎会生成C1, C2的过渡隐藏类，隐藏类内部会记录属性的偏移量（offset)。之所以存在过渡隐藏类是为了在多个对象间能够共享隐藏类。

这里, 注意到 user_1 和 user_2 两个对象使用的是同一个构造函数, 所以它们会共享同一个隐藏类 C2。随后虽然 user_1 和 user_2 两个对象都添加了 job 和 email 两个属性, 但由于初始化顺序不同, 会生成不同的隐藏类。

不同初始化顺序的对象，所生成的隐藏类是不一样的。因此，在实际开发过程中，应该尽量保证属性初始化的顺序一致，这样生成的隐藏类可以得到共享。同时，尽量在构造函数里就初始化所有对象成员，减少隐藏类的产生。

```javascript
function User1 (name, age) {
    this.name = name;
    this.age = age;
}

function User2 (name, age) {
    this[name] = name;
    this.age = age;
}

// 花费 287 ms
for ( let i = 0; i < 9999999; i++ ) {
    new User1(`User_${i}`, i);
}

// 花费 9261 ms
for ( let i = 0; i < 9999999; i++ ) {
    new User2(`User_${i}`, i);
}
```
