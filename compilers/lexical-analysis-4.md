# Implementing Lexical Analysis

## Regular Expressions - common notation

- At least one: A+
- Union: A | B
- Option: A?
- Range: [a-z]
- Excluded range: [^a-z]

To apply a lexical specification, get the set of regular expressions that recognize the strings that belong in the language. Then, given some input, apply the union of all those regular expressions to the input, and remove each matching prefix substring until the entire input is consumed.

Ambiguities:

- If two prefixes are in the language, which one gets removed? The answer that most lexical analyzers go with is to always take the longer prefix: **Maximal Munch**, which mirrors how humans process text. This almost always does the right thing.
- Which token is used if a prefix could belong to two possible token classes? For example, 'if' is both a keyword and an identifier. The answer is that we assign token classes with priorities.
- What should the analyzer do if no rule matches? Answer is to right a token class for all strings that are not in the error class, that is the last in priority. This lets us be sloppy with how it is defined -- the error class matches all strings, and since it is last in priority, it will only be applied if none of the other token classes have matched.

Good algorithms: Want to pass over the input just once, and do few operations per character if possible.

## Implementing Regular Expressions with Finite Automata

A finite automaton consists of 

- Input alphabet
- Set of states
- Start state
- A set of accepting states
- A set of valid transitions from state to state, based on the next input character it receives

If machine terminates in anything other than an accepting state, or if the machine gets stuck in the middle of the input, the input string is rejected.

The "language" of a finite automaton is the set of all strings that the automaton can accept.

On execution, you track the current state and the current position in the input.

## Implementing REgular Expressions with Nondeterministic Finite Automata

Determnistic FInite Automata (DFA)

- One transition per input per state
- No epsilon-moves
- DFA is in only one state at a time.

Nondeterministic Finite Automata (NFA)

- Epsilon moves
- Multiple transitions for one input in a given state
- NFA maintains a set of states that it is currently in
- NFA accepts if, at the end of the input, the set of states it is in contains an accepting state.

Both NFAs and DFAs recognize only the regular languages. DFAs are faster to execute, but NFAs are much smaller in memory.

## Converting Regular Expressions into NFAs

Regular Expressions --> NFA --> DFA --> Table-driven implementation of the DFA

- For each kind of regexp, define an equivalent NFA.
- See 04-03 for how to build the full NFA recursively from the base cases (epsilon and character) and the operators. Also see the theory of computation book.


## Converting an NFA to a DFA