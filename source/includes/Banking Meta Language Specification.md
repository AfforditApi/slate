# BML Specification
## 1 Introduction
The Banking Meta Language is a functional, non-Turing complete, strongly typed language used to represent a financial institution’s product catalog which includes dynamic rate adjustments and explicit loan denial.

The language is intentionally non-Turing complete to avoid the halting problem.

Once Banking Meta Language source code has been parsed and determined to be valid, it must be executed. We leverage existing tools in the .NET ecosystem, for example the Roslyn scripting API, to translate our parsed expressions into C# code which can then be compiled and executed all at run time. This provides us with multiple benefits. the largest being we can leverage the existing C# compiler to do all the dirty low-level work for us.

An example of some Banking Meta Language code (hopefully it's easy to grok).

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
| A &#124; B | Matches A or B, but not both |
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
Banking Meta Language supports simple single line comments of the form '//'. The lexer when encountering this sequence of characters will consume all the tokens until the next line is reached, ignoring the comment.

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
| Pipe | &#124; |
| Comma | , |
| LeftBrace | { |
| RightBrace | } |
| Question | ? |
| Colon | : |
| Discard | _ |
| Percent | % |

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
| Identifier | [a-zA-Z] [_a-zA-Z0-9]* |
| String | '"' [^"]* '"' |
| Integer | [0-9]+ |
| FloatingPoint | [0-9]+ '.' [0-9]+ |

Strings are allowed to be multi line. The scanner when consuming a string will consume any character until it reaches a terminating double quote character, '"'. There is currently not support for escaping the double quote.

### 3.7 Special Tokens
| Token Type | Character Sequence |
| ---------- | ------------------ |
| Eof        | Special token indicating that the end of the character stream has been reached. |

### 3.8 Domain Identifiers
Domain identifiers are special function identifiers that correspond either to functions which provide input or which can be defined to provide output. Input functions fall into two categories: those that can be referenced in an `adjust` rule and those that cannot. Output functions also fall into two categories: those that must be defined and those which are optional. Optional output functions have default values, which are specified below.

As an example, `fun InterestRate => 2.75` defines the required output function `InterestRate`,while `CreditScore > 750 ? -0.15 : 0` calls the non-adjustable input function `CreditScore`. More specific behavior regarding domain keywords and functions can be found in the function expressioni section.

```
domain_identifier
  ::= input identifier             | output identifier
input_identifier
  ::= adjustable identifier        | nonadjustable identifier
output_identifier
  ::= required_identifier          | optional_identifier
adjustable_identifier
  ::= 'LoanAmount'                 | 'LoanTerm'                     | 'InterestRateAdjustment'
nonadjustable_identifier
  ::= 'CreditScore'                | 'LoanToValuePercent'           | 'DebtToIncomePercent'          
  | 'UnsecuredDebtToIncomePercent' | 'PaymentToIncomePercent'       | 'Mileage'
  | 'VehicleAge'                   | 'WatercraftLength'             | 'LiabilityCount'
  | 'OldestLiabilityInMonths'      | 'GrossMonthlyIncome'           | 'GrossMonthlyIncomeRatio'
  | 'TotalUnsecuredDebt'           | 'TotalUnsecuredDebtInstitutionSpecific'
required_identifier
  ::= 'InterestRate'
optional_identifier
  ::= 'MinimumMonthlyPayment'      | 'PromotionalInterestRate'      | 'PromotionalDuration'
  | 'PrimeRateModifies'            | 'PrimeRateType'                | 'DrawPeriodInMonths'
  | 'MaxLienPosition'              | 'CompoundingPeriodsPerYear'    | 'AllowableLoanTerms'
  | 'MaxLoanToValuePercent'        | 'MaxDebtToIncomePercent'       | 'MaxPaymentToIncomePercent'
``` 

## 4 Expressions
Expressions are the fundamental building block of Banking Meta Language and are expected to evaluate eventually to either a number, a string, a boolean, a domain keyword, an identifier, or an expression grouping. Almost everything is an expression.

### 4.1 Interval Expressions
An expression of the form `( LeftBracket | LeftParen ) expression Comma expression ( RightBracket | RightParent )` is an interval expression. Interval expressions are used to represent an interval between two numbers, as in Mathematics. 
#### 4.1.1 Example
`[42, 100)`
`(10.1111, 11)`

### 4.2 Unary Expressions
An expression of the form `( Bang | Minus | Greater | GreaterEqual | Less | LessEqual | BangEqual | EqualEqual | In | Out ) unary | primary` is a unary expression. We deviate a bit from the traditional unary operators `!` and `-` to also include comparison operators. This change exists to have valid tuple expressions in our table language construct.

### 4.3 Tuple Expressions
An expression of the form `expression ( Comma expression )*` is a tuple expression. This allows for any number of expressions to be joined by a comma. A tuple can be expressed as a single expression and will be inferred depending on usage. Banking Meta Language has no support for empty tuples also referred to as the 0-tuple. In the following examples we demonstrate the usage of comparison unary expressions which we'll build upon further in the table expressions section.

#### 4.3.1 Example
`in [42, 100), out (10.1111, 11), == 10`
`1 + 1 < 3`

### 4.4 Table Expressions
An expression of the form `Table tuple ( Pipe tuple Lambda expression )+ ( Discard Lambda expression )?` is a table expression. A table expression always begins with the keyword `table` and is followed by a tuple. This tuple is used by the table statement to determine the required length of each pipe tuple as these must match, and the values which should be supplied to the pipe expression. Because a tuple is required to define the table, tables are effectively n-dimensional.

Continuing along the expression, `( Pipe tuple Lambda expression )+`, signifies the pattern matching expression and because of `+` at least one pattern is expected. When expanding the pipe, the tuple must be of the same length as the starting table tuple. Each branch is evaluated in descending order and if a match is found the right-hand side of the lambda token is returned. The first matching branch will be returned and no further branches will be evaluated.

The final grammar, `( Discard Lambda expression )?`, makes the table exhaustive, meaning if no match is found then this is the default branch which will be taken. However, because the exhaustive branch is optional, no branch could match the table arguments. In that case we opt to return a null object, rather than throw an exception like would happen in C#.

#### 4.4.1 Example
The following example is table which returns an interval representing thr allowable loan terms when constrained by a borrowers credit score and the age of a vehicle liability.

```
table CreditScore, VehicleAge
 | in [0, 659), in [0, 6] => [12, 24]
 | in [0, 659), in [7, 9] => [12, 24]
 | in [660, 850], in [0, 6] => [12, 120]
 | in [660, 850], in [7, 9] => [12, 96]
 _ => [12, 12]
```

### 4.5 Interval Operations
An expression of the form `expression ( In | Out ) expression` is an interval operation. The left-hand side expression must evaluate to either a number or an interval. The right-hand side expression must evaluate to an interval.

### 4.6 Ternary Expressions
A ternary expression has the syntax `interval_op ( Question interval_op Colon interval_op )?`. A ternary expression in BML is very similar to the ternary experessions found in most programming languages. More details are provided in the semantics section below.

### 4.7 Progression Expressions
A progression expression has the form `Percent expression interval`. A progression allows you to specify the set of integers in an arithmetic progression that fall within an interval. A progression can be thought of as a list comprehension or a generator function. The expression following the percent is the common difference. Beginning with the interval's starting value, the common difference is added to it and then each subsequent value until the interval's final bound is reached.

### 4.7.1 Example
Looking at the first return progression `%12 [36, 84]`, this gets expanded to `36, 48, 60, 72, 84` as these are the numbers starting with 36 which are additions of 12.

```
fun AllowableLoanTerms =>
    table CreditScore
    | in (720, 850] => %12 [36, 84]
    | in (640, 720] => %12 [36, 60]
    | in (0, 640) => 36
    _ => 0
```

## 5 Declarations
Declarations are a way to neatly represent all of the various rules and functions comprising source code. Declarations are not statements, as declarations will not produce any side effects.

### 5.1 Function Declarations 
A function is declared using the syntax `Fun ( Identifier | output_identifier ) Lambda expression`. A function declaration always begins with the keyword `fun`. The next element can either be an Identifier token or an output_identifier phrase.

All function names must be unique. A function can be called in any expression context.

#### 5.1.1 Example
```
fun LoanAmount =>
  table CreditScore
  | in (720, 850] => 75000
  | in (640, 720] => 40000
  | in (0, 640) => 28000
  _ => 0
```

### 5.2 Rule Declarations
Rules are a special language construct and aren't classified as an expression. Rules allow for modifications to runtime values and provide an explicit loan denial system. There are two supported rule declarations, deny and adjust.

#### 5.2.1 Rule Deny
A declaration of the form `Rule Deny String Lambda expression` is a deny rule. The `String` token is human readable text to clearly express the rule and will be used as the display text to the end user. The right-hand side expression is required to evaluate to either true or false.

##### 5.2.1.1 Example
```
rule deny "Reject this loan if credit score is negative!" =>
  CreditScore < 0
```

#### 5.2.2 Rule Adjust
A declaration of the form `Rule Adjust AdjustableIdentifier String Lambda expression` is an adjust rule, where `adjustable_identifier` is the input function to be adjusted. The required return type of the expression is a number. The adjustment is performed using addition, so if the adjustment is negative, provide a negative number.

##### 5.2.2.1 Example
```
rule adjust InterestRateAdjustment "Loans exceeding 100 months should be raised half a point." =>
  LoanTerm > 100 
    ? .5
    : 0
```

## 6 Semantic Analysis
Banking Meta Language has no user defined types, and only a handful of supported types. The base types are `integer`, `floating point`, `string`, and `bool`. The base `floating point` type complies with the IEC 60559:1989 (IEEE 754) standard for binary floating-point arithmetic. Banking Meta Language also has the compound type `interval` which is an aggregation of base types. Another compound type, which is only valid in the context of a `table` expression is the `tuple` type. All types in Banking Meta Language are inferred, meaning that the compiler fully deduces the type without any explicit type annotations. 

There are a couple of language constructs which have types associated to them.
* Functions: All functions have a return type associated. In Banking Meta Language functions are always parameterless so there's no need to worry about function arguments during calls.
* Expressions: All expressions evaluate to a single type. So every expression has an associated type.

The language grammar and parser do a pretty good job defining the language rules and enforcing the language rules respectively. However there are several rules which require a semantic analysis pass. A note on scoping, Banking Meta Language only has a global scope.

### 6.1 Coercion
* Banking Meta Language supports coercion in the direction of an `integer` to a `floating point` and will do so when determining compatible types, e.g., either doing a comparison or determining the return type of a conditional expression.
* BML also supports coercion in the direction of `integer` to `progression` and `interval` to `progression`.
  * When an `integer` is coerced to a `progression`, the `progression` becomes `%1 [N, N+1)` where `N` is the coerced `integer`. This effectively represents a single item equal to the `integer`.
  * When an `interval` is coerced to a `progression` the `progression` becomes `%1, interval`. The coercion is only valid if the opening delimeter of the interval is '[', and expressions in the interval are `integer`s.

### 6.2 Functions Semantic Rules
* Function identifiers must be unique.

### 6.3 Rule Semantic Rules
* Rules have a required `string` token which must be unique between rules.
* Deny rules must have a return type of `bool`.
* Adjust rules must have a return type of either `integer` or `floating point`.
* Adjust rules are guaranteed by the language runtime to be executed before any other rules or functions. Thus any reference to an `adjustable_identifier` in a deny rule or in a function will evaluate to the _adjusted_ value.

### 6.4 Interval Operation Expression Semantic Rules
* An interval operation has two operands, the left-hand side can either be an `integer`, `floating point` or an `interval` and the right-hand side must be an `interval`. The resulting type from an interval operation is always `bool`.

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
* A table has a single return type, and because of this all branch expressions must have compatible types. In the event the types differ but type coercion is possible, the resulting type is the coerced to type.

### 6.13 Interval Expression Semantic Rules
Let the two sub-expressions in an interval be referred to as *X1* and *X2*. Let us call these the _endpoints_ of the interval. In order to be valid, an interval expression must satisfy *X1 < X2*. Thus the empty interval is not valid. The opening and closing characters indicate if an interval's endpoint is included or excluded, as in Mathematics. If *n* is any `integer`, then the following table defines if *n* is included in an interval.

| Expression | Meaning |
| - | - |
| *[ X1,*  | *n >= X1* |
| *( X1,*  | *n > X1* |
| *, X2 ]* | *n <= X2* |
| *, X2 )* | *n < X2* |

### 6.14 Progression Expression Semantic Rules
* A progression starts with a percent character, `%`, followed be an expression which must be of type `integer`. BML does not support `floating point` common differences. A progression then expects an `interval`. The `interval` must start with an inclusive bound and each _endpoint_ of the interval must be an `integer`.
* A progression when expanded requires that there is at least one item, and the expansion does not exceed 1000 items.

### 6.15 Domain Identifier Types
The types of all `domain_identifier` functions are given in the following table

| Name                                  | Type             | Category              | Default Value |
| -                                     | -                | -                     | -             |
| LoanAmount                            | `floating point` | input, adjustable     | N/A           |
| LoanTerm                              | `integer`        | input, adjustable     | N/A           |
| CreditScore                           | `integer`        | input, non-adjustable | N/A           |
| LoanToValuePercent                    | `floating point` | input, non-adjustable | N/A           |
| DebtToIncomePercent                   | `floating point` | input, non-adjustable | N/A           |
| UnsecuredDebtToIncomePercent          | `floating point` | input, non-adjustable | N/A           |
| PaymentToIncomePercent                | `floating point` | input, non-adjustable | N/A           |
| Mileage                               | `integer`        | input, non-adjustable | N/A           |
| VehicleAge                            | `integer`        | input, non-adjustable | N/A           |
| WatercraftLength                      | `integer`        | input, non-adjustable | N/A           |
| LiabilityCount                        | `integer`        | input, non-adjustable | N/A           |
| OldestLiabilityInMonths               | `integer`        | input, non-adjustable | N/A           |
| GrossMonthlyIncome                    | `floating point` | input, non-adjustable | N/A           |
| GrossMonthlyIncomeRatio               | `floating point` | input, non-adjustable | N/A           |
| TotalUnsecuredDebt                    | `floating point` | input, non-adjustable | N/A           |
| TotalUnsecuredDebtInstitutionSpecific | `floating point` | input, non-adjustable | N/A           |
| InterestRate                          | `floating point` | output, required      | N/A           |
| InterestRateAdjustment                | `floating point` | output, optional      | 0             |
| AllowableLoanTerms                    | `progression`    | output, optional      | %1 [0, 1)     |
| MaxLoanAmount                         | `floating point` | output, optional      | 0             |
| MaxLoanToValuePercent                 | `floating point` | output, optional      | 0             |
| MaxDebtToIncomePercent                | `floating point` | output, optional      | 0             |
| MaxPaymentToIncomePercent             | `floating point` | output, optional      | 0             |
| CompoundingPeriodsPerYear             | `integer`        | output, optional      | 12            |
| MinimumMonthlyPayment                 | `floating point` | output, optional      | 0             |
| PromotionalInterestRate               | `floating point` | output, optional      | 0             |
| PromotionalDuration                   | `integer`        | output, optional      | 0             |
| PrimeRateModifies                     | `boolean`        | output, optional      | false         |
| PrimeRateType                         | `string`         | output, optional      | "Fed"         |
| DrawPeriodsInMonths                   | `integer`        | output, optional      | 0             |
| MaxLienPosition                       | `integer`        | output, optional      | 2             |

The meaning of each of the domain_identifiers is given in the following table.

| Name                                  | Description |
| -                                     | -           |
| LoanAmount                            | The amount of the loan. |
| LoanTerm                              | The total number of payments over the life of the loan, if it is structured. |
| CreditScore                           | A credit score for the borrower. The specific credit score product, or combination thereof (e.g. Tri Merge), is outside the scope of this specification. |
| LoanToValuePercent                    | The ratio of the loan value to asset appraisal value, expressed as a percentage. |
| DebtToIncomePercent                   | The ratio the borrowers total monthly payments towards debt and their monthly income, expressed as a percentage. |
| UnsecuredDebtToIncomePercent          | The ratio of the borrowers monthly payment towards unsecured debt and their their monthly income, expressed as a percentage. |
| PaymentToIncomePercent                | The ratio of the payment towards this loan and the borrowers monthly income, expressed as a percentage. |
| Mileage                               | The mileage of the vehicle that this loan is secured by. |
| VehicleAge                            | The age of the vehicle, measured in years, that this loan is secured by. Note that only full years are counted, so if a vehicle is 14 months old, it would be considered 1 year old. |
| WatercraftLength                      | The length of the watercraft that this loan is secured by. If this loan is not a watercraft loan then this function returns 0. |
| LiabilityCount                        | The total number of liabilities that a borrower has. |
| OldestLiabilityInMonths               | The number of months that the oldest liability has been open. |
| GrossMonthlyIncome                    | The total gross monthly income of the borrower. |
| GrossMonthlyIncomeRatio               | The ratio of this loan's amount to gross monthly income. |
| TotalUnsecuredDebt                    | The value of a borrowers unsecured debt. |
| TotalUnsecuredDebtInstitutionSpecific | The total value of a borrowers unsecured debt that is held by the current financial institution. |
| InterestRate                          | The interest rate of this loan, stated as nominal interest per year. |
| InterestRateAdjustment                | Provides a hook to the adjustment rules to adjust the interest rate. |
| AllowableLoanTerms                    | The allowable loan terms for a product. |
| MaxLoanAmount                         | The maximum allowed value for a loan. |
| MaxLoanToValuePercent                 | The maximum allowed value for the loan to value percentage. |
| MaxDebtToIncomePercent                | The maximum allowed value for the debt to income percentage. |
| MaxPaymentToIncomePercent             | The maximum allowed value for the payment to income percentage. |
| CompoundingPeriodsPerYear             | The number of compounding periods for this loan in one year. |
| MinimumMonthlyPayment                 | The minimum amount that can be paid on an unstructured loan. |
| PromotionalInterestRate               | The promotional interest rate on this loan, expressed as nominal interest per year. |
| PromotionalDuration                   | The number of payment periods which are subject to the PromotionalInterestRate. |
| PrimeRateModifies                     | If true then the current _prime rate_ will be added to InterestRate to determine the final interest rate of the loan. |
| PrimeRateType                         | The type of _prime rate_ to use when modifying the InterestRate. Currently, only "Fed" is supported, which refers to the Federal Reserve Bank prime loan rate. |
| DrawPeriodsInMonths                   | If this loan is a HELOC, then this is the number of months in its draw period. |
| MaxLienPosition                       | The highest lien position that this loan can occupy, if it is secured by a home. |

## 7 Resources
Google Scholar aptly summarizes the development approach to the Banking Meta Language, "Stand on the shoulders of giants". Resources made use of in no particular order.

* F# 4.1 Language Specification (https://fsharp.org/specs/language-spec/4.1/FSharpSpec-4.1-latest.pdf)
* EBNF Grammar Visualizer (https://www.bottlecaps.de/rr/ui)
* XQuery 1.0 XML Query Language Specification (Specifically EBNF notation) (https://www.w3.org/TR/2010/REC-xquery-20101214/#EBNFNotation)
* Practical application for building interpreters (https://craftinginterpreters.com/)
* Stanford CS143 Compilers (https://web.stanford.edu/class/cs143/)