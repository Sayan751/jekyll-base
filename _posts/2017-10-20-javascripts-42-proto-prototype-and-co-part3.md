---
layout: post
title:  "JavaScript's 42: proto, prototype & Co. (Part III)"
date:   2017-10-20 10:30:00+0100
categories: javascript
postid: "javascripts_42_proto_prototype_and_co_part3"
tags: [typescript, javascript, proto, prototype, inheritance]
---

This is the third post on inheritance in JavaScript.
Where in the [first]({{ site.baseurl }}{% post_url 2017-09-09-javascripts-42-proto-prototype-and-co %}), and [second post]({{ site.baseurl }}{% post_url 2017-09-16-javascripts-42-proto-prototype-and-co-part2 %}) was about `proto`, `prototype` and `Object.create` (in JavaScript), this post focusses on the inheritance in TypeScript.
More specifically, this post presents the details about what is going on *behind the scene* with inheritance in TypeScript.

**Before you start:** If you are not sure about functionalities of `proto`, `prototype` and `Object.create`, then you must (re)visit these concepts. You may refer my earlier blog post (as said above) to brush up these concepts.

As you may already know that TypeScript codes are finally gets compiled into plain JavaScript code. Thus, this post focusses on how the transformed javascript looks like.

If you have gone through my earlier blog posts in this series then you know that to use inheritance in plain JavaScript, we need to do quite a lot of stuffs.
With TypeScript the things get easier, as here you just have to use the `extends` keyword, where the magic happens. Example:

{% highlight typescript %}
export class Animal {
}

export class Bird extends Animal {
}
{% endhighlight %}

And that's all you need to do in order to use inheritance in TypeScript. Magic right?

Well, not quite.
If we check the JavaScript code generated by the TypeScript compiler, we will see the same rigorous implementation for inheritance in JavaScript.
When I compile this TypeScript code, the following JavaScript is generated. Depending on your version of TypeScript (mine is `2.5.3` at the time of writing this post) you may have a slightly different version.

{% highlight javascript %}
"use strict";
var __extends = (this && this.__extends) || (function () {
    var extendStatics = Object.setPrototypeOf ||
        ({ __proto__: [] } instanceof Array && function (d, b) { d.__proto__ = b; }) ||
        function (d, b) { for (var p in b) if (b.hasOwnProperty(p)) d[p] = b[p]; };
    return function (d, b) {
        extendStatics(d, b);
        function __() { this.constructor = d; }
        d.prototype = b === null ? Object.create(b) : (__.prototype = b.prototype, new __());
    };
})();
exports.__esModule = true;
var Animal = /** @class */ (function () {
    function Animal() {
    }
    return Animal;
}());
exports.Animal = Animal;
var Bird = /** @class */ (function (_super) {
    __extends(Bird, _super);
    function Bird() {
        return _super !== null && _super.apply(this, arguments) || this;
    }
    return Bird;
}(Animal));
exports.Bird = Bird;
{% endhighlight %}

Though it may look too much crammed up alien code, they are plain JavaScript.
If you have gone through the blog posts in this series then you know that we can breakdown the task of implementing inheritance into two parts:

  1. Inheriting the instance-scoped properties (defined on `this`) and members (defined on `prototype`) of base class, and
  1. Inheriting the `static`s.

For the first part, we look at the `__extends` function, which is roughly defined as follows:

{% highlight javascript %}
function (d, b) {
  extendStatics(d, b);
  function __() { this.constructor = d; }
  d.prototype = b === null ? Object.create(b) : (__.prototype = b.prototype, new __());
}
{% endhighlight %}

