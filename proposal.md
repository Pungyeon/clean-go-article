# Proposal for speaking at GopherCon 2019 

# Clean Golang Code

## Abstract
Learn how the core principles of writing clean code and how to refactor Golang, to produce a maintainable and stable code base.

## Talk description
The talk will introduce the ideas behind writing clean Golang code, as well as looking more specifically at refactoring already existing Golang code. The talk will be structured as such:

* [Introduction to Clean Code](#Introduction-to-Clean-Code)
    * Test Driven Development
    * Clean Functions & Writing Prose

* [Clean Golang](#Refactoring-Code)
    * Let's talk about `interface{}`
    * Spotting Code Smell
    * Refactor examples

### Introduction to Clean Code
Clean Code, is the pragmatic concept of ensuring that code is readable, maintanable. Clean Code establishes trust in the codebase, and will derive developers from making silly mistakes or introducing bugs. Clean Code will also establish much more stability in development speed, which typically will take a nose dive in the later stages of projects, due to higher risk of increasing bugs as the codebase expands. 

#### Test Driven Development
The core of creating clean code stems from creating good tests. Writing good tests helps create clean code, as it invites developers to think about the outcomes and test coverage of functions / functionality. It's easier to test a function that is only 4 lines, rather than a function, which is 40. In the same manner, a function which is 4 lines, is typically easier to understand than a function of 40 lines. Therefore, when using test driven development, the resulting code is much more likely to be of a cleaner nature. 

The next important part of test driven development, which is very closely related to clean code, is the TDD cycle:

1. Write a test which fails
2. Make the test pass
3. Refactor code
4. Repeat

Step three of the cycle, ensures that we can refactor our code as we are writing it. The tests ensure that our refactor doesn't change the outcome of our functions and we can therefore, essentially, go crazy refactoring our code to be as clean as possible. As we go along, and our codebase expands, we will still have our tests, to make sure that our refactoring will not affect the outcome of our functions.

#### Cleaning Functions
In the words of Robert C. Martin: 

> "How small should a function be? Smaller than that!"

When writing clean code, our primary goal is to make our code easily digestable. The most effective way to do this, is to make our functions as small as possible. It's important to understand, that this is not to avoid code duplication, the actual reason for this is to heighten the code comprehension. Another way of explaining this, is to look at a function description: 

```
fn GetOrder:
    - parse json input for order id
    - get user from context
    - check user has appropriate role
    - get order from database
```

When using small functions (typically 5-8 lines in Golang), we can create a function that reads almost as easily as our description:

```go
var (
    NullOrder = Order{}
    ErrInsufficientPrivliges = errors.New("user does not have sufficient priviliges")
)

func GetOrder(ctx context.Context, json []bytes) (Order, error) {
    order, err := NewOrderFromJSON(json)
    if err != nil {
        return NullOrder, err
    }
    if GetUserFromContext(ctx).IsAdmin() {
        return db.GetOrder(order.ID)
    }
    return NullOrder, ErrInsufficientPrivliges
}
```

Using smaller functions also has a side-effect of eliminating another horrible habit of writing code: indentation hell. Indentation hell, typically occurs when a chain of if statements are clumsily inserted into a function. This makes the code very, very difficult to parse and should be eliminated whenever spotted. This is particularly common when working with `interface{}` and using type casting:

```go
func GetCallIfActive(reference string) (Call, err) {
    refIface, err := db.KVStore.Get(reference)
    if err != nil {
        return EMPTY_CALL, err
    }
    if refobj, ok := refIface.(CallReference); ok {
        callIface, err := db.KVStore.Get(refobj.CallID)
        if err != nil {
            return EMPTY_CALL, err
        }
        if call, ok := callIface.(Call); ok {
            if call.Active {
                return Call, nil
            }
        } else {
            return EMPTY_CALL, errors.New("could not cast interface to call")
        }
    }
    return EMPTY_CALL, errors.New("could not cast interface to reference")
}
```

### Clean Golang
#### Let's talk about `interface{}`
Before starting with refactoring examples, this section will discuss an important topic within Clean Code, when writing Golang code: the use of `interface{}`. The empty interface struct in Golang is extremely powerful, but can also make it extremely easy to write unreadable and unstable code. The empty interface struct enables gophers to write weakly typed code, which can be extremely beneficial in certain cases. 

As an example, the `json.Unmarshal` function, use of the empty interface struct, is fine. It's not great, but there aren't really any alternatives, without generics being implemented. 

The general rule of using the empty interface struct, is to use it as an input type, but never to use it as an output type for a function. So whereas, this is maintanable:

```go
func Unmarshal(i interface{}) error { ... }
```

This isn't:

```go
func SmellyBoi() (interface{}, error) { ... }
```

The main difference is, that when parsing an interface as input to a function, we know the type ahead of time. We won't have to do any type casting (other than inside the function). Our 'smell' is contained and won't have to affect the developer using this function, which is important.

However, returning `interface{)` is a different story. Not only, is it extremely difficult to determine the actual output for other developers, but this forces developers to use type casting, which can cause many unexpected results and errors. We don't know the type ahead of time and the only real way to understand what might be returned, is by reading the implementation of the function. This should be avoided at all costs.

Instead, a more reasonable way of resolving this, is by ensuring that we return an interface, which has some kind of contract associated with it.

```go
func CleanerBoi() (*io.Reader, error) { ... }
```

We know have something concrete to work with. We don't have to do type casting and we know how we can interact with the returning object.