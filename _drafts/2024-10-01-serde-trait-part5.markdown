---
title:  "Serde Trait - Part 5: Lifetimes"
layout: default
date:   2024-10-02 20:21:59 +0100
tags: Rust, serde
---
<h1>Serde for trait objects - Part 5: Lifetimes</h1>

Today, we'll take about lifetimes. This will be a long blog post, because we will discuss almost everything again, and now with lifetimes included.

In this series of blog posts I'm explaining how to use serde with trait objects:
- [Part 1: Overview]({% post_url 2024-10-01-serde-trait-part1 %})
- Part 2: Serialization
- Part 3: Deserialization
- Part 4: Registry
- Part 5: Lifetimes
- Part 6: Sync/Send
- Part 7: Macro Part A: Trait
- Part 8: Marco Part B: Implementation

Remark: If you are in a situation where you want to serialize a trait object, please take a step back.
Check if you can replace your trait object with an enum.
In my experience, the enum approach is much easier to work with.

Remark: All topics covered here are well-known. We follow [typetag](https://github.com/dtolnay/typetag).

So, let's start.


Given a trait `trait Trait { ... }`, there is the trait object type `dyn Trait`.
This type has an implicit `'static` lifetime bound.
To get a non-static trait object with lifetime `'t`, the syntax is `dyn Trait +'t`.
So, this blog post is basically: Do everything in the previous blog posts, and replace every instance of `dyn Trait` with `dyn Trait 

Before we do so, first a digression about teaching.

<h1>Teaching</h1>

Let's consider a beginner/intermediate/expert scale.
Up to now, everything was hopefully more or less standard, let's say intermediate level.
This means, that I assumed you, the reader, are familiar with <b>serde</b>, <b>Trait object</b>s, <b>Generics</b> and so on.
I was explaining how to use <b>erased-serde</b>, but never showed a <b>cargo.toml</b> adding a dependency.
Moreover, I assumed that you, the reader, can find examples yourself.
Hence I would label this series as intermediate level, but not as beginner friendly.

But today we will talk about lifetimes, which is itself an advanced topic.
I believe that trait objects and lifetimes together are a rare topic (because it is a rare topic for me).
Hence I will include more examples than in the previous.

Upshot: I believe that today's entry in this series is more advanced than the previous entries.
Also note that this entry is independent of the other entries.

<h1>Serialization</h1>

Serialization is quite easy, we just add the tag as expected.

But before doing so, let's review our code from part 2:
{% highlight rust %}
trait Trait: erased_serde::Serialize {
    fn message(&self) -> String;

    // Will be generate by macro    
    fn type_info(&self) -> &'static str;
}

#[derive(serde::Serialize)]
struct S {
    data: i32,
}
impl Trait for S {
    fn message(&self) -> String {
        format!("Message: {}", self.data)
    }

    // Will be generate by macro    
    fn type_info(&self) -> &'static str {
        "S"
    }
}

struct Wrap<'a, T: ?Sized>(pub &'a T);
impl<'a, T> serde::Serialize for Wrap<'a, T>
where
    T: ?Sized + erased_serde::Serialize + 'a,
{
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        erased_serde::serialize(self.0, serializer)
    }
}

impl serde::Serialize for dyn Trait {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        use serde::ser::SerializeMap;
        let mut ser = serializer.serialize_map(Some(1))?;
        let type_info = self.type_info();        
        ser.serialize_entry(type_info, &Wrap(self))?;
        ser.end()
    }
}

fn main() {
    let s = S { data: 0 };
    let t: &dyn Trait = &s;
    let ser = serde_json::to_string(t).unwrap();
    println!("Serialized json: {}", ser);
}
{% endhighlight %}
Note that neither the trait nor the type has a lifetime.

Let's add one to the type:
{% highlight rust %}
#[derive(serde::Serialize)]
struct S<'s> {
    data: &'s str,
}
{% endhighlight %}
and adjust main as well:
{% highlight rust %}
fn main() {
    let text = String::from("this String is not a 'static str!");
    let s = S { data: &text };
    let t: &dyn Trait = &s;
    let ser = serde_json::to_string(t).unwrap();
    println!("Serialized json: {}", ser);
}
{% endhighlight %}
which does not compile:
{% highlight shell%}
error[E0597]: `text` does not live long enough
  --> blog/src/main.rs:52:23   
51 |     let text = String::from("this String is not a 'static str!");
   |         ---- binding `text` declared here
52 |     let s = S { data: &text };
   |                       ^^^^^ borrowed value does not live long enough
53 |     let t: &dyn Trait = &s;
   |                         -- cast requires that `text` is borrowed for `'static`
...
56 | }
   | - `text` dropped here while still borrowed
}
{% endhighlight %}
If we replace
{% highlight rust %}
let text = String::from("this String is not a 'static str!");
{% endhighlight %}
with
{% highlight rust %}
let text = "this String is a 'static str!";
{% endhighlight %}

