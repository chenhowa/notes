# Lexical Analysis

## What do lexical analyzers do?

Lexical analyzer sends turns string into tokens, which are then sent to the parser.

A token is a (class, string) tuple.

Substrings of the program that correspond to a token are called "lexemes". So we could also say that a token is a (class, lexeme) pair.

## What troubles might lexical analyzers run into?

You want to recognize a substring as part of a token. But how do you know when a substring ends as a complete token?

Some lexical analysis requires lookahead, like when you look forward to see whether "do" is a keyword, or just part of a variable name.

But lookahead may be expensive. So in designing a language, you may include and enforce natural separators like whitespace to limit lookahead

So the primary issue is lookahead.

## How can regular languages be used to describe programming languages?

Regular languages can be used to specify which strings belong to a token class.

Two kinds of regular expressions: single character, and epsilon (the empty string). These regular expressions can be combined with operators that combine the languages of regular expressions into different, usually larger languages:

- Union
- Concatenation
- Iteration (0 or more times)

Regular expressions over an alphabet are epsilon, a single letter from the alphabet, or the operator expressions built from the alphabet. This comprises the 'grammar' of the language.

### Formal Languages

A language over an alphabet is some set of strings of characters drawn from the alphabet.

Many formal languages also have a meaning function that maps the string to its meaning. For example, in regular expressions, a meaning function maps it to the set of all strings that the regular expression can represent.

Meaning functions make clear what is syntax, and what is semantics. This is what allows us to change the syntax of a language while keeping the meaning of the overall program the same. Since meaning functions are not injective (1-1), it is possible to do optimizations -- to write a different, faster program that does the same thing. When it comes to lexical analyzers, this means that each lexeme corresponds to just one token class -- never two or more.