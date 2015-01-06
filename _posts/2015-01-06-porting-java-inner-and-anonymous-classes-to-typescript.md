---
layout: post
title:  "Porting Java Inner and Anonymous Classes To TypeScript"
date:   2014-01-06 13:18:00
---
Sometime between version 1.5 and 1.7.10 of the Minecraft, the source code has undergone significant refactoring and made use of previously unused Java features, which didn't exist in the previous version, namely [Nested](http://docs.oracle.com/javase/tutorial/java/javaOO/nested.html) and [Anonymous](http://docs.oracle.com/javase/tutorial/java/javaOO/anonymousclasses.html) classes, making my transpiler no longer "good enough" and thus needing.

Click [here](#main) to jump straight to the solution and code, otherwise, if you're interested...
A Little History First
---
I am presently working on "version 3" of my [JavaScript Redstone Simulator](http://mordritch.com/mc_rss/).

"Version 1" was a [clean room design](http://en.wikipedia.org/wiki/Clean_room_design) where I looked at the logic of the game and then devised my own strategies to "copy" the behaviour of the game. While it was an quite an accomplishment, it suffered from a fundamental design problem. Each block in the 3D space was an independant instance of a "class". This worked fine for small schematics, of say 30x30x80, but when one made really big schematics, memory usage became a signifcant problem. If one tried opening a somewhat large schematic, let's say 256x256x128, 32-bit Chrome would allocate about 800MB, and then the tab would crash. Another issue was that getting the behaviour to be *exactly* like the game was near impossible.

With "Version 2" (the current live version), I decided to decompile the game's source code and try copy the logic as closely as possible. Like the game, my newer version used the [Flyweight pattern](http://en.wikipedia.org/wiki/Flyweight_pattern) which solved the memory problems of "Version 1" and as the behaviour was literally a port of the game's decompiled Java code into JavaScript, the behaviour was exact and the result was a perfect simulation of the game's behaviour in a web browser. The problem I found though was that porting the code was quite time consuming, meaning that adding new block types was exceptionally time consuming. To make matters worse, a new version of Minecraft come out, had making my simulator behaviour no longer up to date, I also realized that to ensure I hadn't missed out any changes in the new version, I would have to re-port everything, a prospect which was very demoralizing.

So, for "Version 3" I decided that the only way to try keep up with the changes in the game in a timely manner, would be to automate the porting process as much as possible. Towards this end, I landed up writing a "good enough" transpiler which parses select parts of the game's decompiled Java code and emits equivalent TypeScript code. Then it was a matter of writing integration between the game's ported logic and my simulator. I also landed up porting all the JavaScript to TypeScript.

A few days ago, after about 18 months of work, I finally experienced fruition of my labour, I got a [simple schematic](http://mordritch.com/mc_rss/1001) simulation working.

The version of the game I had been transpiling from was version 1.5, I hadn't updated in a long time because I wanted to avoid aiming for a moving target, but with it working now, before I did anymore work, it would be best to move to the latest available decompiled source of the game, version 1.7.10.

<a name="main"></a>The need to Transpile Java Anonymous and Nested Classes to TypeScript
---

As a C# guy who has done no Java development, nested classes were immediately recognizable, but Anonymous classes looked very weird, I didn't even know what they were called. Fortunately, a little searching on the web quickly enlightened me on their name and how they work.

Nested (or inner) classes are simple enough to understand and look like this:

{% highlight ts %}
class OuterClass {
	...
	class NestedClass {
		...
	}
}
{% endhighlight %}

Anonymous classes are a little more complicated:

{% highlight java %}
class OuterClass {
	public OuterClass.NestedClass nestedClassInstance = new OuterClass.NestedClass();
	public OuterClass.NestedClass anonymousClassInstance = new OuterClass.NestedClass() {
		public void SomeMethod() {
			// Does something different from usual
		}
	};
	
	public class NestedClass {
		public void SomeMethod() {
			// Does something
		}
	}
}
{% endhighlight %}

However, it's easy enough to understand that anonymous classes are inheriting from the nested class and overriding members.

So, now to devise a strategy of how best to represent these concepts in TypeScript. In JavaScript, this can definitely be achieved, by doing something like:

{% highlight ts %}
// A helper function allowing one to "Extend" classes:
var classExtender = function (_child, _parent) {
	for (var p in _parent) {
		if (_parent.hasOwnProperty(p)) {
			_child[p] = _parent[p];
		}
	}
	
	function __() {
		this.constructor = _child;
	}
	__.prototype = _parent.prototype;
	_child.prototype = new __();
};

var OuterClass = (function() {
	function OuterClass() {
		this.nestedClassInstance = new OuterClass.NestedClass();
		
		// Anonymous class and it's instatiation:
		this.anonymousClassInstance = new ((function(_super) {
			classExtender(anonymousClass, _super);
			
			function anonymousClass() {
				_super.apply(this, arguments);
			}
			
			anonymousClass.prototype.SomeMethod = function() {
				console.log("Does something different from usual");
			};
			
			return anonymousClass;
		})(OuterClass.NestedClass))();
	}
	
	// Nested class definition
	OuterClass.NestedClass = (function() {
		function NestedClass() {
		}
		
		NestedClass.prototype.SomeMethod = function() {
			console.log("Does something");
		};
		
		return NestedClass;
	})();
	
	return OuterClass;
})();

var outerClass = new OuterClass();
outerClass.nestedClassInstance.SomeMethod(); // Prints "Does something"
outerClass.anonymousClassInstance.SomeMethod(); // Prints "Does something different from usual"
{% endhighlight %}

I used the same patterns as TypeScript for class inheritance, if you want to understand better, check the [Simple Inheritance example on the TypeScript Playground](http://www.typescriptlang.org/Playground#src=class%20Animal%20%7B%0A%20%20%20%20constructor(public%20name%3A%20string)%20%7B%20%7D%0A%20%20%20%20move(meters%3A%20number)%20%7B%0A%20%20%20%20%20%20%20%20alert(this.name%20%2B%20%22%20moved%20%22%20%2B%20meters%20%2B%20%22m.%22)%3B%0A%20%20%20%20%7D%0A%7D%0A%0Aclass%20Snake%20extends%20Animal%20%7B%0A%20%20%20%20constructor(name%3A%20string)%20%7B%20super(name)%3B%20%7D%0A%20%20%20%20move()%20%7B%0A%20%20%20%20%20%20%20%20alert(%22Slithering...%22)%3B%0A%20%20%20%20%20%20%20%20super.move(5)%3B%0A%20%20%20%20%7D%0A%7D%0A%0Aclass%20Horse%20extends%20Animal%20%7B%0A%20%20%20%20constructor(name%3A%20string)%20%7B%20super(name)%3B%20%7D%0A%20%20%20%20move()%20%7B%0A%20%20%20%20%20%20%20%20alert(%22Galloping...%22)%3B%0A%20%20%20%20%20%20%20%20super.move(45)%3B%0A%20%20%20%20%7D%0A%7D%0A%0Avar%20sam%20%3D%20new%20Snake(%22Sammy%20the%20Python%22)%3B%0Avar%20tom%3A%20Animal%20%3D%20new%20Horse(%22Tommy%20the%20Palomino%22)%3B%0A%0Asam.move()%3B%0Atom.move(34)%3B%0A).

Okay, so it's *possible*, but ideally we want to be able to do it in a more readable TypeScript syntax. Some quick research shows that it's at least [possible to nest classes](http://stackoverflow.com/a/21920276/706555). But I haven't found a way to do anonymous classes in plain TypeScript, which leaves us with code like this:

{% highlight ts %}
// A helper function allowing one to "Extend" classes:
var classExtender = function (_child, _parent) {
	for (var p in _parent) {
		if (_parent.hasOwnProperty(p)) {
			_child[p] = _parent[p];
		}
	}
	
	function __() {
		this.constructor = _child;
	}
	__.prototype = _parent.prototype;
	_child.prototype = new __();
};

class OuterClass {
	public nestedClassInstance: OuterClass.NestedClass;
	public anonymousClassInstance: OuterClass.NestedClass;
	
	constructor() {
		this.nestedClassInstance = new OuterClass.NestedClass();
		
		this.anonymousClassInstance = new ((function(_super): { new(...args: any[]): OuterClass.NestedClass } {
			classExtender(anonymousClass, _super);
			function anonymousClass() {
				_super(this, arguments);
			}
			
			anonymousClass.prototype.SomeMethod = function() {
				console.log("Does something different from usual");
			};
			
			return <any>anonymousClass;
		})(OuterClass.NestedClass))();
	}
}

module OuterClass {
	export class NestedClass {
		public SomeMethod(): void {
			console.log("Does something");
		}
	}
}

var outerClass = new OuterClass();
outerClass.nestedClassInstance.SomeMethod(); // Prints "Does something"
outerClass.anonymousClassInstance.SomeMethod(); // Prints "Does something different from usual"
{% endhighlight %}

The above works, the fact that we can do Nested Classes quite neatly in TypeScript clearly helps, but unfortunately I can't think of a better way to handle the anonymous classes. I'll probably make it a little neater by moving the extender into a separate source file, or possibly use the one that TypeScript generates.

So now that I have a pattern I can generate in TypeScript, it's now just a matter of implementing it in the transpiler.