Remark: If you want to check that a value is `'static`, you can use:
{% highlight rust %}
fn assert_static<A: 'static>(_a: A) {}
{% endhighlight %}
If you apply this function to a value and your code compiles, the value is static.
If the value is not static, your code does not compile.

But for serialization, the lifetime is NOT important. 
If you have a value which is valid to use (hence is "alive"), you should be able to serialize it.

So, let's correct this. To do so, we replace:
{% highlight rust %}
impl serde::Serialize for dyn Trait {
{% endhighlight %}
with
{% highlight rust %}
impl<'t> serde::Serialize for dyn Trait+'t {
{% endhighlight %}
That's it. Now the example with the non-static lifetime compiles.

Our implementation of `serde::Serialize` was already fine, there just was the implicit lifetime bound. 
After explicitly opting out of it, we get what we want.

<h1>Deserialization</h1>

We should take a step back and think about deserialization.

But, let's do this later.

For now, we add lifetimes to the deserialization part.

Here is the final code from part 3:

{% highlight rust %}
trait Trait {}

#[derive(serde::Deserialize)]
struct S {
    data: i32,
}
impl Trait for S {}

impl<'de> serde::Deserialize<'de> for Box<dyn Trait> {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        let visitor = HelperVisitor {};
        deserializer.deserialize_map(visitor)
    }
}

type DeserializeFn =
    fn(&mut dyn erased_serde::Deserializer) -> erased_serde::Result<Box<dyn Trait>>;
fn runtime_reflection(type_info: &str) -> Option<DeserializeFn> {
    fn deserialize_fn_generic<A>(
        deserializer: &mut dyn erased_serde::Deserializer,
    ) -> erased_serde::Result<Box<dyn Trait>>
    where
        A: Trait,
        A: serde::de::DeserializeOwned,
        A: 'static,
    {
        let a: A = erased_serde::deserialize(deserializer)?;
        let boxed_trait_object: Box<dyn Trait> = Box::new(a);
        Ok(boxed_trait_object)
    }
    if type_info == "S" {
        Some(deserialize_fn_generic::<S>)
    } else {
        None
    }
}

struct TypeVisitor {
    deserialize_fn: DeserializeFn,
}
impl<'de> serde::de::DeserializeSeed<'de> for TypeVisitor {
    type Value = Box<dyn Trait>;

    fn deserialize<D>(self, deserializer: D) -> Result<Self::Value, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        let mut erased = <dyn erased_serde::Deserializer>::erase(deserializer);
        let deserialize_fn = self.deserialize_fn;
        deserialize_fn(&mut erased).map_err(|e| serde::de::Error::custom(e))
    }
}

struct HelperVisitor {}
impl<'de> serde::de::Visitor<'de> for HelperVisitor {
    type Value = Box<dyn Trait>;

    fn expecting(&self, formatter: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(formatter, "Trait object 'dyn Trait'")
    }

    fn visit_map<A>(self, mut map: A) -> Result<Self::Value, A::Error>
    where
        A: serde::de::MapAccess<'de>,
    {
        let type_info = map.next_key::<String>()?.ok_or(serde::de::Error::custom(
            "Expected externally tagged 'dyn Trait'",
        ))?;
        let deserialize_fn = runtime_reflection(&type_info).ok_or(serde::de::Error::custom(
            format!("Unknown type for 'dyn Trait': {type_info}"),
        ))?;
        let boxed_trait_object = map.next_value_seed(TypeVisitor { deserialize_fn })?;
        Ok(boxed_trait_object)
    }
}

fn main() {
    let json = r#"{"S":{"data":0}}"#;
    let t: Box<dyn Trait> = serde_json::from_str(json).unwrap();
}
{% endhighlight %}
Again, our aim is not to add a lifetime parameter to the trait, like `Trait<'t>`, but instead replacing all instances of `dyn Trait` with `dyn Trait + 't`.

