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