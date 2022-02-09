# BML Specification
## 1 Introduction
The Banking Meta Language is a functional, non-Turing complete, strongly typed language used to represent a financial institution’s product catalog which includes dynamic rate adjustments and explicit loan denial.

Once Banking Meta Language source code has been parsed and been determined to be valid it must be executed. We leverage existing tools in the .NET ecosystem, for example the System.Linq.Expressions namespace to translate our parsed expressions into .NET domain expressions which can then be compiled and executed all at run time. This provides us with multiple benefits. the largest being we can leverage the existing C# compiler to do all the dirty low-level work for us.

An example of some Banking Meta Language code, hopefully it's easy to grok.

```
fun LoanAmount =>
  table CreditScore
  | in (720, 850] => 75000
  | in (640, 720] => 40000
  | in (0, 640) => 28000
  _ => 0

rule deny "Reject this loan if credit score is negative!" =>
  CreditScore < 0

rule deny "Only those with great credit can borrow 100K+" =>
  CreditScore < 800 and LoanAmount >= 100000
```

The lexical and syntactic grammars make use of regular expressions to concisely convey themselves. In the following table `A` and `B` are representations of patterns.

| Notation | Meaning |
| -------- | ------- |
| (A) | A is treated as unit and may be combined with anything below |
| A+ | Matches one or more occurrences of A |
| A* | Matches zero or more occurrences of A |
| A? | Matches A or nothing, treating A as optional |
| A B | Matches A and then B |
| A \| B | Matches A or B, but not both |
| [ char - char ] | Range of ASCII characters |
| [ ^ char - char ] | Any characters except those in the range |

## 2 Execution Flow

```
+-------------+     +-----------+     +----------+     +--------------+     +------------+
|             |     |           |     |          |     | Expression   |     |            |
| Source code |---->| Lexer     |---->| Parser   |---->| Translation  |---->| .NET JIT   |
|             |     |           |     |          |     | Layer        |     |            |
+-------------+     +-----------+     +----------+     +--------------+     +------------+
```

Source code is fed into the lexer which takes a stream of characters and converts meaningful sequence of characters into tokens. This list of ordered tokens is then passed into the Parser which when given a valid sequence of tokens produces a corresponding syntax tree. From here the syntax tree is translated into .NET specific expressions which can then be evaluated by the .NET JIT.

## 3 Lexical Analysis
Lexical analysis converts an input stream of Unicode characters into a stream of tokens by iteratively processing the stream. If more than one token can match a sequence of characters in the source file, lexical processing always forms the longest possible lexical element. What this means practically is that if we have the sequence of characters '!=', this could match both the unary not '!' and '!=' binary not equals, so the lexer would chose the longer and the character sequence would be tokenized as not equals.

### 3.1 Ignored Character Sequences
In Banking Meta Language white space, new line characters whether these come in the form of carriage return or line break, and the tab character are all ignored. The exception to this is when these character sequences are found in a string, in that case they will be consumed as part of the string.

| Character Sequence |
| ------------------ |
| ' ', '\r', '\t', '\n' |

### 3.2 Comments
Banking Meta Language supports simple comments of the form '//'. The lexer when encountering this sequence of characters will consume all the tokens until the next line is reached, ignoring the comment.

### 3.3 Keywords
| Token Type | Character Sequence |
| ---------- | ------------------ |
| And | and |
| Or | or |
| True | true |
| False | false |
| Function | fun |
| Rule | rule |
| Table | table |
| Deny | deny |
| Adjust | adjust |
| In | in |
| Out | out |

### 3.4 Single Character Tokens
| Token Type | Character Sequence |
| ---------- | ------------------ |
| LeftParen | ( |
| RightParen | ) |
| LeftBracket | [ |
| RightBracket | ] |
| Slash | / |
| Star | * |
| Minus | - |
| Plus | + |
| Pipe | \| |
| Comma | , |
| LeftBrace | { |
| RightBrace | } |
| Question | ? |
| Colon | : |
| Discard | _ |

### 3.5 One or Two Character Tokens
| Token Type | Character Sequence |
| ---------- | ------------------ |
| Bang | ! |
| BangEqual | != |
| Lambda | => |
| EqualEqual | == |
| Greater | > |
| GreaterEqual | >= |
| Less | < |
| LessEqual | <= |