We start with the implementation of `serde::Deserialize`:
{% highlight rust %}
impl<'de> serde::Deserialize<'de> for Box<dyn Trait> { ... }
{% endhighlight %}
to
{% highlight rust %}
impl<'de> serde::Deserialize<'de> for Box<dyn Trait + 't> { ... }
{% endhighlight %}
which makes no sense:
{% highlight text %}
9 | impl<'de> serde::Deserialize<'de> for Box<dyn Trait + 't> {
  |      -                                                ^^ undeclared lifetime
  |      |
  |      help: consider introducing lifetime `'t` here: `'t,`
{% endhighlight %}
The only lifetime in scope is `'de`, so let's use this one (as mentioned before, this will be discussed later).
{% highlight rust %}
impl<'de> serde::Deserialize<'de> for Box<dyn Trait + 'de> { ... }
{% endhighlight %}
This leads to a compile error. The root is here:
{% highlight rust %}
type DeserializeFn = fn(&mut dyn erased_serde::Deserializer) -> 
    erased_serde::Result<Box<dyn Trait>>;
{% endhighlight %}
Following our general philosophy, we replace `dyn Trait` with `dyn Trait + 't`:
{% highlight rust %}
type DeserializeFn = fn(&mut dyn erased_serde::Deserializer) -> 
    erased_serde::Result<Box<dyn Trait + 't>>;
{% endhighlight %}
Again, we have an unbound lifetime on the right hand.
Instead of adding it on the right hand side, we use a [Higher-Rank Trait Bounds (HRTB)](https://doc.rust-lang.org/nomicon/hrtb.html):
{% highlight rust %}
type DeserializeFn = for <'t> fn(&mut dyn erased_serde::Deserializer) -> 
    erased_serde::Result<Box<dyn Trait + 't>>;
{% endhighlight %}
This does not compile, but makes no sense:
{% highlight text %}
error[E0581]: return type references lifetime `'t`, which is not constrained by the fn input types
  --> blog/src/main.rs:27:57
   |
27 |     for<'t> fn(&mut dyn erased_serde::Deserializer) -> erased_serde::Result<Box<dyn Trait + 't>>;
   |   
{% endhighlight %}

We basically say: For any input value of type `&mut dyn erased_serde::Deserializer`, the function produces a result with OK type `Box<dyn Trait + 't>` for all lifetimes `'t`. In particular, for the lifetime `'static`. So, our HRTB changed nothing, and the compiler noted this even.

The critical point is that there are two implicit lifetimes: One for the borrow, one for the deserializer:
{% highlight rust %}
type DeserializeFn = fn(&'b mut dyn erased_serde::Deserializer<'de>) -> 
    erased_serde::Result<Box<dyn Trait + 't>>;
{% endhighlight %}
So, we can couple `'t` to either `'de`, `'b` or both.

The correct one is `'de` (this will be discussed later). So we end up with
{% highlight rust %}
type DeserializeFn = for <'de> fn(&mut dyn erased_serde::Deserializer<'de>) -> 
    erased_serde::Result<Box<dyn Trait + 'de>>;
{% endhighlight %}
Let's make this down-to-earth:

For any input value of type `&mut dyn erased_serde::Deserializer<'de>`, the function produces a result with OK type `Box<dyn Trait + 'de>`, and this for all lifetimes `'de`. If our deserializer is valid for the lifetime `'static`, so will be our output values. 
But if the deserializer is only valid for a shorter lifetime, so is our output value. (More on this later.)

