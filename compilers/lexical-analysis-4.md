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

epsilon-closure of a state is the set of states you can reach from the state by following 1 or more epsilon transitions

If a NFA has N states, then can be in at most N states at the same time, and therefore at most 2^N - 1 configurations exist.

So if we have an NFA with set of states S, start state s, and accepting states F, and for each character _a_ and state X, _a_(X) is the set of states you can reach using the transition _a_ from X, and we also have _e_(X), which is the epsilon closure of X.

Then the DFA is 

- States: the set of all subsets of S.
- Start State: _e(s)
- Accepting states: all states where at least one member is in F (from the NFA).
- Transition function: state X can go to Y on input _a_ if _e_(_a_(X)) is equal to state Y, which guarantees that this is a function.

This DFA records the set of states the NFA could go into on an input.

## Implementing an NFA and DFA in code

Implementing a DFA can be done with a 2D table.

- One dimension is the states (subsets of the NFA)
- One dimension is the input symbol.
- For each transition _a_ on state S, the next state is stored at Table[S][_a_]
- While there is still input, and while Table[S][_a_] is not an error, we continue using the input to update the current state. If Table [S][_a_] is an error, we reject. If we run out of input and we are not in an accepting state, we reject. Otherwise we accept.

To avoid duplicate rows, we could instead do some indirection and have an Array, where each index represents a state. Each cell in the array points to a different array, where each index represents an input character, and each cell stores the next state. Thus, multiple states can use the same input transition array, which saves space.


Implementing the NFA can be done with a 2D table as well. This table will be much smaller.

- One dimension per state of the NFA.
- One dimension per input AND epsilon.
- Each cell of the array stores the set of states that can be reached from state X using the input _a_

We track the set of states the NFA is currently in, so the processing loop is much more expensive than in the DFA, since you have to do things like follow the epsilon closures