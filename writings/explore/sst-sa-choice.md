SST – Static Strict Typing
SA – Static Analysis

We’ve already agreed to use SA and SST; what exactly are we discussing?

SA or SST?

Is program correctness a low priority, or merely an option?

Like many others, we deploy our PHP code in a packaged, immutable format.

## Static Strict Typing

Pros of SST:
- Increases code locality by orders of magnitude.
- Provides much clearer signals regarding the purpose of the code.
- When applied correctly, it helps reduce logical errors.
- Facilitates Static Analysis (SA).
- Encourages simpler code.
- Enhances code transferability (makes it easier for others to work with).

Cons of SST:
- “Difficult” – everything requires learning, but learning SST once is easier than having to constantly decipher new, dynamic code.
- “Time-consuming” – adding type annotations takes less time than debugging or discovering bugs later on.
- “Poor support in legacy code” – this is a mischaracterization; in reality, it’s the legacy code and libraries that lack support for more disciplined coding practices.
- “Poor support for dynamic code” – this is, by definition, exactly what SST is designed to counter, as “dynamic code” creates more long-term problems. ...but static type checking (STC) doesn’t forbid it entirely either.
- “A false sense of security” — static typing cannot protect against every type of logical error.

Facts about STC:
- Many of the company’s projects already use STC.
- Many large-scale PHP projects use STC.
- New STC-related features are added to PHP itself with every release.
- Many major projects in other programming languages ​​use STC (implicitly).
- Many large projects in dynamically typed languages ​​rely on automated tests and assertions—we have neither.

## Static Analysis (SA)

Pros of SA:
- Think of it as a compiler for the language.
- Without it, STC effectively “doesn’t work.”
- Beyond basic errors, SA can detect other classes of issues or potential problem areas; this set can even be customized for a specific project (e.g., identifying dead code or performing taint analysis).
- It enables the detection of systematic errors.
- This cannot be achieved through testing or QA alone.
- It helps prevent regressions—which is especially critical given the lack of automated tests.
- It encourages simpler code.
- In particularly complex scenarios, specific exceptions to the analysis rules can be defined.

Cons of SA:
- “It’s difficult” — there is a learning curve, but ultimately, it reduces the overall complexity of a project.
- “It’s time-consuming” — not nearly as time-consuming as discovering bugs late in the development cycle.
- “Poor support for legacy code” — this is a mischaracterization; in reality, it’s the legacy code and libraries themselves that lack support for more rigorous coding practices.
- “A false sense of security” — this can exist even without static analysis, though in that case, it is far less justified.
- “False positives/false negatives” — a common characteristic of all static analysis tools. Vendor-provided or third-party analysis tools may not be flawless, but they are still better than having no analysis at all; furthermore, well-justified exceptions can be made for specific cases.

Facts about Static Analysis (SA):
- A simplified version of SA, with minimal functionality, is already supported and in use within the IDE.
