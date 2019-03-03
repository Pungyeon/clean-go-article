---
title: Clean Golang Code
output: pdf
---

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
fn GetItem:
    - parse json input for order id
    - get user from context
    - check user has appropriate role
    - get order from database
```

When using small functions (typically 5-8 lines in Golang), we can create a function that reads almost as easily as our description:

```go
var (
    NullItem = Item{}
    ErrInsufficientPrivliges = errors.New("user does not have sufficient priviliges")
)

func GetItem(ctx context.Context, json []bytes) (Item, error) {
    order, err := NewItemFromJSON(json)
    if err != nil {
        return NullItem, err
    }
    if GetUserFromContext(ctx).IsAdmin() {
        return db.GetItem(order.ID)
    }
    return NullItem, ErrInsufficientPrivliges
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

#### Variable Scope
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

Look at that, `val` never had to be mutated, nor did it have to have such a large scope. Again, keep in mind that these functions are very simple. Once this kind of code style becomes a part of larger more complex systems, it can be impossible to figure out, why errors are happening. We don't want this to happen. Not only because we generally dislike errors happening in software, but it is also disrespectful to our colleagues, and ourselves, that we are potentially wasting each others live's, having to debug this type of code. So, let's make a promise that, even though we disagree with golang declaration method, which isn't always the clearest, let's make sure that we never have to worry about it.

On a side not, if the `// do something else` part is another attempt to mutate the `val` variable. We should extract whatever logic in there as a function, as well as the previous part of it. This way, instead of prolonging the mutational scope of our variables, we can just return a new value:

```go
func getVal(num int) (string, error) {
    val, err := getStringResult(32)
    if err != nil {
        return "", err
    }
    if val == "" {
        return NewValue() // pretend function
    }
}

func main() {
    val, err := getVal(32)
    if err != nil {
        panic(err)
    }
    fmt.Println(val)
}
```

### Clean Golang
This section will describe some less generic aspects of writing clean golang code, but rather be discussing aspects that are very go specific. Like the previous section, there will still be a mix of generic and specific concepts being discussed, however, this section marks the start of the document, where the document changes from a generic description of clean code with golang examples, to golang specific descriptions, based on clean code principles.
 

#### Returning Defined Errors
We will be started out nice an easy, by describing a cleaner way to return errors. Like discussed earlier, our main goals with writing clean code, is to ensure readibility, testability and maintanability of the code base. This error returnign method will improve all three aspects, with very little effort.

Let's consider the normal way to return a custom error. This is a hypothetical example taken from a thread-safe map implementation, we have named `Store`:

```go
package smelly

func (store *Store) GetItem(id string) (Item, error) {
    store.mtx.Lock()
    defer store.mtx.Unlock()

    item, ok := store.items[id]
    if !ok {
        return Item{}, errors.New("item could not be found in the store") 
    }
    return item, nil
}
```

There is as such, nothing inherently smelly about this function, in its isolation. We look into the `items` map of our `Store` struct, to see if we already have an item with this `id`. If we do, we return the item, if we don't, we return an error. Pretty standard. So, what is the issue with returning custom errors like this? Well, let's look at what happens, when we use this function, from another package:

```go
func GetItemHandler(w http.ReponseWriter, r http.Request) {
    item, err := smelly.GetItem("123")
    if err != nil {
        if err.Error() == "item could not be found in the store" {
            http.Error(w, err, http.StatusNotFound)
	        return
        }
        http.Error(w, errr, http.StatusInternalServerError)
        return
    } 
    json.NewEncoder(w).Encode(item)
}
```

This is actually not too bad. However, there is one glaring problem with this. Errors in golang, are simply just an `interface` which implements a function (`Error()`) which returns a string. Therefore, we are now hardcoding the expected error code into our code base. This isn't too great. Mainly, because if we wish to change this error message to something else at another point, our code is too closely coupled, meaning that we would have to change our code in, possibly, many different places. Even worse, this could be a client using our package in their software, their software would inexplicably break all of a sudden after a package update. This is quite obviously somethign that we want to avoid. Fortunately, this fix is very simple. 

```go
package clean

var (
    NullItem = Item{}

    ErrItemNotFound = errors.New("item could not be found in the store") 
)

func (store *Store) GetItem(id string) (Item, error) {
    store.mtx.Lock()
    defer store.mtx.Unlock()

    item, ok := store.items[id]
    if !ok {
        return NullItem, ErrItemNotFound
    }
    return item, nil
}
```

With this simple change of making the error into a variable `ErrItemNotFound`, we ensure that anyone using this package can check against the variable, rather than the actual string that it returns:

```go
func GetItemHandler(w http.ReponseWriter, r http.Request) {
    item, err := clean.GetItem("123")
    if err != nil {
        if err == clean.ErrItemNotFound {
            http.Error(w, err, http.StatusNotFound)
	        return
        }
        http.Error(w, errr, http.StatusInternalServerError)
        return
    } 
    json.NewEncoder(w).Encode(item)
}
```

This feels much nicer and is also much safer. Some would even say that it's easier to read as well. In the case of a more verbose error message, it certainly would be preferable for a developer to simply read `ErrItemNotFound` rather than a novel on why a certain error has been returned.

This approach is not limited to errors and can be used for other returned values. As an example, we are also returning a `NullItem` instead of `Item{}` as we did before. Again, this is more foolproof for future development and thereby also makes the code more maintainable. There can be many different scenarios in which it might be preferable to return this defined object, rather than initialising it on return. As an example, we could find out that we want to introduce more safety, in case developers using our package forget to check for errors and invoke a function, which now attempts to access a nil pointer reference. If we were to return the value on return, this could end up in an annoying code refactor. Whereas with our defined `Null` value, we just have to change a single line:

```go
var NullItem = Item{ pointerObj: NewPointerObj() }
```

#### Returning Dynamic Errors
There are certainly some scenarios, where returning an error variable might not actually be viable. In cases where customised errors' information is dynamic, to describe error events more specifically, we cannot define and return our static errors anymore. As an example:

```go
func (store *Store) GetItem(id string) (Item, error) {
    store.mtx.Lock()
    defer store.mtx.Unlock()

    item, ok := store.items[id]
    if !ok {
        return NullItem, fmt.Errorf("Could not find item with ID: %s", id)
    }
    return item, nil
}
```

So, what to do? There is no well defined / standard method for handling and returning these kind of dynamic errors. My personal preference, is to return a new interface, with a bit of added functionality:

```go
type ErrorDetails interface {
    Error() string
    Type() string
}

type errDetails struct {
    errtype error
    details string
}

func NewErrorDetails(err error, details ...interface{}) ErrorDetails {
    return &errDetails{
        errtype: err,
        details: details,
    }
}

func (err *errDetails) Error() string {
    return fmt.Sprintf("%v: %v", err.details)
}

func (err *errDetails) Type() error {
    return err.errtype
}
```

This new data structure still works as our standard error struct. We can still compare it to `nil` since it's an interface implementation and we can still call `.Error()` on it, so it won't break any already existing implementations. However, the advantage is that we can now check our error type as we could previously, despite our error now containing the *dynamic* details:

```go
func (store *Store) GetItem(id string) (Item, error) {
    store.mtx.Lock()
    defer store.mtx.Unlock()

    item, ok := store.items[id]
    if !ok {
        return NullItem, fmt.Errorf("Could not find item with ID: %s", id)
    }
    return item, nil
}
```

And our http handler function can then be refactored to check for a specific error again:


```go
func GetItemHandler(w http.ReponseWriter, r http.Request) {
    item, err := clean.GetItem("123")
    if err != nil {
        if err.Type() == clean.ErrItemNotFound {
            http.Error(w, err, http.StatusNotFound)
	        return
        }
        http.Error(w, err, http.StatusInternalServerError)
        return
    } 
    json.NewEncoder(w).Encode(item)
}
```


#### Interfaces in Go
In general, the go method for handling `interface`'s is quite different from other languages. Interfaces aren't explicitly implemented, like they would be in Java or C#, but are implicitly implemented, if they fulfill the contract of the interface. As an example, this means that any `struct` which has an `Error()` method, implements / fullfills the `Error` interface and can be returned as an error. This has it's advantages, as it makes golang feel more fast-paced and dynamic, as implementing an interface is extremely easy. The general proverb for writing golang functions is: 

*"Be consdervative in what you do, be liberal in what you accept from others"*

In other words, you should write functions that accept an interface and return a concrete type. This is generally a good practice, and becomes super beneficial when doing tests with mocking. As an example, we can create a function which takes a writer interface as input and invokes the `Write` method of that inteface.

```go
type Pipe struct {
    writer io.Writer
    buffer bytes.Buffer
}

func NewPipe(w io.Writer) *Pipe {
    return &Pipe{
        writer: w,
    }
}

func (pipe *Pipe) Save() error {
    if _, err := pipe.writer.Write(pipe.FlushBuffer()); err != nil {
        return err
    }
    return nil
}
```

Let's assume that we are writing to a file when our application is running, but we don't want to write to a new file for all tests which invokes this function. Therefore, we can implement a new mock type, which will basically do nothing. Essentially, this is just basic dependency injection and mocking, but the point is that it is extremely easy to use in go:

```go
type NullWriter struct {}

func (w *NullWriter) Write(b []byte) (int, error) {
    return len(b), nil
}

func TestFn(t *testing.T) {
    ...
    pipe := NewPipe(NullWriter{})
    ...
}
```

When constructing our `Pipe` struct with the `NullWriter` (rather than a different writer), when invoking our `Save` function, nothing will happen. The only thing we had to do, was add 4 lines of code. This is why in idiomatic go, it is encouraged to make interface types as small as possible, to make implement a pattern like this as easy as possible. This is great and as I mentioned, is a great advantage of go. However, this implementation of interfaces, also comes with a *huge* downside. 


### The empty `interface{}`
Unlike other languages, go does not have an implementation for generics. There have been many proposals on how to implement this and will eventually be implemented. However, without generics, developers are trying to find creative ways around this issue, very often using the empty `interface{}`. The next section, will describe why these, often too creative, implementations should be considered bad practice and unclean code. There will also be good examples of usage of the empty `interface{}` and how to avoid some pitfalls of writing code with the empty `interface{}`. 

But first and foremost. What drives developers to use the empty `interface{}`? Well, as I said in the previously, the way that golang determines whether a concrete type implements an interface, is by checking whether it implements the methods of a specific interface. So what happens, if our interface implement no methods at all?

```go
type EmptyInterface interface {}
```

The above being equivalent to the built-in type `interface{}`. The result of this interface type is that **any** type is accepted. Meaning, that we can write functions in which any type is accepted. This is super useful for certain kind of functions, such as when creating a printer function. This is how it's possible to give any type to the `Println` function from the `fmt` package:

```go
func Println(v ...interface{}) {
    ...
}
```

In this case, we aren't only accepting a single `interface{}` but rather, a slice of types. These types can be of any type and can even be of different types, as long as they implement the empty `interface{}`, which funnily enough, they 100% will. This is a super common pattern when handling string conversation (both from and to string). The reason being, this is the only way in golang to implement generic methods.  