### 3.6 Literals
| Token Type | Character Sequence |
| ---------- | ------------------ |
| Identifier | Any sequence of alpha numeric characters which start with an alpha character including underscore '_' and which does match a defined keyword. |
| String | '"' '<any character except ">'* '"' |
| Integer | [0-9]+ |
| FloatingPoint | [0-9]+ '.' [0-9]+ |

Strings are allowed to be multi line. The scanner when consuming a string will consume any character until it reaches a terminating double quote character, '"'. There is currently not support for escaping the double quote.

### 3.7 Special Tokens
| Token Type | Character Sequence |
| ---------- | ------------------ |
| Eof        | Special token indicating that the end of the character stream has been reached. |

### 3.8 Domain Keywords
Domain keywords are a special type of keyword which when used as an expression signal to the expression translation layer that a substitution with a domain object property is required here. Domain keywords when used as the identifier in a function signal to the expression translation layer that the function will return a value for that domain property. 

As an example, `fun LoanAmount => ...` says this function defines a product loan amount, while `table CreditScore ...` says use the domain object credit score property value as the table tuple expression. More specific behavior regarding domain keywords and functions can be found in the function expression section.

```
domain keywords
  ::= 'CreditScore'                  | 'LoanAmount'                   | 'LoanTerm'                     
    | 'InterestRate'                 | 'LoanToValuePercent'           | 'DebtToIncomePercent'          
    | 'UnsecuredDebtToIncomePercent' | 'PaymentToIncomePercent'       | 'MinimumMonthlyPayment'   
    | 'Mileage'                      | 'VehicleAge'                   | 'WatercraftLength' 
    | 'PromotionalInterestRate'      | 'PromotionalDuration'          | 'PrimeRateModifies'            
    | 'PrimeRateType'                | 'DrawPeriodInMonths'           | 'MaxLienPosition'              
    | 'LiabilityCount'               | 'OldestLiabilityInMonths'      | 'GrossMonthlyIncome'      
    | 'GrossMonthlyIncomeRatio'      | 'TotalUnsecuredDebt'           
    | 'TotalUnsecuredDebtInstitutionSpecific'
``` 

## 4 Expressions
Expressions are the fundamental building block of Banking Meta Language and are expected to evaluate eventually to either a number, a string, a boolean, a domain keyword, an identifier, or an expression grouping. Almost everything is an expression.

### 4.1 Interval Expressions
An expression of the form `( '[' | '(' ) expression ',' expression ( ']' | ')' )` is an interval expression. Interval expressions are used to represent an interval between two numbers where substituting `expression` from the interval expression form with *X1* and *X2*, respectively, *X1 < X2* must be true. The opening and closing characters represent if the interval is exclusive or inclusive. Using the earlier notation and introducing a number *n* the following table can be used to determine if *n* satisfies an interval.

| Expression |   |
| - | - |
| *[ X1,*  | *n >= X1* |
| *( X1,*  | *n > X1* |
| *,X2 ]* | *n <= X2* |
| *,X2 )* | *n < X2* |

#### 4.1.1 Example
`[42, 100)`
`(10.1111, 11)`

### 4.2 Unary Expressions
An expression of the form `( '!' | '-' | '>' | '>=' | '<' | '<=' | '!=' | '==' | 'in' | 'out' ) unary | primary` is a unary expression. We deviate a bit from the traditional unary operators `!` and `-` to also include comparison operators. This change exists to have valid tuple expressions in our table language construct.

### 4.3 Tuple Expressions
An expression of the form `expression ( ',' expression )*` is a tuple expression. This allows for any number of expressions to be joined by a comma. A tuple can be expressed as a single expression and will be inferred depending on usage. Banking Meta Language has no support for empty tuples also referred to as the 0-tuple. In the following examples we demonstrate the usage of comparison unary expressions which we'll build upon further in the table expressions section.

#### 4.3.1 Example
`in [42, 100), out (10.1111, 11), == 10`
`1 + 1 < 3`

### 4.4 Pipe Expressions
An expression of the form `'|' tuple` is a pipe expression. The pipe expression is an essential building block of the table expression and is signaled to the parser by the start of the pipe operator. As detailed in the tuple expression section, the single element tuple would be inferred if following the pipe operator.

