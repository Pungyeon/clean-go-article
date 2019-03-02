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

Using smaller functions also has a side-effect of eliminating another horrible habit of writing code: indentation hell. Indentation hell, typically occurs when a chain of if statements are clumsily inserted into a function. This makes the code very, very difficult to parse (for human beings) and should be eliminated whenever spotted. This is particularly common when working with `interface{}` and using type casting:

```go
func GetItemIfActive(extension string) (Item, error) {
    if refIface, ok := db.ReferenceCache.Get(extension); ok {
        if ref, ok := refIface.(string); ok {
            if itemIface, ok := db.ItemCache.Get(ref); ok {
                if item, ok := itemIface.(Item); ok {
                    if item.Active {
                        return Item, nil
                    } // return no item active
                } // return cast error on item interface
            } // return no item found in cache by reference
        } // return cast error on reference 
    }
    return EMPTY_ITEM, errors.New("reference not found in cache")
}
```

Not only, can this kind of code result in a very bad IDE experience for some programmers (due to line length), it's also almost impossible to track the flow of the code. It has to be noted, that go is not particularly likely to produce this kind of code, compared to languages with `try` `catch` blocks or typical callback implementations, but this still should be avoided at all costs. Please, keep in mind that the code above is a small example, with very simple code. As soon as it becomes more complex, the sphagetti factor of the code, grows exponentially. 

So, how do we clean this? Well, it's actually quite simple. The first iteration, is to ensure that we are returning an error as soon as we can. Once we are done with this, we can split up our function into smaller functions as mentioned previously: 

```go
func GetItemIfActive(extension string) (Item, error) {
    refIface, ok := db.ReferenceCache.Get(extension)
    if !ok {
        return EMPTY_ITEM, errors.New("reference not found in cache")
    }

    if ref, ok := refIface.(string); ok {
        // return cast error on reference 
    }

    if itemIface, ok := db.ItemCache.Get(ref); ok {
        // return no item found in cache by reference
    }

    if item, ok := itemIface.(Item); ok {
        // return cast error on item interface
    }

    if !item.Active {
        // return no item active
    }

    return Item, nil
}
```

And on second iteration, we will get something similar to the following:

```go
func GetItemIfActive(extension string) (Item, error) {
    if ref, ok := getReference(extension) {
        // return cast error on reference 
    }
    return getActiveItemByReference(ref)
}

func getItemByReference(extension string) (string, bool) {
    refIface, ok := db.ReferenceCache.Get(extension)
    if !ok {
        return EMPTY_ITEM, false
    }
    return refIface.(string)
}

func getActiveItemByReference(reference string) (Item, ) {
    item, ok := getItemByReference(reference)
    if !item.Active || !ok {
        return EMPTY_ITEM, false
    }
    return Item, nil
}

func getItemByReference(reference string) (Item, bool) {
    if itemIface, ok := db.ItemCache.Get(ref); ok {
        return EMPTY_ITEM, false
    }
    return itemIface.(Item), true
}
```
> For production code, one should elaborate on the code even further, by returning errors instead of a `bool` values. This makes it much easier to understand where the error is originating from. However, as these are just example functinos, the `bool` values will suffice for now.  

Now, this is many more lines of code than our first iteration. However, the code is so much easier to read. It's layered in an onion-style fashion, where we can ignore code that we aren't interested in knowing the details of and diving deeper into the functions that we wish to know the workings behind. When we do deep-dive into the lower level functionality, it will be extremely easy to comprehend, because we will only have to understand 3-5 lines in this case. This example illustrates, that we cannot score the cleaniless of our code from the line count of our functions. The first function iteration was much shorter. However, it was artificially short and very difficult to read. In most cases cleaning code will, to begin with, expand the already existing code base, in terms of lines of code. However, the benefit of readability is far preferred. If you are ever in doubt about this, think of how you feel about the following function, which does the same:

```go
func GetItemIfActive(extension string) (Item, error) {
    if refIface,ok := db.ReferenceCache.Get(extension); ok {if ref,ok := refIface.(string); ok { if itemIface,ok := db.ItemCache.Get(ref); ok { if item,ok := itemIface.(Item); ok { if item.Active { return Item,nil }}}}} return EMPTY_ITEM, errors.New("reference not found in cache")
}
```

While we are on the topic. There are also a bunch of other side-effects that writing this style of code. Rather obviously, it makes our code much easier to test. It's much easier to get 100% code coverage on a function that is 4 lines (but written by a sane person), than a function which is 400 lines. That's common sense. However, this doesn't necessarily mean that people are willing to refactor their code and thereby make their lives easier. However, I advise, that if you are ever having difficulties with testing your code. Please consider refactoring your functions and trying again. It's most likely not: "because some things are just difficult to test", but rather that really large functions are just always difficult to test.

Another nice side-effect of writing smaller functions. Is that it can typically eliminate using longer lasting mutable variables. Writing code with global variables, at least at a higher level, is a pratice of the past, it doesn't belong in clean code. Now, why is that? Well, the problem with using global variables is that we make it very difficult for programmers to understand the current state of a variable. If this variable is global and mutable, then, by definition, it's value can be changed by any other code in the codebase. At no point can you guarantee that this variable is going to be a specific value... This is a headache for everyone. But let's, look at a short example of how even larger scoped (not global) variables can cause problems. This is taken from an article named: [`Golang scope issue - A feature bug: Shadow Variables`](https://idiallo.com/blog/golang-scopes):

```go
func doComplex() (string, error) {
    return "Success", nil
}

func main() {
    var val string
    num := 32

    switch num {
    case 16:
    // do nothing
    case 32:
        val, err := doComplex()
        if err != nil {
            panic(err)
        }
        if val == "" {
            // do something else
        }
    case 64:
        // do nothing
    }
    
    fmt.Println(val)
}
```

The problem with this code, from a quick skim, it seems like that the `var val string` value, should be printed out as: `Success` by the end of the `main` function. Unfortuantely, this is not the case. The reason for this is, the line:

> val, err := doComplex()

This declares a new variable `val` in the the switch case `32` scope and has nothing to do with the variable declared in the first line of `main`. Of course, it can be argued that the golang syntax is a little tricky, which I don't necessarily disagree with, but there is a much worse issue at hand. The declaration of `var val string` as a mutable largely scoped variable, is completely unecessary. If we do a **very** simple refactor, we will no longer have this issue:

```go
func getStringResult(num int) (string, error) {
    switch num {
    case 16:
    // do nothing
    case 32:
       return doComplex()
    case 64:
        // do nothing
    }
    return "" 
}

func main() {
    val, err := getStringResult(32)
    if err != nil {
        panic(err)
    }
    if val == "" {
        // do something else
    }
    fmt.Println(val)
}
```

Look at that, `val` never had to be mutated, nor did it have to have such a large scope. 

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