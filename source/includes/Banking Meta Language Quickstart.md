# BML Quickstart

## 1 Hello World
Banking Meta Language (BML) is a fully functional language, so everything is a function and every function a composition of expressions! BML is strongly typed, but implicitly typed, so while every function has a type associated to it that association is being determined by the compiler. Without further ado hello, hello world,
    
    `fun hello_world => "Hello World"`

This simple line of code introduces a few important topics.

1. `fun` is the function keyword, meaning everything after this defines a function.
2. `hello_world` is the function name.
3. `=>` is the lambda operator and expects some kind of expression to follow.
4. `"Hello World"` is a string literal and also the function's return value, which means that the hello_world function returns the string type.

Every function is parameterless, so invoking a function is as easy as calling it directly. `hello_world` would print `"Hello World"` in an interpreter. Also you may have observed that there was no function ending symbol, BML does not need to make use of line endings or semi colons. So new lines are not significant and any formatting is provided for readability.

## 2 Function Names
In the hello world example, our function was named `hello_world`. BML supports custom named functions which can be recalled later when composing other functions. BML also has a host of reserved function keywords, which are used to communicate product information. Reserved keywords can be found in the specification document. If you wanted to communicate an interest rate table from a product sheet you would simply define a function for interest rate like so

    `fun InterestRate => ...`

## 3 Tables
### 3.1 Introduction
Tables are the fundamental building block of BML. Tables are a language construct which are meant to merge switch statements found in most programming languages with tables traditionally found in a banking product sheet. The following code introduces a large chunk of functionality.

```
fun LoanAmount =>
  table CreditScore
  | in (720, 850] => 75000
  | in (640, 720] => 40000
  | in (0, 640) => 28000
  _ => 0
```

