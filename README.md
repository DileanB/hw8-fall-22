# Homework 8: Interpreter

## Notes

//My notes

- Please download the homework from [here](https://github.com/umass-compsci-220/hw8-fall-22/raw/main/hw8-fall-22.zip)
- This project will be using Node.js and VSCode
  - Reference the [previous homeworks installation instructions](https://github.com/umass-compsci-220/hw6-fall-22/blob/main/INSTALLATION.md) if need help getting everything installed
- After you download and unzip the project, open the folder one level higher than the `/src` folder in VSCode (File -> Open Folder)
- Run `npm install` inside the terminal (Terminal -> New Terminal)
- Your directory should look something like this:

```txt
hw8-fall-22/
  node_modules/
  include/
  src/
    interpreter.js
    interpreter.test.js
    main.js
  package-lock.json
  package.json
  build.js
```

## Index

- [Description](#description)
- [Learning Objectives](#learning-objectives)
- [Student Expectations](#student-expectations)
- [Getting Started](#getting-started)
  - [Concrete Syntax](#concrete-syntax)
  - [Parser](#parser)
  - [State](#state)
  - [Error Handling](#error-handling)
- [Programming Tasks](#programming-tasks)
  - [interpExpression](#interpexpression)
  - [interpStatement](#interpstatement)
  - [interpProgram](#interpprogram)
- [Testing and Approach](#testing-and-approach)
- [Submitting](#submitting)

## Description

For this project, you will write an [_interpreter_](<https://en.wikipedia.org/wiki/Interpreter_(computing)>) for a small programming language similar to JavaScript. To write an interpreter, you need a parser to turn the program's concrete syntax into an abstract syntax tree (AST; as explained in class). You don't need to write a parser yourself. We have provided one for you. You will then traverse the AST making necessary checks and executing corresponding statements and expressions.

## Learning Objectives

Throughout this assignment, students will learn:

- The fundamentals of programming language implementation
- How to read the grammar for a concrete syntax
- How to work and read abstract syntax trees
- Elementary principles of expression and statement evaluation

## Student Expectations

Students may be graded on:

- Their ability to complete the [programming tasks](#programming-tasks) documented below
- How they design unit-tests for all relevant functions and methods

## Getting Started

### Concrete Syntax

The following grammar describes the concrete syntax of the fragment of JavaScript that you will be working with:

```txt
Numbers             n ::= ...                 numeric (positive and negative integer numbers)

Variables           x ::= ...                 variable name (a sequence of uppercase or lowercase alphabetic letters)

Expressions         e ::= n                   numeric constant
                      | true                  boolean value true
                      | false                 boolean value false
                      | x                     variable reference
                      | e1 + e2               addition
                      | e1 - e2               subtraction
                      | e1 * e2               multiplication
                      | e1 / e2               division
                      | e1 && e2              logical and
                      | e1 || e2              logical or
                      | e1 < e2               less than
                      | e1 > e2               greater than
                      | e1 === e2             equal to

Statements          s ::= let x = e;          variable declaration
                      | x = e;                assignment
                      | if (e) b1 else b2     conditional
                      | while (e) b           loop
                      | print(e);             display to console

Blocks               b ::= { s1 ... sn }

Programs            p ::= s1 ... sn
```

`Numbers` and `Variables` have been omitted for simplicity. However, you may assume that they are structured the same as they are in JavaScript. The parser does not accept real numbers, `Infinity` or `NaN`, but your interpreter should produce these as expression results where JavaScript would. The parser only accepts identifiers formed of letters.

### Parser

We have provided two parsing functions the function `parseExpression` parses an expression (`e`) and the function `parseProgram` parses a program (`p`). Their type signatures are documented below:

```ts
type Result<T> = { ok: true, value: T } | { ok: false, message: string };
type BinOp = "+" | "-" | "*" | "/" | "&&" | "||" | ">" | "<" | "===";
type Expr =
  | { kind: "boolean"; value: boolean }
  | { kind: "number"; value: number }
  | { kind: "variable"; name: string }
  | { kind: "operator"; op: BinOp; e1: Expr; e2: Expr };

type Stmt =
  | { kind: "let"; name: string; expression: Expr }
  | { kind: "assignment"; name: string; expression: Expr }
  | { kind: "if"; test: Expr; truePart: Stmt[]; falsePart: Stmt[] }
  | { kind: "while"; test: Expr; body: Stmt[] }
  | { kind: "print"; expression: Expr };

parseExpression(str: string): Result<Expr>
parseProgram(str: string): Result<Stmt[]>
```

On success, these functions will return an object that contains the the corresponding AST for the given string. On failure, these functions return an object with the failure message.

### State

The State type is defined as follows:

```ts
type State = { [key: string]: State | number | boolean };
```

This notation indicates that a `State` object has a variable number of properties with values of type `number`, `boolean` (representing values of variables that are in scope), or of type `State` (link to the parent scope).

A block starts a new inner scope. A variable declared in a block will shadow an outer declaration (any variable use will refer to the inner declaration). On exiting a scope, variables declared there are no longer accessible (since we don't have closures). Thus, they should not be in the global state at the end. The nesting of block scopes corresponds to a stack, which you can implement as a linked list, by adding to your `State` object a link to an outer scope. Since the link is just another property, this allows all functions to keep their signatures. To ensure the link name does not clash with a program variable, use a property name that is not an identifier (JavaScript allows this). The global state cannot have extra properties, but does not need a link, as the last state on the list.

### Behavior

The behavior of our interpreter should be similar to the `node` interpreter in "[strict mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode)" (with some exceptions). To test what your interpreter should do in a scenario, you may use the `node --use-strict` command in a terminal to open a Read Eval Print Loop (REPL). This interface will allow you to input statements and expressions and will display an error or an the evaluated result.

Exceptions:

- _Arithmetic and greater/less-than comparison may only happen between numbers_
- _Logical operations may only happen between booleans_

### Error Handling

An interpreter can generally not continue meaningfully after an error (as opposed to compilers). Thus, if you find an error, **you should throw an error, using an informative error message (i.e. "Arithmetic may only happen between numbers")**. You need to do a number of checks (e.g., correct typing, and missing or duplicate declarations). You may assume that an AST has the right fields and types. **Do can not assume other checks**, even if done by the parser, as your functions can be tested with ASTs that don’t come from the parser.

## Resources

- [Accessing Object Fields in Vanilla JavaScript](https://github.com/umass-compsci-220/hw6-fall-22#accessing-fields-in-vanilla-javascript)

## Programming Tasks

Your task is to implement the following functions inside of `src/interpreter.js`. You may do them in any order. The inputs of these functions are abstract syntax trees (`object`), not concrete syntax (`string`). Therefore, you can run your code by using the parser, or by directly constructing ASTs by hand.

### `interpExpression`

Given a state object and an AST of an expression as arguments, `interpExpression` returns the result of the expression (number or boolean). It should throw an error if the statement is invalid (see [Behavior](#behavior) and [Error Handling](#error-handling))

```ts
interpExpression(state: State, e: Expr): number | boolean
```

### `interpStatement`

Given a state object and an AST of a statement, `interpStatement` updates the `State` object and returns it. It should throw an error if the statement is invalid (see [Behavior](#behavior) and [Error Handling](#error-handling)).

```ts
interpStatement(state: State, p: Stmt): State
```

### `interpProgram`

Given the AST of a program, `interpProgram` returns the final state of the program. It should throw an error if any statement of expression is invalid. It should throw an error if the statement is invalid (see [Behavior](#behavior) and [Error Handling](#error-handling)).

```ts
interpProgram(p: Stmt[]): State
```

Example:

```js
interpProgram(parseProgram("let x = 10; x = x * 2;").value);

// {
//   x: 20;
// }

interpProgram([
  { kind: "let", name: "x", expression: { kind: "number", value: 10 } },
  {
    kind: "assignment",
    name: "x",
    expression: {
      kind: "operator",
      op: "*",
      e1: { kind: "variable", name: "x" },
      e2: { kind: "number", value: 2 },
    },
  },
]);

// {
//   x: 20;
// }
```

## Testing and Approach

Implement `interpExpression`, following the template shown in class. You can use an empty object (`{ }`) for the state if you do not have any variables, or you can set the values of variables by hand. For example:

```js
const assert = require("assert");

test("multiplication with a variable", () => {
  const r = interpExpression({ x: 10 }, parseExpression("x * 2").value);

  assert(r === 20);
});
```

Implement `interpStatement` and `interpProgram`, following the template shown in class. You should be able to test that assignment updates variables. For example:

```js
const assert = require("assert");

test("assignment", () => {
  const st = interpProgram(parseProgram("let x = 10; x = 20;").value);

  assert(st.x === 20);
});
```

Finally, test your interpreter with some simple programs. For example, you should be able to interpret an iterative factorial or Fibonacci sequence computation.

You may find yourself in a scenario where you need to write a test that verifies a program throws an error. To run your tests in Node.js, we are using the Jest testing framework. We have ignored its features the past few homeworks for simplicity, but here is an example of how you would write a test like that:

```js
function sqrt(n) {
  if (n < 0) throw new Error("Input must be positive or zero.");

  // Do some iterations of Newton's method
}

test("sqrt fails on invalid input", () => {
  expect(() => {
    sqrt(-1);
  }).toThrow();
});
```

Notice that we passed a function that calls `sqrt` with invalid input. If we just passed the function call with invalid input, our test would fail (because `sqrt(-1)` would throw) before the testing framework could capture that error and verify we were actually expecting it to error (`.toThrow()`).

You can read more on the [Jest documentation on `.toThrow()`](https://jestjs.io/docs/expect#tothrowerror).

## Submitting

- Run `npm run build`
- Login to Gradescope
- Open the assignment submission popup
  - Click the assignment
- Open your file explorer and navigate to the folder of the project
  - This is the folder that immediately contains: `hw8-submission.zip`, `node_modules`, `src/`, `package.json`, `package-lock.json`
- Drag and drop the generated `hw8-submission.zip` into Gradescope
- Click upload
