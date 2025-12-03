---
title: Building a tiny recursive descent parser in Java to think about invariants
category: Computer Science
---

Programs look like they run on instructions, but they do not. They run on shape.

Every nontrivial piece of software hides what one could call a geometric structure that guides what the program is allowed to do and what it can never do. This structure is not visible in the syntax, nor is it obvious from the sequence of operations. It lives beneath the code, in a network of invariants that quietly hold everything together.

I began to think about this after writing a tiny arithmetic evaluator in Java. It was not meant to do anything remarkable. It was only supposed to take a string like:

```
1 + 2 * 3
```

and return the number `7`.

But as the evaluator grew into a small language with its own rules, its own grammar, its own semantics, and its own failure modes, I realized that what held the program together was not only the logic I wrote explicitly. What held it together were the constraints that remained true in every successful run, even when I was not thinking about them.

Those constraints were **invariants**, and one could say they formed the invisible geometry of the evaluator.

To see how they work, it helps to look at the evaluator itself.

---

## A Tiny Language

Even a toy expression evaluator defines a language. Not in the loose everyday sense of the word, but in the formal sense studied in the theory of computation: the set of all strings the evaluator accepts as valid expressions.

For this experiment, I defined a minimal arithmetic language:

- integers
- the operators `+`, `-`, `*`, `/`
- parentheses
- optional whitespace

A grammar for that language looks like this:

```
expr   = term (('+' | '-') term)*
term   = factor (('*' | '/') factor)*
factor = NUMBER | '(' expr ')'
```

This is (close to) context free grammar notation, and it describes the legal shapes of expressions in our mini language.

Each line describes a nonterminal symbol and the patterns it can expand into.
- expr
- term
- factor

A nonterminal symbol is a symbol that does not appear literally in the input, but instead stands for a syntactic construct that needs to be expanded into other symbols. They are like “categories” of expressions. In our case:
- a `factor` is a number or something in parentheses.
- a `term` is a factor with multiplications and divisions.
- an `expr` is a term with additions and subtractions.

This grammar already gives the language a recognizable structure. It encodes operator precedence. It encodes associativity. It encodes the idea that parentheses carve out subexpressions. These rules describe the legal shapes of expressions, the same way a geometric axiom describes the legal shapes of polygons.

Let’s look closely at how operator precedence is encoded in the grammar. The grammar ensures the correct precedence because parsing proceeds in a structured, bottom up way. When the parser encounters an expr, it does not evaluate it directly. Instead, expr is defined in terms of term, so the parser must call the method that parses a term before it can do anything at the expression level. The same relationship exists between term and factor: a term is built from factors, so factors are parsed and evaluated first.

This layering of nonterminals is what produces precedence. Operations inside parentheses have the highest priority because they appear at the lowest level of the grammar, as part of the factor rule. Since the parser must successfully parse a factor before completing a term, and a term before completing an expr, the structure of the grammar itself guarantees that operations inside parentheses and multiplications and divisions are handled before additions and subtractions.

Now here is the Java evaluator that implements this language:

```java
// A tiny arithmetic expression evaluator that parses and computes expressions
// like "1 + 2 * 3" using a hand written recursive descent parser.
//
// Grammar:
//   expr   = term (('+' | '-') term)*
//   term   = factor (('*' | '/') factor)*
//   factor = NUMBER | '(' expr ')'
//
// This evaluator combines parsing and evaluation in one pass.

public class ExprEvaluator {

    // The full input expression
    private final String input;

    // Current position (cursor) inside the input string
    private int pos;

    public ExprEvaluator(String input) {
        this.input = input;
        this.pos = 0;
    }

    // Entry point: parse a full expression and return its numeric result
    public int evaluate() {
        int value = parseExpression();

        // After parsing the expression, consume any remaining whitespace
        skipWhitespace();

        // If we are not at the end, the input had extra garbage
        if (pos != input.length()) {
            throw new IllegalStateException("Unexpected characters at end: " + input.substring(pos));
        }

        return value;
    }

    // ------------------------------------------------------------
    // Grammar rule:
    //   expr = term (('+' | '-') term)*
    //
    // This handles + and - at the top level
    // ------------------------------------------------------------
    private int parseExpression() {
        int value = parseTerm();

        // After reading the first term, keep consuming + term or - term
        while (true) {
            skipWhitespace();
            char c = peek();

            if (c == '+') {
                match('+');
                int right = parseTerm();
                value = value + right;

            } else if (c == '-') {
                match('-');
                int right = parseTerm();
                value = value - right;

            } else {
                // No more + or - operators, expression ends here
                break;
            }
        }

        return value;
    }

    // ------------------------------------------------------------
    // Grammar rule:
    //   term = factor (('*' | '/') factor)*
    //
    // This handles * and /, which have higher precedence,
    // so this is called before parseExpression processes + and -.
    // ------------------------------------------------------------
    private int parseTerm() {
        int value = parseFactor();

        // Keep consuming * factor or / factor
        while (true) {
            skipWhitespace();
            char c = peek();

            if (c == '*') {
                match('*');
                int right = parseFactor();
                value = value * right;

            } else if (c == '/') {
                match('/');
                int right = parseFactor();
                value = value / right;  // integer division

            } else {
                break;
            }
        }

        return value;
    }

    // ------------------------------------------------------------
    // Grammar rule:
    //   factor = NUMBER | '(' expr ')'
    //
    // A factor is either a literal number or a parenthesized expression.
    // ------------------------------------------------------------
    private int parseFactor() {
        skipWhitespace();
        char c = peek();

        // Case 1: expression inside parentheses
        if (c == '(') {
            match('(');                 // consume '('
            int value = parseExpression(); // recursively parse inside
            if (!match(')')) {
                throw new IllegalStateException("Expected ')' at position " + pos);
            }
            return value;

            // Case 2: number
        } else {
            return parseNumber();
        }
    }

    // ------------------------------------------------------------
    // Parse a sequence of digits and return the integer value.
    // Example: input "123 + ..." should return 123 and move pos forward.
    // ------------------------------------------------------------
    private int parseNumber() {
        skipWhitespace();
        int start = pos;

        // Read all consecutive digits
        while (pos < input.length() && Character.isDigit(input.charAt(pos))) {
            pos++;
        }

        // If no digits were found, that's a syntax error
        if (start == pos) {
            throw new IllegalStateException("Expected number at position " + pos);
        }

        // Convert the substring to an integer
        String slice = input.substring(start, pos);
        return Integer.parseInt(slice);
    }

    // ------------------------------------------------------------
    // Utility: consume whitespace characters
    // ------------------------------------------------------------
    private void skipWhitespace() {
        while (pos < input.length() && Character.isWhitespace(input.charAt(pos))) {
            pos++;
        }
    }

    // ------------------------------------------------------------
    // Utility: try to match a specific character.
    // If the next non whitespace character is `expected`, consume it and return true.
    // Otherwise return false and do NOT consume anything.
    //
    // This method is where the cursor invariant lives.
    // ------------------------------------------------------------
    private boolean match(char expected) {
        skipWhitespace();

        if (pos < input.length() && input.charAt(pos) == expected) {
            pos++;  // consume the character
            return true;
        }

        return false; // do not consume anything if it does not match
    }

    // ------------------------------------------------------------
    // Utility: look at the next non whitespace character without consuming it.
    // Returns '\0' if we reached the end.
    // ------------------------------------------------------------
    private char peek() {
        skipWhitespace();
        if (pos >= input.length()) {
            return '\0';
        }
        return input.charAt(pos);
    }
}
```

This is enough to evaluate expressions of reasonable complexity. It respects precedence. It handles parentheses. It computes the correct result for every valid input.

But it only works because certain invariants hold at every step, even though the code never states them.

To understand the geometry, let's break it.

---

## Breaking the Invariant

The crucial internal state of this evaluator is the cursor, the variable `pos` that points into the input string. The parser moves forward through the string one character at a time, and every parsing method relies on a quiet assumption:

**At the moment a parsing method begins, the cursor must be at the start of a valid instance of the grammar rule the method implements.**

That is the syntactic invariant that makes the evaluator work.

To see what happens when it fails, introduce a tiny bug. Change the method `match` so that it increments the cursor even when the character does not match:

```java
private boolean match(char expected) {
    skipWhitespace();
    if (pos < input.length() && input.charAt(pos) == expected) {
        pos++;
        return true;
    } else {
        pos++;  // Bug: move cursor on mismatch
        return false;
    }
}
```

This single incorrect increment breaks the cursor invariant. The bug seems harmless. The parser still recognizes the grammar. The code still compiles and runs.

Then you give the evaluator the simplest possible expression:

```
1 + 2
```

Sometimes it evaluates correctly. Sometimes it skips the plus sign. Sometimes it eats a digit. Sometimes it interprets half the expression as a number and half as nothing at all. If you give it longer input, the behavior becomes unpredictable in ways that do not feel random but also do not feel explainable. The evaluator enters states that the grammar never allowed it to enter.

The cause is straightforward: the parser begins calling methods in states that violate the grammar. The cursor no longer points at a valid remainder of the input. The invisible geometry collapses.

The program is no longer running inside the shape defined by the language. It is running outside of it, where nothing is well defined.

---

## The Syntactic and Semantic Layers

This failure exposes a distinction that programming often hides but that the theory of computation makes explicit.

In formal language theory, a language is a set of strings over an alphabet. Arithmetic expressions form a context free language. Parentheses matching requires stack like behavior. A parser is a device that verifies that the input belongs to that language. When the cursor invariant is broken, the parser no longer respects the shape of the language. It loses the guarantee that the remaining suffix of the input is a valid continuation of the grammar. In the automata world, this is equivalent to a pushdown automaton losing sync between its input and its stack. You end up with a state that does not correspond to any legal configuration.

That is the syntactic layer.

Above that lies the semantic layer. Even if the syntax is correct, the evaluator also relies on semantic invariants. For example, `parseTerm` must compute the correct value for the part of the input it consumed. Precedence must be honored. Subexpressions must evaluate to meaningful values. None of these conditions are properties of the input string. They belong to the meaning of the language.

In the broken version of the evaluator, even when the syntax layer is lucky enough to align, the semantic layer still collapses because the syntactic invariants no longer guarantee that the parsed structure corresponds to anything meaningful.

This separation between syntactic invariants and semantic invariants is exactly what makes programming languages possible. Syntax constrains the shape of the input. Semantics constrains the behavior of evaluation. They form two layers of geometry that must agree for a program to behave.

---

## The Invariant Reveals Itself

The moment the evaluator fails, the underlying invariant becomes visible.

You discover that the correctness of the evaluator depends completely on a fact you never stated:

**Whenever a parsing method begins, the cursor must point to the beginning of a valid instance of the grammar rule that method implements.**

If this stops being true, everything stops making sense. The logic is still in place. But the geometry that held the evaluator together is gone.

It is only by repairing the invariant that the evaluator regains its footing:

```java
if (pos < input.length() && input.charAt(pos) == expected) {
    pos++;
    return true;
}
return false;
```

The moment this correction is made, meaning returns. The evaluator moves through the input in the shape allowed by the grammar. The stack like structure of nested parentheses comes back. The semantics of precedence come back. The syntax and semantics lock together again.

The program becomes a program again.

---

## Closing Thought

We often think of correctness as something we verify with tests. But long before a test runs, long before a function is called, and even before the program is fully written, there are truths that must remain true. These invariants are the rails on which the logic travels. They are the laws that prevent the program from drifting into states where nothing means anything.

A tiny evaluator written in an afternoon can reveal this. The geometry is always there, even in systems far simpler than the ones we normally build. It becomes visible only when the invariant fails.

---

If you're curious and want to run this evaluator by yourself, you can find the code in the <a href="https://github.com/puzzled-frog/tiny-recursive-descent-parser" target="_blank" rel="noopener noreferrer">GitHub repository</a>.