1. `fun LoanAmount =>` is already familiar, this function defines the loan amount for a product.
2. `table` a table ahead! 
3. `CreditScore` following the table keyword are the table's parameters. So, this table uses the credit score of the applicant to find a loan amount.
4. `| in (720, 850] => 75000` is a branch of the table, the branch starts with the pipe operator `|`.
5. `in (720, 850]` is an interval operation. Read more about intervals [here](#4-type-system). This is converted to `CreditScore in (720, 850]`. This is some syntactic sugar provided by the language.
6. `=> 75000` then we have the expression which is returned if the branch arm evaluates to true.
7. The same follows for the other two branches.
8. `_ => 0` this is the exhaustive branch. If no branch is found to be true, then you can provide a default value. The exhaustive branch is not required, however.

An example let's say the credit score of the applicant is 680. Branches are evaluated in order so, the first branch will be tested, `680 in (720, 850]`. This is false so we go to the next branch, `680 in (640, 720]`. This is true! So, the table returns 40000, which in turn is used as the loan amount.

### 3.2 A More Involved Example
Let's look at the following code 

```
fun InterestRate =>
  table CreditScore, LoanTerm
  | >= 720, > 72 => 5.4
  | >= 720, <= 72 => 4.9
  | in (600, 720], <= 60 => 6.3
  | <= 599, true => 7.0
```

So, we've introduced a couple of things, multiple table parameters `table CreditScore, LoanTerm`, unary comparison operators `>= 720`, and a boolean literal `true`. 

This table is more representative of a traditional 2-dimensional table with columns and rows, but BML supports tables in 10 dimensions, meaning you can supply 10 parameters to a table if you so choose. Each table branch must have the same number of expressions as table parameters. Additionally, the parameters are applied in order, so the branch `| >= 720, > 72 ` gets converted to `CreditScore >= 720 and LoanTerm > 72`. I just introduced another important keyword `and`. `and` is the boolean logical AND. Each branch expression must be true for the branch to evaluate to true.

Unary comparison operators in a table branch are just syntactic sugar for a comparison expression and are only valid in the context of a table branch.

Finally, `true`, as seen in the last branch, is a literal value. The final branch becomes `CreditScore <= 599 and true`. This means that an applicant with a credit score less than or equal to 599 and any loan term qualifies for a 7.0% interest rate.

## 4 Type System
BML has only a few primitive types, `integer`, `floating point`, `string`, and `bool`, and has the compound types `interval` and `progression` which are aggregations of base types.

`integer` is any whole number between -2,147,483,648 to 2,147,483,647. e.g., `12`

`floating point` is any decimal number between positive 79,228,162,514,264,337,593,543,950,335 to negative 79,228,162,514,264,337,593,543,950,335. e.g., `42.24`

`string` is a collection of characters in between double quotes, e.g., `"Hello"`

`bool` is a boolean value either `true` or `false` and is the result of any comparison or equality operation.

`interval` is anything which takes the form of an opening character `[` or `(` some `value`, a `comma` another `value` and then a closing character `)` or `]`. Some examples, `[100, 300]`, `(-123, 4.3]`. The first value must be smaller than the second value. The bracket `[`, `]` means that the number is included in the range. The parenthesis `(`, `)` means that the number is excluded in the range.

`progression` is anything which takes the form `%` `integer` `interval`. Some examples, `%12 [12, 84]`, `%2 [4, 400)`. The integer must be larger than zero, and the interval must only contain `integer` types. The interval also must start with an inclusive bound. A progression allows for an expansion of the values within it. The `integer` following the `%` is the common difference and beginning with the interval start is summed by the common difference until the end of the interval is reached.
* `%12 [12, 84]` Starting with 12 and then incrementing by 12 until the end of the interval is reached
  * `12`
  * `12 + 12 = 24`
  * `24 + 12 = 36`
  * ...
  * `72 + 12 = 84` 84 is inclusive so it is included
  * `84 + 12 = 96` which is out of the interval bounds and expansion stops
  * The expanded progression contains the values `12, 24, 36, 48, 60, 72, 84`

## 5 Comparison Operators
BML supports the comparison operators `>`, `>=`, `<`, `<=`, `in`, and `out`. Hopefully the first four are already familiar to you. `in` and `out` are exclusive to checking whether or not a number is `in` an interval or `out` of an interval. As an example, 

`1 in [0, 100] // true`

`1 in (1, 2) // false`

`4 out [1, 2] // true`

the `in` and `out` operators can also be used to compare two intervals. As an example,

`[100, 120] in (1, 300) // true` 

## 6 Equality Operators
BML supports the equality operators `==` and `!=`. Any two compatible types can be compared using equality operators. The resulting type is always a `bool`. As an example,

`1 == 1 // true`
`4.1 != 4 // true`
`"hello" == "goodbye" // false`

## 7 Type Coercion
Banking Meta Language supports coercion in the direction of an `integer` to a `floating point` and will do so when determining compatible types, e.g., either doing a comparison or determining the return type of a conditional expression. 
BML also supports the coercion in the direction of `integer` to `progression` and `interval` to `progression`. 
* An `integer` when coerced to a progression represents a single value. E.g. `4` becomes `%1 [4, 5)`. 
* An `interval` can only be coerced when the start of the `interval` is inclusive, `[`, and both expressions in the interval are `integers`. An `interval` when coerced is given the default common difference of `1`. E.g. `[4, 20)` becomes `%1 [4, 20)`.

## 8 Underwriting Guidelines/Rules
BML supports underwriting guidelines in the form of rules. There are two rule types `Deny` and `Adjust`.

### 8.1 Deny Rules
A deny rule looks very similar to a function

```
rule deny "Reject this loan if credit score is negative!" =>
  CreditScore < 0
```

1. `rule` is the rule keyword, meaning everything after this defines a rule.
2. `deny` is the rule type keyword, this rule is a deny rule.
3. `"Reject this loan if credit score is negative!"` is the rule name, the rule name is a string literal and acts as a user facing message when the rule evaluates to true.
4. `=>` lambda operator, expects an expression to follow.
5. `CreditScore < 0` is an expression which when evaluated returns a boolean.

So, if `CreditScore` is equal to -10 then the loan is denied! If the `CreditScore` is 100 then for the purposes of this rule the loan can continue processing. This does not indicate that the loan is approved, however.

Deny rules always expect a return type of boolean. This indicates whether or not the applicant should be denied for this loan type. When the deny rule evaluates to true, the loan is denied, otherwise the loan can continue processing.

### 8.2 Adjust Rules
Adjust rules are syntactically similar the deny rule but introduce another item. They're used to perform adjustments to a loan's properties.

```
rule adjust InterestRate "Loans exceeding 100 months should be raised half a point." =>
  LoanTerm > 100 
    ? .5
    : 0
```

1. `rule` is the rule keyword, meaning everything after this defines a rule.
2. `adjust` is the rule type keyword, this rule is an adjustment rule.
3. `InterestRate` a keyword which specifies which loan property is being adjusted.
4. `=>` lambda operator, expects an expression to follow.
5. `LoanTerm > 100 ? .5 : 0` finally the rule expression.

So we've introduced ternary operators. A ternary operator is of the form `condition ? consequent : alternative`. If the condition, in this case `LoanTerm > 100` evaluates to true, then the `consequent` is chosen, otherwise the `alternative` is taken. So, if the loan currently being processed has a loan term of 120, then the condition `120 > 100` evaluates to true and the adjustment evaluates to positive `.5`. This adjustment is then added to the interest rate, So, if the current interest rate for the processing loan is `4.3` then it becomes `4.8`.

If you want to decrease the interest rate for someone who has an excellent credit score and it's more convenient than putting it in a table, the consequent would be a negative value like `-0.1`.

## 9 Full Example

```
fun MaxLoanToValuePercent =>
  table CreditScore
  | in [659, 669] => 70
  | in [670, 739] => 70
  | in [740, 850] => 90

fun MaxDebtToIncomePercent =>
  table CreditScore
  | in [659, 669] => 50
  | in [670, 739] => 55
  | in [740, 850] => 60

fun MaxPaymentToIncomePercent =>
  table CreditScore
  | in [659, 669] => 200
  | in [670, 739] => 210
  | in [740, 850] => 220

fun MaxLoanAmount =>
  table CreditScore
  | > 800 => 350000
  |  in [659, 800] => 300000

fun AllowableLoanTerms =>
  table LoanAmount, CreditScore
  | > 5000, in [659, 850] => %12 [0, 60]

fun InterestRate =>
  table CreditScore, LoanTerm
  | in [740, 850], in [12, 60] => 3.1
  | in [700, 739], in [24, 60] => 4.1
  | in [670, 699], in [36, 60] => 5.1
  | in [659, 669], in [48, 60] => 6.1

rule adjust InterestRateAdjustment "Rate Adjustment - credit score between 700 - 850 and LTV >= 80% and <= 90%" =>
  CreditScore in [700, 850] and LoanToValuePercent in [80, 90]
  ? 2.0
  : 0

rule deny "The applicant's credit score is less than or equal to 739, and the loan term equals 60 months. The annual unsecured debt ratio must be less than 35%" =>
  CreditScore <= 739 and LoanTerm == 60 and TotalUnsecuredDebt > 35

rule deny "There are less than 2 open tradelines or the oldest tradeline is less than or equal to 24 months old" =>
  LiabilityCount < 2 and OldestLiabilityInMonths <= 24
```

## 10 Additional Information
For more detailed descriptions of the various language constructs, the supporting document `Banking Meta Language Specification` can be viewed. For any other questions you can reach out to Affordit directly.