#### 4.4.1 Example
`| in [42, 100), out (10.1111, 11), == 10`
`| 1 + 1 < 3`

### 4.5 Table Expressions
An expression of the form `'table' tuple ( pipe '=>' expression )+ ( '_' '=>' expression )?` is a table expression. A table expression always begins with the keyword `table` and is followed by a tuple. This tuple is used by the table statement to determine the required length of each pipe tuple as these must match, and the values which should be supplied to the pipe expression. Because a tuple is required to define the table, tables are effectively n-dimensional.

Continuing along the expression, `( pipe '=>' expression )+`, signifies the pattern matching expression and because of `+` at least one pattern is expected. When expanding the pipe, the tuple must be of the same length as the starting table tuple. Each branch is evaluated in descending order and if a match is found the right-hand side of the lambda token is returned. The first matching branch will be returned and no further branches will be evaluated.

The final grammar, `( '_' '=>' expression )?`, makes the table exhaustive, meaning if no match is found then this is the default branch which will be taken. However, because the exhaustive branch is optional, no branch could match the table arguments. In that case we opt to return a null object, rather than throw an exception like would happen in C#.

#### 4.5.1 Example
The following example is table which returns an interval representing thr allowable loan terms when constrained by a borrowers credit score and the age of a vehicle liability.
```
table CreditScore, VehicleAge
 | in [0, 659), in [0, 6] => [12, 24]
 | in [0, 659), in [7, 9] => [12, 24]
 | in [660, 850], in [0, 6] => [12, 120]
 | in [660, 850], in [7, 9] => [12, 96]
 _ => [12, 12]
```

### 4.6 Interval Operations
An expression of the form `expression ( 'in' | 'out' ) expression` is an interval operation. The left-hand side expression must evaluate to either a number or an interval. The right-hand side expression must evaluate to an interval. The `in` keyword

### 4.7 Function Expressions
An expression of the form `'fun' ( IDENTIFIER | DOMAIN ) '=>' expression` is a function expression. A function expression is always preceded by the keyword `fun`. A function name can either be an identifier token or a domain keyword token. The identifier can be any alpha numeric character sequence if it starts with an alpha character. The domain is allowed to be any reserved domain keyword. 

All function names must be unique. Because a function is an expression it can be called where an expression is expected. This allows for reusable expressions.

#### 4.7.1 Example
```
fun LoanAmount =>
  table CreditScore
  | in (720, 850] => 75000
  | in (640, 720] => 40000
  | in (0, 640) => 28000
  _ => 0
```

## 5 Declarations
Declarations are a way to neatly represent all of the various rules, functions, and top-level expressions comprising source code. Declarations are not exactly statements as declarations will not produce any side effects.

### 5.1 Rule Declarations
Rules are a special language construct and aren't classified as an expression. Rules allow for modifications to runtime values and provide an explicit loan denial system. There are two supported rule declarations, deny and adjust.

#### 5.1.1 Rule Deny
A declaration of the form `'rule' 'deny' STRING '=>' expression` is a deny rule. The `STRING` token is human readable text to clearly express the rule and will be used as the display text to the end user. The right-hand side expression is required to evaluate to either true or false.

##### 5.1.1.1 Example
```
rule deny "Reject this loan if credit score is negative!" =>
  CreditScore < 0
```

#### 5.1.2 Rule Adjust
A declaration of the form `'rule' 'adjust' DOMAIN STRING '=>' expression` is an adjust rule. We've introduced a new token to the rule, `DOMAIN`. The `DOMAIN` is the domain property which can be adjusted. The expected return type of the expression is a number. The adjustment will be an arithmetic add operation, so if the adjustment should be a subtraction provide a negative number.

##### 5.1.2.1 Example
```
rule adjust InterestRate "Loans exceeding 100 months should be raised half a point." =>
  LoanTerm > 100 
    ? .5
    : 0
```