Next, we do the obvious adjustments, replacing `dyn Trait` with `dyn Trait + 't` in the deserialization helpers:
{% highlight rust %}
impl<'de> serde::de::DeserializeSeed<'de> for TypeVisitor {
    type Value = Box<dyn Trait>;
    ...
}
impl<'de> serde::de::Visitor<'de> for HelperVisitor {
    type Value = Box<dyn Trait>;
    ...
}
{% endhighlight %}
Then we proceed with the generic deserialization function. From
{% highlight rust %}
fn deserialize_fn_generic<A>(
    deserializer: &mut dyn erased_serde::Deserializer,
) -> erased_serde::Result<Box<dyn Trait>>
where
    A: Trait,
    A: serde::de::DeserializeOwned,
    A: 'static,
{
    let a: A = erased_serde::deserialize(deserializer)?;
    let boxed_trait_object: Box<dyn Trait> = Box::new(a);
    Ok(boxed_trait_object)
}
{% endhighlight %}
to
{% highlight rust %}
fn deserialize_fn_generic<'de, A>(
    deserializer: &mut dyn erased_serde::Deserializer<'de>,
) -> erased_serde::Result<Box<dyn Trait + 'de>>
where
    A: Trait + 'de,
    A: serde::de::Deserialize<'de>,
    A: 'de,
{
    let a: A = erased_serde::deserialize(deserializer)?;
    let boxed_trait_object: Box<dyn Trait + 'de> = Box::new(a);
    Ok(boxed_trait_object)
}
{% endhighlight %}
This was a mechanical change: "Add the lifetime to all possible places, except the borrow."

We're almost done with this part. But there is one more compiler error:
{% highlight rust %}
if type_info == "S" {
    Some(deserialize_fn_generic::<S>)
} else {
    None
}
{% endhighlight %}
fails to compile with the following error message:
{% highlight text %}
error[E0308]: mismatched types
   --> blog/src/main_de.rs:43:14
    |
43  |         Some(deserialize_fn_generic::<S>)
    |         ---- ^^^^^^^^^^^^^^^^^^^^^^^^^^^ one type is more general than the other
    |         |
    |         arguments to this enum variant are incorrect
    |
    = note: expected fn pointer `for<'a, 'de> fn(&'a mut (dyn erased_serde::Deserializer<'de> + 'a)) -> Result<Box<(dyn main_de::Trait + 'de)>, _>`
                  found fn item `for<'a> fn(&'a mut (dyn erased_serde::Deserializer<'_> + 'a)) -> Result<Box<dyn main_de::Trait>, _> {deserialize_fn_generic::<'_, main_de::S>}`
help: the type constructed contains `for<'a> fn(&'a mut (dyn erased_serde::Deserializer<'_> + 'a)) -> Result<Box<dyn main_de::Trait>, erased_serde::Error> {deserialize_fn_generic::<'_, main_de::S>}` due to the type of the argument passed
   --> blog/src/main_de.rs:43:9
{% endhighlight %}
This one I couldn't solve on my own, but after asking on [URLO](https://users.rust-lang.org/), I got not a single answer in the first hour, but two!
Thank you again to quinedot and alice. The rust forums are really, really great.

It turns out that a closure helps the compiler to understand our intention:
{% highlight rust %}
// failing: Some(deserialize_fn_generic::<S>)
Some(|input| deserialize_fn_generic::<S>(input)) // this works
{% endhighlight %}

Now we are done with this part.

<h1>Example: Deserialization</h1>
To check this, let's add a lifetime to our type S (and not to the trait!):
{% highlight rust %}
struct S {
    data: i32,
}
impl Trait for S {}
{% endhighlight %}
is changed to
{% highlight rust %}
struct S<'s> {
    data: &'s str,
}
impl<'s> Trait for S<'s> {}
{% endhighlight %}
And in the main function:
{% highlight rust %}
let json = r#"{"S":{"data":0}}"#;
{% endhighlight %}
is changed to
{% highlight rust %}
let json = r#"{"S":{"data":"bla"}}"#;
{% endhighlight %}
Then the code compiles and executes without panic.