Here `d`, and `b` stands for derived and base class respectively.
And then we have the regular drill.
In the first line of the function it copies the `static` members of base class by invoking `extendStatics`.
Then in the last two lines it assigns the `prototype` of base class to `d.prototype.__proto__`, which is needed to inherit the members defined on `prototype` of base class (I am sure you remember [the basics]({{ site.baseurl }}{% post_url 2017-09-09-javascripts-42-proto-prototype-and-co %}#basics)).

As we know that we are (sort of) replacing the `prototype` of the derived class, any member defined on `prototype` of derived class before doing the gymnastic of inheritance is lost (refer the [second post]({{ site.baseurl }}{% post_url 2017-09-16-javascripts-42-proto-prototype-and-co-part2 %})).
That's why the first thing done in the derived class (in our case `Bird`) is to call `__extends(Bird, _super);`.
Note that `_super` is supplied to the derived class by wrapping the code of derived class in a Immediately-Invoked Function Expression (IIFE), and passing `Animal` as value for `_super`. Simplified code chunk below:

{% highlight javascript %}
var Bird = (function (_super) {
    __extends(Bird, _super);  //<-- extend base/_super

    // other stuff, we want to avoid for now.
}(Animal));
{% endhighlight %}

Not so magical now is it? :stuck_out_tongue:

Next thing we need to check is what happened behind the call of `extendStatics(d, b)`, which inherits the static members of base class? Well, that part is also straightforward if we break up the definition of `extendStatics`, as it is done below.

{% highlight javascript %}
var extendStatics =

        // alternative#1
        Object.setPrototypeOf ||

        // alternative#2 and first fallback
        ({ __proto__: [] } instanceof Array && function (d, b) { d.__proto__ = b; }) ||

        // alternative#3 and final fallback (our good old inheritStatics)
        function (d, b) { for (var p in b) if (b.hasOwnProperty(p)) d[p] = b[p]; };

{% endhighlight %}

We see that here we have actually 3 options for the implementation. It checks if an alternative is available in the JavaScript engine; if it is then use it else fallback to next alternative. The last one is our [good old `inheritStatics`]({{ site.baseurl }}{% post_url 2017-09-09-javascripts-42-proto-prototype-and-co %}#accessing-static-members-of-base-class). So nothing is new about that.

The first and second alternatives are the new stuffs we have here. However, both of them does the same thing; i.e. to assign the base class to the `__proto__` of the derived class (you may consider to check the [polyfill of `Object.setPrototypeOf` on MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf#Polyfill) at this point). The second alternative additionally checks before succeeding if the property `__proto__` is supported by JavaScript engine or not (as it was used to be a [non-standard property](http://2ality.com/2015/09/proto-es6.html#proto-)). And it does that by assigning an empty array to `__proto__` of an object and then checking if the object is an `Array` or not. But the main juice of these two alternatives is this part: `d.__proto__ = b`. Below is an example to show this.

{% highlight javascript %}
function Animal() {}
function Bird(name) {}
Object.setPrototypeOf(Bird, Animal);
console.log(Bird.__proto__ === Animal); // true
{% endhighlight %}

And once you set `Animal` to `Bird`'s `__proto__`, whenever you want to access any static member from `Bird` JavaScript will climb the inheritance/prototypical ladder for you and fetch the member; i.e. `Animal`'s statics are now accessible from `Bird`.

And that's how `extends` in TypeScript translates to JavaScript.

That's it then? Well, before ending this post I want to point out one major difference between the first 2 alternatives and the last one. If we look closely then we see that the first 2 alternatives handles the problem of inheriting statics by setting the `__proto__` of derived, whereas the third alternative makes a copy of static members of base class. This makes a huge difference for simple value-properties, as making copies of those prevents pointing to the same value-property as copying makes those available directly to derived class, and thereby preventing the climb of prototypical ladder for those property if not explicitly done so. To demonstrate the difference we recall [the population example shown in the first post]({{ site.baseurl }}{% post_url 2017-09-09-javascripts-42-proto-prototype-and-co %}#accessing-static-members-of-base-class). A brief version of the example is shown below:

{% highlight javascript %}
function Animal() {
    Animal.population++;
}
Animal.population = 0;
Animal.printPopulation = function () {
    console.log("Animal population: " + Animal.population + ", Bird population: " + Bird.population + ", Fish population: " + Fish.population);
}

function Bird() {
    Animal.call(this);
    Bird.population++;
}

function Fish() {
    Animal.call(this);
    Fish.population++;
}

new Bird();
Animal.printPopulation();
new Bird();
Animal.printPopulation();
new Fish();
Animal.printPopulation();
{% endhighlight %}

Now if we define the `extendStatics` as the third fallback option then we get following results.

{% highlight shell %}
Animal population: 1, Bird population: 1, Fish population: 0
Animal population: 2, Bird population: 2, Fish population: 0
Animal population: 3, Bird population: 2, Fish population: 1
{% endhighlight %}

However, if we use `Object.setPrototypeOf` to inherit the statics we will get a different result, as shown below.

{% highlight shell %}
Animal population: 1, Bird population: 2, Fish population: 1
Animal population: 2, Bird population: 3, Fish population: 2
Animal population: 3, Bird population: 3, Fish population: 4
{% endhighlight %}

The difference of the results are explained by the difference in the behavior of how these 2 alternatives works; i.e. `Object.setPrototypeOf` sets the `__proto__` of derived class to base class, whereas the other one merely makes copy and thus points to different instances.

Thus, to replicate the former behavior while using `Object.setPrototypeOf` we can simply define the `population` member on subclasses. Example:
{% highlight javascript %}
// in JavaScript
function Bird() {
   // omitting the rest for brevity
}
Bird.population = 0;

// in TypeScript
export class Bird extends Animal {
    static population = 0;
    // omitting the rest for brevity
}
{% endhighlight %}

If you are wondering if the same happens with TypeScript as well, then I invite you check it by yourself. However, you already know the answer, isn't it? And with that I will close this post.

Hope this helps.