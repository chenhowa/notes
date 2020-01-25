# Parsing

## Beyond Regular Languages

Many languages are not regular languages, so they cannot be analyzed through Regular Expressions and Finite Automata. In particular, nested constructs usually cannot be expressed with regular languages. Intuitively, you can see that regular expressions cannot keep track of how many times they've been in a particular state. Therefore they cannot verify things like matching parentheses.

## Context-free Grammars

A natural notation for describing recursive languages.

A CFG consists of

- A set of terminals (tokens of the language)
- A set of non-terminals
- A start symbol
- A set of productions
    - X -> Y1, ... Yn where X is a non-terminal and Yi is a non-terminal, a terminal, or epsilon (empty string)

To analyze input using a CPG,

- Represent the string with just the start symbol.
- Replace any non-terminal in the representation by the right-hand side of some production.
- Repeat the previous step until there are no more non-terminals in the representation.

By doing this, you can generate all the strings that in the language of the CFG.

But we also need a way to use the CFG to generate the parse tree of the input, and to handle errors gracefully.

## Derivations with CFGs

A derivation is a sequence of applied productions that generate a string. But a derivation can be represented as a tree. A parse tree.

A parse tree, when an in-order traversal of the leaves is done, generates the original input. Subtrees bind more tightly.

Left-most derivation - at each step, replace the left-most non-terminal. There is an equivalent notion of a right-most derivation. They generate the same parse tree.

## How do we handle ambiguity in CFGs while building parse trees?

Take operators for example. The concept of __operator precedence__ is not present in the basic concept of CFG derivations, leading to more than one parse tree for some string (more than one right-most or left-most derivations).

This can leave a program ill-defined, so how do we avoid this?

- Rewrite the grammar so that it is unambiguous, so that ambiguous rules are separated out into separate rules that make the precedence clear. For example, E -> E + E | E * E can be written into two rules, E -> E' + E | E' and E' -> (E) * E' | (E), which forces '+' operators to be handled before '*' operators.
    - But this is difficult to trace and verify correctness.
- If you're using tools, you can specify which precedence and associativity that you want.
    - But again, the problem with tools is that it is harder to do custom error handling.
- If you're writing the code, you can force the parser to handle the rule sin a certain order
    - But this isn't declarative and is probably prone to mistakes, although you can use your test suites to verify the precedence and associativity properties still hold.

## How do we handle errors in a parser?

We want the parser to give good feedback upon detecting an error.

- lexical errors (lexical)
- syntax errors (parser)
- semantic errors (type checker)
- correctness ( tests )

Error handler should report errors accurately and clearly for the programmer's benefit; it should recover from an error quickly to continue analyzing the code for syntax issues, and it should not slow down the compilation of valid programs.

- Panic - discard tokens until it finds a __synchronizing token__ like a statement terminator, or a variable declaration, and restarts the analysis from there. Or it may skip until it finds a token that allows it to complete the current subtree, and then simply reports the skipped tokens as not belonging.
- Error Productions - specify known common mistakes as actual rules in the grammar. While this complicates the grammar, it lets you provide warnings or really good error messages.
- Automatic local/global error correction. Find a correct program that is "close" to the program, and hope that the program is the one the programmer desired. But this is hard to implement and slows down parsing of correct programs, since you have to maintain state to figure out what program is "close" to the program. This was useful when compilation was still very slow (a day or more), so it was worth it.
    - But today, the compilation cycle is quick, so users tend to correct just one error per cycle (the most reliable error).

## Abstract Syntax Trees

The parse uses the grammar to generate a parse tree, with some details ignored, that make it convenient to work on.

The AST removes

- single-successor nodes
- parentheses (since the tree itself shows association and precedence)

Hence the AST is more compact and easier to use. It abstracts away the concrete syntax in favor of this abstract syntax.

## Recursive Descent Parsing

This is a top-down parsing algorithm. The AST is constructed from the top, and from left-to-right (in the order of the input tokens). Terminals are occur in the order of the input tokens.

The following functions are needed:

- Need functions that check whether a token matches a terminal production. If it matches, then the input pointer should move to the next token.
- Need functions that check whether a token matches a non-terminal production.
- Need function that tries all productions.

A production E -> T + E would be written as follows:
    - return T() && term(PLUS) && E()
        - The short-circuiting && allows us to avoid evaluating more than we need to.

Productions are combined as follows:
    - input = next;
    - return (next = save, E1()) || (next = save, E2())
        - We save our place in the input so we can backtrack if the production fails.

While this is occurring, we have to build the AST in each production and pass it back with each production for use by the parent production.

To start the parser, we need to initialize 'next' and invoke the production that tries all productions.

## Limitations of Recursive Descent

In naive implementations, parses can fail because in a set of productions, once a clause of the production has succeeded, we cannot go back to that production if we later find out that the **wrong clause** of the production succeeded.

For example, take the production

- E -> T
- T -> int | int * T

with the input __int * int__. In the naive implementation, we have a callstack 

- E() -> E1() -> T() -> T1(),
- true <- true <- true <- true

But since we didn't consume all the input, the parse will fail. Clearly, because we never get an opportunity to try out T2(), the parse never succeeds -- we hit the terminal too early, and there's no way to go back and try T2(), because the code can't automatically detect that that's where the issue is, and doing an exhaustive backtrack would be very expensive.

So the presented recursive descent algorithm is not general, although there are Recursive Descent Algorithms -- they require more complicated backtracking strategies, however.

So the presented algorithm, while easy to implement, will parse an algorithm if
    - For any non-terminal, at most ONE production can succeed, so there is never a need to go back and revisit the decision.
    - In other words, it works only for LEFT FACTORED grammars.

## Left Recursion and Left Factoring

Left recursion in a grammar makes recursive descent into an infinite recursion.

For example, E -> E | T will cause the algorithm to call E() endlessly.

In a left-recursive grammar, some sequence of replacements eventually involves the same non-terminal in the leftmost position, so the input is never consumed.

Instead, we need to turn left-recursion into right-recursion, which is generally done by creating a new non-terminal that is right-recursive, and using it for recursion instead.

See the dragon book for the full general algorithm that can automatically rewrite such grammars.