## 6 Semantic Analysis
Banking Meta Language has no user defined types, and only a handful of supported types. The base types are `integer`, `floating point`, `string`, and `bool`. The base `floating point` type complies with the IEC 60559:1989 (IEEE 754) standard for binary floating-point arithmetic. Banking Meta Language also has the compound type `interval` which is an aggregation of base types. Another compound type, which is only valid in the context of a `table` expression is the `tuple` type. All types in Banking Meta Language are inferred, meaning that the compiler fully deduces the type without any explicit type annotations. 

There are a couple of language constructs which have types associated to them.
* Functions: All functions have a return type associated. In Banking Meta Language functions are always parameterless so there's no need to worry about function arguments during calls.
* Expressions: All expressions have some type based on the evaluation of it. Because most everything is an expression everything has a type associated to it.

The language grammar and parser do a pretty good job defining the language rules and enforcing the language rules respectively. However there are several rules which require a semantic analysis pass. A note on scoping, Banking Meta Language only has a global scope.

### 6.1 Coercion
Banking Meta Language supports coercion in the direction of an `integer` to a `floating point` and will do so when determing compatible types, e.g. either doing comparison or determing the return type of conditional expression.

### 6.2 Functions Semantic Rules
* Function identifiers must be unique.

### 6.3 Rule Semantic Rules
* Rules have a required `string` token which must be unique between rules.
* Deny rules must have a return type of `bool`.
* Adjust rules must have a return type of either `integer` or `floating point`.

### 6.4 Interval Operation Expression Semantic Rules
* An interval operation has two operands, the left-hand side can either be a `integer`, `floating point` or an `interval` and the right-hand side must be an `interval` type. The interval operation resulting type is always a `bool`.

### 6.5 Ternary Expression Semantic Rules
* A ternary operator has three operands, the condition, an expression when the condition evaluates to true, and an expression for when the condition evaluates to false. The condition operand must be a `bool` type. The expressions can be any type but must be compatible, i.e. one can be coerced to the other.

### 6.6 Logical Expression Semantic Rules
* Logical operators `and` and `or` both have two operands and the two operands must be a `bool` type. The resulting type is always a `bool`.

### 6.7 Equality Expression Semantic Rules
* Any two compatible types can be compared using equality operators. The resulting type is always a `bool`.

### 6.8 Comparison Expression Semantic Rules
* Comparison expressions have two operands. The only valid types for the operands are `integer` and `floating point`. Type coercion will occur in the event the operand types are valid but differ. The resulting type is always a `bool`.

### 6.9 Term Expression Semantic Rules
* Term expressions have two operands. The only valid types for the operands are `integer` and `floating point`. Type coercion will occur in the event the operand types are valid but differ. The resulting type will either be an `integer` or a `floating point`. In the event the operand types differed, the resulting type will be the type which was coerced to.

### 6.10 Unary Expression Semantic Rules
* Unary operator `!` can only be used with a `bool` type and will return a `bool` type.
* Unary operator `-` can be used with either an `integer` or a `floating point` and produces the same type as the operand.
* All other unary operators adhere to the same right-hand side semantics as their binary expression equivalents.

### 6.11 Tuple Semantic Rules
* A tuple can consist of at most 10 element, this limit is artificial and can be expanded in the future if the need arises. A tuple can consist of mixed types. A tuple is itself not an expression, but can consist of expression and is just the `tuple` type during type checking.

### 6.12 Table Expression Semantic Rules
* A table expression consists of an argument which is a `tuple` type, and multiple branches. Each branch consists of a condition and an expression. 
* A branch condition must be `tuple` type. The tuple length of a branch must equal the same tuple length of the table argument. Each expression in the tuple must evaluate to a `bool`. From here each evaluated expression is logically and'ed together and a resulting type of `bool` is produced.
* A branch condition `tuple` may contain unary comparison operators, Banking Meta Language will combine the expression at the same position from the table argument tuple to form a binary expression, where the branch tuple expressions make up the operator and the right-hand side operand.
    * Some further clarity. In the following table expression the argument is a single positional tuple. When evaluating each branch and upon encountering a comparison unary operator a binary expression is formed. So, `>= 90` becomes `Grade >= 90`.
      ```
      table Grade | >= 90 => "A" | >= 80 => "B" ...
      ```
* A table has a single return `type`, and because of this all branch expressions must have compatible types. Like earlier in the event the types differ but type coercion is possible the resulting type is the coerced to type.

