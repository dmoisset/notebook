# My reading of PEP-622

After reading the recently proposed [PEP-622](https://www.python.org/dev/peps/pep-0622/) and the discussion that ensued in the [python-dev](https://mail.python.org/archives/list/python-dev@python.org/thread/RFW56R7LTSC3QSNIZPNZ26FZ3ZEUCZ3C/) mailing list I wanted to write some notes. Both to organise my own thoughts but also to share with other people discussing this topic.

## Is this actually about pattern matching?

After a couple of reads, even if this document is titled "structural pattern matching" and revolves a lot around a new matching statement, I don't think the match statement is the main idea in this PEP. The proposed statement itself is explicitly inspired in other languages that have pattern matching statements based on Algebraic Types (you can think of an Algebraic Type in CS terms as a disjoint union of a tagged tuple types, where each tuple element is of an algebraic type themselves; these are mathematician tuples, not python `tuple`). They've been a popular way of modelling data since the '70s (and possible earlier in for example LISP without being formalised). 

Why is this important?

My take on this PEP is that it's trying to introduce Algebraic Types into Python. I'm not saying that the PEP is trying to mislead us, only that it may be wrongly titled (or perhaps very correctly titled for a less formal audience).

In any data representation, there are some "core" operations that are essential and other operations can be built on them. For arrays is indexing (and possibly length); for dictionaries is setting/getting values at a given key, etc. For algebraic types, the core operation is destructuring. So having a way of pattern match is a side effect of having algebraic types. (That may answer some posters that were asking in python-dev *"why are we mixing up extracting elements, binding, and selection?"*... the answer is "because that's the way you access an algebraic type, and this PEP is about algebraic types")

## How do you add algebraic types to Python?

An important question and required discussion (that's not present on the PEP, but probably should be if it was titled "Algebraic Types for Python") is "how do we map the CS concept to the existing language?". 

The first approach could be define a separate type hierarchy for algebraic types without touching the existing types, so you have two separate worlds. Python has done that at least twice: namedtuples (which fail with some porperties of algebraic types but are a good approximation) and dataclasses. But that doesn't help you if your data is not stored in namedtuples or dataclasses. So the question the PEP tries to answer is "how we make *everything else* an algebraic type, or at least make it a duck that quacks like an algebraic type.

In CS, a value of an algebraic type consists of a symbolic identifier (sometimes called "tags" or "labels" or "constructors") and a typed tuple of values. The structure of the tuple is determined by the label (I'm not intending this article to be explanatory, but if you check the list example [in wikipedia](https://en.wikipedia.org/wiki/Algebraic_data_type), `Nil` and `Cons` are the labels, nil is associated to a 0-tuple, and `Cons` to a 2 tuple with a value and a list). Some people have mentioned a natural duality between label and type indicators in a polymorphic OO language. In fact dynamic languages like Python are sometimes considered by theorists to have a single main algebraic type (in the static sense) which is a union of values, each using the runtime class as its label. So this is actually a good fit. It also explains the reasoning by the authors: "we're doing a lot of `if ìsinstance()` calls" --> "we're actually selecting labels from algebraic types" --> "this code looks ugly because python doesn't have algebraic types and we're hand-compiling algebraic pattern matching each time". So there's a more or less obvious label, which is the `__class__` attribute of an object (it would be interesting to explore if somethign else could make sense)

The second part of mapping algebraic types to Python is how do we map the "tuple" part. CS classic theory uses a tuple, but for pragmatic reasons, many other languages allows adding names to elements. These are called records in Haskell for example and structs in Rust. In Python we'd normally use either a namedtuple, or a modern dict (order matters) with string keys. And actually if your Python class follows a algebraic data type, the arguments for its constructor should actually correspond with this structure. 

This derives in a relevant and non-obvious decision that is made in the PEP and non-explicit: python values are not obviously a dict. For most user defined classes, the `__dict__` attribute actually captures something very similar, but that may not be the abstract state of the value that you'd like to expose. There's also the question of important builtins, but there are some reasonably "standard" ways to map those to algebraic types, and the pep uses those for numeric types, bools, strings (somewhat) and sequences. But for most objects there's not a canonical way to see tham as a "struct/record/namedtuple".

The PEP decides to have a `__match__` protocol, where the `__match_args__` are the field names, and the values of the fields are pulled with the dot operator from whatever `__match__` returns. In other words, given a pattern `Matcher(foo, bar, baz)`, our mapping from python objects (with match protocol) to algebraic data type structs is in pseudocode:

```python
def algebraic_type_from_object(o):
    pre_structured = Matcher.__match__(o)
    struct = {field: getattr(pre_structured, field) for field in Matcher.__match_args__}
    return AlgebraicType(label=o.__class__, record=struct)
```

Side note: repeating my position that this is NOT about matching, I think that the names of the protocol methods may be terrible. Possibly `__struct__` would be a better name, given that the goal of this method is not matching anything but providing a structured version.

There are some unusual decisions which may be well thought, but I'm really curious about how the PEP ended up in this region of the design space. Things that caught my attention are:

* The obvious way (in the sense of first idea coming to mind, not necessarily the best) to get the structure of something, is return something that looks like a structure. i.e. having a `__structure__` method on an object that returns a namedtuple, or dict or similar. I can imagine that the authors may want to prevent allocating auxiliar objects during the matching process, but that's just a guess. If that's the case, returning by default an object `__dict__` or some sort of mapping view on the attributes could still be fine. It's not clear to me why the "keys" of this structure are placed apart.
* Something that surprises me (perhaps I've missed something) is that the job of determining the structure doesn't fall in the object, but in the matcher instead. For me there should be an instance method in `object` (that subclasses can override) that returnes the algebraic structure of the value. The PEP as is creates different destructuring views depending on which matching class you use (something that was mentioned but not discussed a lot in the python-dev list). The `__match__` method could remain in the matcher (i.e, possible a default implementation that just wraps `isinstance`)

# About syntax and clarity

Discussions about syntax are readibility are hard. Having an opinion and a taste on concrete syntax (this operator we're adding should be prefix or suffix? which actual characters do we use? which keyword) are extremely easy, but everybody has a different one, supported only by personal taste and futurology (which will be more clear? which will confuse less users). I generally prefer to defer on these discussions to people with a long history in the python core dev community because they've generally shown to be good at making good guesses for us. All they need for us is some feedback and ideas, but I won't be pushing for a decision on any specific syntax.

What I *can* do is do some general analysis that can put the alternatives in at least a framework we we can discuss facts that can drive the decision ("the keyword foo is new in this PEP" is a fact. "the keyword foo will be confusing for users" isn't).

## What's new

Most new language constructs include new syntax in some way or another, but let's see what's new here syntactically and how much it looks the same or different than other constructs

* There are new keywords introduced: `match` and `case`. Although "new keyword" always raises the risk of breaking backwards compatibility, the fact that these keywords are contextual thanks to the new python PEG-based parser (i.e. you still can have variables and functions called `match`) seem to have put at ease virtually everyone (I haven't seen significant discussions around this)
* The "pattern" is a new important syntactic family (comparable with "expressions" and "statements"), although it certainly has some visual similarity with both assignment targets and expressions (by visual I mean "the input strings are the same", even if their interpretations are not).
* the dot prefix in constant_patterns look unlike anything else in python syntax (except perhaps completely unrelated relative imports).
* The `_` is used as a special symbol (unlike the rest of Python where it's a name like any other).

The last two of these items have caused some contention.

## The pattern sub-language syntax

Patterns have a role where they have to look like a value that you can compare something with (similar to an expression) but also like something that's able to do binding to variables (like an assignment target). As a reminder, python assignments are `<target> = <expression>`, so consider here "target" anything that could go to the left of the equal sign (like `myvar`, `l[0]`, `f(x+1)["key"].attribute[1:]`) and expression anything that could go to the right (essentially any code denoting a value). This creates some natural ambiguity.

The ambiguity is likely less than people think about. In Python , targets and expressions already look similar (in fact, every piece of code denoting a target also denotes a valid expression), and that doesn't seem to have caused any kind of serious confusion. Patterns share some of the syntax of both; some of them are "target-like" (i.e. they would be syntactically valid as targets), and others are expression like. The following table shows a detailed breakdown:


|Pattern               |example       |target-like|expression-like|binds a variable|
|----------------------|--------------|:---------:|:-------------:|----------------|
| name_pattern         |`foo`         |✔️|✔️|✔️|
| literal_pattern      |`42`          |✖️|✔️|✖️|
| constant_pattern(A)  |`Color.BLACK` |✔️|✔️|✖️|
| constant_pattern(B)  |`.BLACK`      |✖️|✖️|✖️|
| sequence_pattern     |`[x,y]`       |✔️*|✔️*|✔️†|
| mapping_pattern      |`{"x": x}`    |✖️|✔️*|✔️†|
| class_pattern        |`Point(x,y)`  |✖️|✔️*|✔️†|

\* Only if the component sub-patterns are correspondingly expression-like/target-like
† If and only if a subpattern does

Looking at this table, a few things stand out:
1. As mentioned before, there's one form of `constant_pattern` that's completely new syntax, so that's bound to raise a few eyebrows. (I'm not saying it's bad, but definitely something that developers will have to get used to)
2. Everything else looks like an expression. But only half the things look like targets. This means that the branches of a `match` statement will look more like values than stuff you assign to. Of the 3 things that don't look like targets (`literal`, `mapping`, and `class`) 2 actually can definitely do binding. That could actually be alleviated by alowing them as targets (which is now a deferred idea).
3. Also some elements are ambiguous (they could be either expressions or targets. In some of those cases (name patterns like `foo`) they actually behave as targets(it's a name to be bound to, not evaluated), in others (like a constant expression `Color.black`) they behave as expressions (denoting a value).
4. Almost all valid combinations are present in the table above, which I believe feeds the sense of the syntax being "off" or not matching current Python practices.

Assuming that making this table more homogenous is a good thing (I think so, but that's 100% in the land of opinion, not fact), what could be done?

![butterfly meme - confusing patterns with targets](https://imgflip.com/46lnzc.jpg)

* Making more things to be target-like (by adding these variants to the target syntax, meaning that they would be allowed as an assignment target or looping variables in a `for` loop) certainly would make this table more homogeneity. This is actually mentioned as a deferred idea, and given that we already have some forms of destructuring assignment (being able to nest sequences), it would be surprising to have the mapping versions, and I'd be fine.
* Adjusting syntax so simple rules can be described (Python currently has "if it looks like an expression



Other stuff:
 - patterns as runtime objects?
 
 
 # Some conclusions
 
 I see that this PEPs try to introduce a few new things:
