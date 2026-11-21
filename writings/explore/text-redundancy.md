I really think text is mostly redundant for programs.
Programs are about composing primitives of automation.
Modern complex programs use up from tens of DSL per file - easily. But all of them are mashed as text.
Automation is about solving problems - not about arranging text.
I need a programming "language" or "environment" or "tooling" that does away with text.
All primitives are interlinked as primitives first, text second. This makes changing names trivial and allows having repeated names where it doesn't matter. It does away with problems like shadowing or scope since nothing relies on names under the hood.
It simplifies type systems.
It makes formatting, styling, file structure etc. irrelevant.
It makes code merges trivial.
It even trivialises versioning because you depend on composition of primitives, not on text.
It also trivialises reflection since all of the data is already there and "parsed", just with some minor extra metadata for humans.
It can even make programming faster, because instead of typing and intellisensing against text, you know exactly what **can** be at each place.
It could even be in theory language-agnostic.

E.g. looking at PHP since it is one I am quite familiar with.

A property can have undefined, private, protected, or public visibility. But the "slot" for that data can be also filled with gibberish, or forgotten to be filled. In a new paradigm that is impossible - a state is there, it is always displayed. In that paradigm, you navigate on keyboard much faster - just push left left from function name (for example), then whatever control key + up or down to select the mode. Invalid text is impossible to even enter.

Need a new function? Navigate between visible declarations and press whatever control key and "+" - select a type of declaration with arrows, then type a metadata name (which before was THE name) for a primitive - it is already pre-filled with everything. It is not a visual point and click constructor first, but ergonomics first - but text is no longer the "source", it is just presentation.

Want a better arrangement of code? Press a key and sort code in any style you prefer or makes sense for the task. Need to track anything? Press a key and see a graph - a real, complete graph of dependencies of everything onto everything.

It has to be able to live up to modern expectation of mixing and matching multiple languages or DSL per "zone" of work with effortless hints, intellect, and navigation - which modern IDEs can do with text, but with all caveats of text. So it cannot be built around its own "one true language to rule them all".

Anything less than all of these properties at once is a **downgrade** and is **useless**.