But, our trait object is `'static`. To proof this, we can use our `check_static` function like this:
{% highlight rust %}
fn main() {
    let json = r#"{"S":{"data":"bla"}}"#;
    let t: Box<dyn Trait> = serde_json::from_str(json).unwrap();
    check_static(&t);
}
fn check_static<A: 'static>(_a: &A) {}
{% endhighlight %}
To get a shorter lifetime, we use a string instead of `&'static str':
{% highlight rust %}
fn main() {
    let json = String::from(r#"{"S":{"data":"bla"}}"#);    
    let t: Box<dyn Trait> = serde_json::from_str(&json).unwrap();
    check_static(&t);
}
fn check_static<A: 'static>(_a: &A) {}
{% endhighlight %}
which fails to compile because of a non-static lifetime. If we remove `check_static`, it works again.

<h3>String References, Json and Cows</h3>

Remark: This example is stupid, because using json and `&str` is dangerous. See the following example.
{% highlight rust %}
#[derive(serde::Serialize, serde::Deserialize, Debug)]
struct S<'s>(&'s str);
fn main() {
    let s = dbg!(S("String with escaped\" json"));
    let ser = dbg!(serde_json::to_string(&s).unwrap());
    let _s: S = serde_json::from_str(&ser).unwrap();
}
{% endhighlight %}
This compiles, but executes with a runtime panic:
{% highlight shell %}
[blog/src/main.rs:4:13] S("String with escaped\" json") = S(
    "String with escaped\" json",
)
[blog/src/main.rs:5:15] serde_json::to_string(&s).unwrap() = "\"String with escaped\\\" json\""
thread 'main' panicked at blog/src/main.rs:6:44:
called `Result::unwrap()` on an `Err` value: Error("invalid type: string \"String with escaped\\\" json\", expected a borrowed string", line: 1, column: 28)
{% endhighlight %}
The point is: Inside a json string, the quotation mark character '\"' has to be escaped (because other it would end the json string).
Hence, during deserialization, serde-json has to convert escaped characters to the original ones (by removing backslashes).
Hence, the byte buffer of the input string cannot be borrowed, but needs to be changed.
To solve issues likes this, <b>Rust</b> contains a type called [Cow (Clone-on-write)](https://doc.rust-lang.org/std/borrow/enum.Cow.html).
{% highlight rust %}
#[derive(serde::Serialize, serde::Deserialize, Debug)]
struct S<'s>(#[serde(borrow)] std::borrow::Cow<'s, str>);
fn main() {
    let text = "String with escaped\" json".to_string();
    let s = dbg!(S((&text).into()));
    match &s.0 {
        std::borrow::Cow::Borrowed(_) => println!("borrowed"),
        std::borrow::Cow::Owned(_) => println!("owned"),
    }
    let ser = dbg!(serde_json::to_string(&s).unwrap());
    let s: S = dbg!(serde_json::from_str(&ser).unwrap());
    match &s.0 {
        std::borrow::Cow::Borrowed(_) => println!("borrowed"),
        std::borrow::Cow::Owned(_) => println!("owned"),
    }
    //check_static(&s);
}
fn check_static<A: 'static>(_a: &A) {}
{% endhighlight %}
In this example, the initial Cow is borrowed and the deserialized one is owned.
If we remove the inner quotation mark, both Cows are borrowed.
In either case, the deserialized value has a non-static lifetime.

<h1>Registry</h1>

We continue our lifetime quest with discussion the registry from part 4.

Recall that we define the registry type as follows:
{% highlight rust %}
type DeserializeFn =
    fn(&mut dyn erased_serde::Deserializer) -> erased_serde::Result<Box<dyn Trait>>;
pub struct Registry {
    data: std::sync::Mutex<std::collections::BTreeMap<String, DeserializeFn>>,
}
{% endhighlight %}
If we use our lifetime-adapted definition of `DeserializeFn`
{% highlight rust %}
type DeserializeFn =
    for<'de> fn(&mut dyn erased_serde::Deserializer<'de>) -> erased_serde::Result<Box<dyn Trait + 'de>>;
{% endhighlight %}
the type `Registry` is unchanged as well as the static Registry `REGISTRY_TRAIT` and the runtime reflection:
{% highlight rust %}
pub static REGISTRY_TRAIT: Registry = Registry::new();
fn runtime_reflection(type_info: &str) -> Option<DeserializeFn> {
    REGISTRY_TRAIT.get(type_info)
}
{% endhighlight %}
To complete our example, we 


<h1>High-Level Discussion</h1>


ToDo: Two different worlds.



<h3>Footnotes</h3>
[^FootnoteRant]: (Rant) This series is going to be my reference answer to the question: "I'm struggling with trait objects. How do I solve problem XYZ?" So that I can say: "I suggest using an enum instead of a trait object. This is often much easier. For example, if you want to use serde for you trait object, you need to work through all of the following."
[^FootnoteDotnetNewtonsoft]: I'm aware that this snippet is using Newtonsoft.Json instead of the System.Text.Json, but this is a topic for another day …
[^FootnoteJokeEnumsAvoidTraitObjects]: (Joke) Note that `enum`s avoid trait objects, hence you should also avoid trait objects ;-)
