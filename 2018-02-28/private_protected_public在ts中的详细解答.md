# 写在前面

> 写过java的同学在写ts的时候感觉得心应手,ts支持类型检查支持类型推断,语法啥的基本和java相同,但是对于js选手来说,由于js本身是一门动态语言,写起来是过瘾,在实例化一个构造函数之后更是可以随意修改对象,但是ts引入了class,相应地引入了public private 等访问修饰符,那么问题来了,我们该怎么看待这个访问修饰符？

### **举个例子**

```
	class Animal {
		constructor(private name: string, private legs: number) {
			
		}
		shout():void {
			console.log(this.name, this.legs)
		}
	}
	class Dog extends Animal {
		constructor() {
			super('dog', 4);
			this.name = 'cat'; // error 属性为私有属性，只能在Animal中访问
		}
		shout():void {

		}
	}

	这个在ts中直接报错,无法通过编译。

```

```
	class Animal {
		constructor(protected name: string, private legs: number) {
			
		}
		shout():void {
			console.log(this.name, this.legs)
		}
	}
	class Dog extends Animal {
		constructor() {
			super('dog', 4);
			this.name = 'cat';
		}
		shout():void {
			console.log(this.name);
		}
	}

	let d: Dog = new Dog();
	d.name // error name只能在Animal及其子类中访问。

```

```
	class Animal {
		constructor(public name: string, private legs: number) {
			
		}
		shout():void {
			console.log(this.name, this.legs)
		}
	}
	class Dog extends Animal {
		constructor() {
			super('dog', 4);
			this.name = 'cat';
		}
		shout():void {
			console.log(this.name);
		}
	}

	let d: Dog = new Dog();
	d.name  // 完全OK

```

> 目前已经能看出结论。private所修饰的field或者method不能为子类修改或者重写。protected所修饰的field或者method可以被子类修改或者重写。public随意。但是有一点需要注意,父类是protected或者public的field或者method不能设置成限制优先级低的。这个也是能从上面的例子中得到反推。

#### 总结---类似于java,只不过构造函数可以用(?:, ?:, ?:)来省去好几个, ts中只能有一个构造函数。
	