### 6.13 Interval Expression Semantic Rules
Please refer to the interval expression section for a breakdown of semantic rules.

## 7 Syntactic Grammar (EBNF)
```
program   ::= declaration+ EOF
declaration 
          ::= function 
            | rule 
            | expression

/*
    Declarations
*/
function  ::= 'fun' ( IDENTIFIER | DOMAIN ) '=>' expression
rule      ::= 'rule' ( deny | adjust )
deny      ::= 'deny' STRING '=>' expression
adjust    ::= 'adjust' rule_chain
rule_chain 
          ::= DOMAIN STRING '=>' expression

/*
    Expressions
*/
expression
         ::= ternary
           | function
           | table
ternary  ::= interval_op ( '?' interval_op ':' interval_op )?
interval_op
         ::= or ( ('in' | 'out' ) or )?
or       ::= and ( 'or' and )*
and      ::= equality ( 'and' equality )*
equality ::= comparison ( ( '!=' | '==' ) comparison )*
comparison 
         ::= term ( ( '>' | '>=' | '<' | '<=' ) term )*
term     ::= factor ( ( '-' | '+' ) factor )*
factor   ::= unary ( ( '/' | '*' ) unary )*
unary    ::= ( '!' | '-' | '>' | '>=' | '<' | '<=' | '!=' | '==' | 'in' | 'out' ) unary | primary
primary  ::= NUMBER
           | STRING
           | DOMAIN
           | IDENTIFIER
           | BOOLEAN
           | interval
           | '{' expression '}'
table    ::= 'table' tuple ( pipe '=>' expression )+ ( '_' '=>' expression )?
/* Not a pure expression, we don't allow nested tuples */
tuple    ::= expression ( ',' expression )*
/* Not a pure expression, pipes are components of tables */
pipe     ::= '|' tuple

/*
    Lexical Grammar
    For unicode character classes https://www.compart.com/en/unicode/category
*/
interval ::= ( '[' | '(' ) expression ',' expression ( ']' | ')' )
BOOLEAN  ::= 'true'
           | 'false'
NUMBER   ::= DIGIT+ ( '.' DIGIT+)?
STRING   ::= '"' '<any character except ">'* '"'
IDENTIFIER 
         ::= ALPHA ( ALPHA | DIGIT )*
ALPHA    ::= '<A Unicode character of classes Lu, Ll, Lt, Lm, Lo, or Nl>'
DIGIT    ::= '0'
           | '1'
           | '2'
           | '3'
           | '4'
           | '5'
           | '6'
           | '7'
           | '8'
           | '9'
DOMAIN   ::= 'CreditScore'
           | 'LoanAmount'
           | 'LoanTerm'
           | 'InterestRate'
           | 'LoanToValuePercent'
           | 'DebtToIncomePercent'
           | 'UnsecuredDebtToIncomePercent'
           | 'PaymentToIncomePercent'
           | 'MinimumMonthlyPayment'
           | 'Mileage'
           | 'VehicleAge'
           | 'WatercraftLength'
           | 'PromotionalInterestRate'
           | 'PromotionalDuration'
           | 'PrimeRateModifies'
           | 'PrimeRateType'
           | 'DrawPeriodInMonths'
           | 'MaxLienPosition'
           | 'LiabilityCount'
           | 'OldestLiabilityInMonths'
           | 'GrossMonthlyIncome'
           | 'GrossMonthlyIncomeRatio'
           | 'TotalUnsecuredDebt'
           | 'TotalUnsecuredDebtInstitutionSpecific'
```

## 8 Resources
Google Scholar aptly summarizes the development approach to the Banking Meta Language, "Stand on the shoulders of giants". Resources made use of in no particular order.

* F# 4.1 Language Specification (https://fsharp.org/specs/language-spec/4.1/FSharpSpec-4.1-latest.pdf)
* EBNF Grammar Visualizer (https://www.bottlecaps.de/rr/ui)
* XQuery 1.0 XML Query Language Specification (Specifically EBNF notation) (https://www.w3.org/TR/2010/REC-xquery-20101214/#EBNFNotation)
* Practical application for building interpreters (https://craftinginterpreters.com/)
* Stanford CS143 Compilers (https://web.stanford.edu/class/cs143/)