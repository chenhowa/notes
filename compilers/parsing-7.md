# More Sophisticated Parsing

## Predictive Parsing

Predictive parsing is a top-down parsing strategy that predicts which production to use by looking at the next few tokens to decide.

- Use lookahead
- Restricted grammars

The parsers are thus completely deterministic. However, they only accept the LL(k) grammars. Left to right scan, Leftmost derivation, k tokens of lookahead. But typically we only use k = 1.

### Comparison to recursive descent.

- Recursive descent - at each step, there are many choices of productions to use, so we need backtracking if we go down the wrong path.
- LL(1) has only one choice of production at each step. That is, if we have the current derivation wAB, where A and B are nonterminals, then by knowing that the next input is t, we know exactly which nonterminal A must be -- otherwise the parse fails.

For LL(k) grammars, we need our productions to be left-factored, or else a single token of lookahead will not be enough.

```
E -> T + E | T
T -> int | int * T | (E)
```

when left-factored, becomes

```
E -> T X
X -> + E | ""               ( "" represents empty string.)
```

### General algorithm for parsing with a LL(1) grammar

Once we have a left-factored grammar, we can generate a LL(1) parsing table. What table consists of an X-axis that represents the current leftmost nonterminal, and the Y-axis represents the current token from the input (including the EOF token). Each entry in the table represents how to replace the leftmost non-terminal upon seeing a particular input. Empty entries represent error entries, that indicate the parse has failed.

So generally, the algorithm is:

- If the leftmost thing is a non-terminal,
    - Get the next input token a,
    - Replace S with [S, a]
- If the leftmost thing is a terminal (which might be "", in which case no input is consumed)
    - Get the next input token a
    - match a with the terminal, and remove the terminal. If no match, then error.

We use a stack to record the list of Non-terminals and unmatched terminals. The top of the stack is the leftmost pending terminal or non-terminal.

We reject on getting an error entry. We accept if we reach the end of the input and the stack is empty (the EOF token also matched).

Notice that this process is a lot less different from recursive descent in terms of how to generate an Abstract Syntax tree from this. To generate an AST, each non-terminal represents the root of a subtree, so you need to maintain a pointer to it. When the nonterminal is expanded, the expansion becomes the children of that root. So the implementation should involve putting pointers on the stack mentioned above.

### First Sets

The first part of constructing LL(1) Parsing tables requires First Sets.

The First Set of a non-terminal A is the set of all terminals t for which there exists a derivation A -> ... -> tB, where B is a possibly empty nonterminal. Naturally, if t is a terminal, First(t) = { t } . First Set also includes "" if A -> ... -> ""

Algorithm:
- For a terminal t, First(t) = { t }
- For a nonterminal X, if X -> "" or if X -> A1...An and "" is in the First(Ai) for 1 lte i lte n , then "" is in First(X).
- For nonterminals X, Y, if X -> ... -> A1...AnY and "" is in First(Ai) for 1 lt i lt n, then clearly First(Y) is a subset of First(X).
- For a non-terminal X, Y, C, if X -> .. -> YC and "" is not in First(C), then First(Y) - { "" } is a subset of First(X).

You can build the First set of a nonterminal X by computing the First Sets from the Right-hand side of the productions for X, and merging them into the First Set of X.


### Follow Sets.

The second part of constructing LL(1) parsing tables requires Follow Sets.

The Follow Set of a non-terminal A __for a given production S__ is the set of all terminals t for which there exists a derivation that generates *A t* somewhere in the derivation. This means that it is if we can get rid of A, we can immediately match a terminal from the Follow Set of A.

Observe that if X -> AB, then

- First(B) is a subset of Follow(A). 
- Follow(X) is a subset of Follow(B), since if there is a nonterminal that follows X then this derivation means that it could follow B as well.
- if B -> ... -> "", then Follow(X) is a subset of Follow(A), for the same reason as the previous point.
- if S is the starting parse nonterminal, we say that the EOF symbol ($) is in Follow(S)

Algorithm to compute a follow set for a grammar, where S is the singular starting parse nonterminal:

- $ is in Follow(S).
- For each production A -> XZ:
    - First(Z) - { "" } is a subset of Follow(X).
    - If "" is in the First(Z), then Follow(A) is a subset of Follow(X).
    - Follow(A) is a subset of Follow(Z).

### Building LL(1) Parsing Tables

To build a parsing table T for a grammar,

- For each production A -> B in G, where B represents some right-hand side of a production, including "":
    - For each terminal t in First(B), then we add an entry in the table T[A, t] = B
    - If "" is in First(B), then it is possible for A to disappear completely, so for each t in Follow(A), we can say T[A, t] = B. This means that the input t **uses** the production successfully.
    - If "" is in First(B), and EOF is in Follow(A), then T[A, $] = B. This is just a special case of the previous rule.

When you try to build a parsing table for a non-LL1 grammar, what happens? Basically, you will find that when you build the parsing table, a single space T[A, t] will have two different entries that need to be assigned to it. Or you end up in a recursive loop.... If you can build a LL1 parsing table for a grammar, then it is an LL1 grammar.

Grammars that are ambiguous, not left-factored, left-recursive -- a non-exhaustive list of criteria that means a grammar is not LL(1). Most programming languages are not LL1! They can't capture all the constructs that make the languages great.

So to summarize where we are. Recursive descent parsers are very general, but are hard to generate code for. LL(1) grammars are not general, but they are relatively easy to generate code for.