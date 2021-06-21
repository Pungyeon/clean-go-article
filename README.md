# Clean Go Code

## Preface: Why Write Clean Code?

This document is a reference for the Go community that aims to help developers write cleaner code. Whether you're working on a personal project or as part of a larger team, writing clean code is an important skill to have. Establishing good paradigms and consistent, accessible standards for writing clean code can help prevent developers from wasting many meaningless hours on trying to understand their own (or others') work.

> <em>We don’t read code, we <b>decode</b> it &ndash; Peter Seibel</em>

As developers, we're sometimes tempted to write code in a way that's convenient for the time being without regard for best practices; this makes code reviews and testing more difficult. In a sense, we're <em>encoding</em>&mdash;and, in doing so, making it more difficult for others to decode our work. But we want our code to be usable, readable, and maintainable. And that requires coding the <em>right</em> way, not the easy way.

This document begins with a simple and short introduction to the fundamentals of writing clean code. Later, we'll discuss concrete refactoring examples specific to Go.

##### A short word on `gofmt`
I'd like to take a few sentences to clarify my stance on `gofmt` because there are plenty of things I disagree with when it comes to this tool. I prefer snake case over camel case, and I quite like my constant variables to be uppercase. And, naturally, I also have many opinions on bracket placement. *That being said*, `gofmt` does allow us to have a common standard for writing Go code, and that's a great thing. As a developer myself, I can certainly appreciate that Go programmers may feel somewhat restricted by `gofmt`, especially if they disagree with some of its rules. But in my opinion, homogeneous code is more important than having complete expressive freedom.

## Table of Contents
* [Introduction to Clean Code](#Introduction-to-Clean-Code)
    * [Test-Driven Development](#Test-Driven-Development)
    * [Naming Conventions](#Naming-Conventions)
    * * [Comments](#Comments)
    	* [Function Naming](#Function-Naming)
    	* [Variable Naming](#Variable-Naming)
    * [Cleaning Functions](#Cleaning-Functions)
      * [Function Length](#Function-Length)
      * [Function Signatures](#Function-Signatures)
    * [Variable Scope](#Variable-Scope)
    * [Variable Declaration](#Variable-Declaration)
    
* [Clean Go](#Clean-Go)
    * [Return Values](#Return-Values) 
      * [Returning Defined Errors](#Returning-Defined-Errors)
      * [Returning Dynamic Errors](#Returning-Dynamic-Errors)
    * [Pointers in Go](#Pointers-in-Go)
    * [Closures Are Function Pointers](#Closures-are-Function-Pointers)
    * [Interfaces in Go](#Interfaces-in-Go)
    * [The Empty `interface{}`](#The-Empty-Interface)
* [Summary](#Summary)

## Introduction to Clean Code

Clean code is the pragmatic concept of promoting readable and maintainable software. Clean code establishes trust in the codebase and helps minimize the chances of careless bugs being introduced. It also helps developers maintain their agility, which typically plummets as the codebase expands due to the increased risk of introducing bugs.

### Test-Driven Development

Test-driven development is the practice of testing your code frequently throughout short development cycles or sprints. It ultimately contributes to code cleanliness by inviting developers to question the functionality and purpose of their code. To make testing easier, developers are encouraged to write short functions that only do one thing. For example, it's arguably much easier to test (and understand) a function that's only 4 lines long than one that's 40.

Test-driven development consists of the following cycle:

1. Write (or execute) a test
2. If the test fails, make it pass
3. Refactor your code accordingly
4. Repeat

Testing and refactoring are intertwined in this process. As you refactor your code to make it more understandable or maintainable, you need to test your changes thoroughly to ensure that you haven't altered the behavior of your functions. This can be incredibly useful as the codebase grows.

###  Naming Conventions

#### Comments
I'd like to first address the topic of commenting code, which is an essential practice but tends to be misapplied. Unnecessary comments can indicate problems with the underlying code, such as the use of poor naming conventions. However, whether or not a particular comment is "necessary" is somewhat subjective and depends on how legibly the code was written. For example, the logic of well-written code may still be so complex that it requires a comment to clarify what is going on. In that case, one might argue that the comment is <em>helpful</em> and therefore necessary.

In Go, according to `gofmt`, <em>all</em> public variables and functions should be annotated. I think this is absolutely fine, as it gives us consistent rules for documenting our code. However, I always want to distinguish between comments that enable auto-generated documentation and <em>all other</em> comments. Annotation comments, for documentation, should be written like documentation&mdash;they should be at a high level of abstraction and concern the logical implementation of the code as little as possible.

I say this because there are other ways to explain code and ensure that it's being written comprehensibly and expressively. If the code is neither of those, some people find it acceptable to introduce a comment explaining the convoluted logic. Unfortunately, that doesn't really help. For one, most people simply won't read comments, as they tend to be very intrusive to the experience of reviewing code. Additionally, as you can imagine, a developer won't be too happy if they're forced to review unclear code that's been slathered with comments. The less that people have to read to understand what your code is doing, the better off they'll be.

Let's take a step back and look at some concrete examples. Here's how you <em>shouldn't</em> comment your code:

```go
// iterate over the range 0 to 9 
// and invoke the doSomething function
// for each iteration
for i := 0; i < 10; i++ {
  doSomething(i)
}
```

This is what I like to call a <strong>tutorial comment</strong>; it's fairly common in tutorials, which often explain the low-level functionality of a language (or programming in general). While these comments may be helpful for beginners, they're absolutely useless in production code. Hopefully, we aren't collaborating with programmers who don't understand something as simple as a looping construct by the time they've begun working on a development team. As programmers, we shouldn't have to read the comment to understand what's going on&mdash;we know that we're iterating over the range 0 to 9 because we can simply read the code. Hence the proverb:

> <em>Document why, not how. &ndash; Venkat Subramaniam</em>

Following this logic, we can now change our comment to explain <em>why</em> we are iterating from the range 0 to 9:

```go
// instatiate 10 threads to handle upcoming work load
for i := 0; i < 10; i++ {
  doSomething(i)
}
```

Now we understand <em>why</em> we have a loop and can tell <em>what</em> we're doing by simply reading the code... Sort of.

This still isn't what I'd consider clean code. The comment is worrying because it probably should not be necessary to express such an explanation in prose, assuming the code is well written (which it isn't). Technically, we're still saying what we're doing, not why we're doing it. We can easily express this "what" directly in our code by using more meaningful names:

```go
for workerID := 0; workerID < 10; workerID++ {
  instantiateThread(workerID)
}
```

With just a few changes to our variable and function names, we've managed to explain what we're doing directly in our code. This is much clearer for the reader because they won't have to read the comment and then map the prose to the code. Instead, they can simply read the code to understand what it's doing.

Of course, this was a relatively trivial example. Writing clear and expressive code is unfortunately not always so easy; it can become increasingly difficult as the codebase itself grows in complexity. The more you practice writing comments in this mindset and avoid explaining what you're doing, the cleaner your code will become.

#### Function Naming

Let's now move on to function naming conventions. The general rule here is really simple: the more specific the function, the more general its name. In other words, we want to start with a very broad and short function name, such as `Run` or `Parse`, that describes the general functionality. Let's imagine that we are creating a configuration parser. Following this naming convention, our top level of abstraction might look something like the following:

```go
func main() {
    configpath := flag.String("config-path", "", "configuration file path")
    flag.Parse()

    config, err := configuration.Parse(*configpath)
    
    ...
}
```

We'll focus on the naming of the `Parse` function. Despite this function's very short and general name, it's actually quite clear what it attempts to achieve.

When we go one layer deeper, our function naming will become slightly more specific:

```go
func Parse(filepath string) (Config, error) {
    switch fileExtension(filepath) {
    case "json":
        return parseJSON(filepath)
    case "yaml":
        return parseYAML(filepath)
    case "toml":
        return parseTOML(filepath)
    default:
        return Config{}, ErrUnknownFileExtension
    }
}
```

Here, we've clearly distinguished the nested function calls from their parent without being overly specific. This allows each nested function call to make sense on its own as well as within the context of the parent. On the other hand, if we had named the `parseJSON` function `json` instead, it couldn't possibly stand on its own. The functionality would become lost in the name, and we would no longer be able to tell whether this function is parsing, creating, or marshalling JSON. 

Notice that `fileExtension` is actually a little more specific. However, this is because its functionality is in fact quite specific in nature:

```go
func fileExtension(filepath string) string {
    segments := strings.Split(filepath, ".")
    return segments[len(segments)-1]
}
```

This kind of logical progression in our function names&mdash;from a high level of abstraction to a lower, more specific one&mdash;makes the code easier to follow and read. Consider the alternative: If our highest level of abstraction is too specific, then we'll end up with a name that attempts to cover all bases, like `DetermineFileExtensionAndParseConfigurationFile`. This is horrendously difficult to read; we are trying to be too specific too soon and end up confusing the reader, despite trying to be clear! 

#### Variable Naming
Rather interestingly, the opposite is true for variables. Unlike functions, our variables should be named from more to less specific the deeper we go into nested scopes.

> <em>You shouldn’t name your variables after their types for the same reason you wouldn’t name your pets 'dog' or 'cat'. &ndash; Dave Cheney</em>

Why should our variable names become less specific as we travel deeper into a function's scope? Simply put, as a variable's scope becomes smaller, it becomes increasingly clear for the reader what that variable represents, thereby eliminating the need for specific naming. In the example of the previous function `fileExtension`, we could even shorten the name of the variable `segments` to `s` if we wanted to. The context of the variable is so clear that it's unnecessary to explain it any further with longer variable names. Another good example of this is in nested for loops:

```go
func PrintBrandsInList(brands []BeerBrand) {
    for _, b := range brands { 
        fmt.Println(b)
    }
}
```

In the above example, the scope of the variable `b` is so small that we don't need to spend any additional brain power on remembering what exactly it represents. However, because the scope of `brands` is slightly larger, it helps for it to be more specific. When expanding the variable scope in the function below, this distinction becomes even more apparent:

```go
func BeerBrandListToBeerList(beerBrands []BeerBrand) []Beer {
    var beerList []Beer
    for _, brand := range beerBrands {
        for _, beer := range brand {
            beerList = append(beerList, beer)
        }
    }
    return beerList
}
```

Great! This function is easy to read. Now, let's apply the opposite (i.e., wrong) logic when naming our variables:

```go
func BeerBrandListToBeerList(b []BeerBrand) []Beer {
    var bl []Beer
    for _, beerBrand := range b {
        for _, beerBrandBeerName := range beerBrand {
            bl = append(bl, beerBrandBeerName)
        }
    }
    return bl
}
```

Even though it's possible to figure out what this function is doing, the excessive brevity of the variable names makes it difficult to follow the logic as we travel deeper. This could very well spiral into full-blown confusion because we're mixing short and long variable names inconsistently.

### Cleaning Functions

Now that we know some best practices for naming our variables and functions, as well as clarifying our code with comments, let's dive into some specifics of how we can refactor functions to make them cleaner.

#### Function Length

> <em>How small should a function be? Smaller than that! &ndash; Robert C. Martin</em>

When writing clean code, our primary goal is to make our code easily digestible. The most effective way to do this is to make our functions as short as possible. It's important to understand that we don't necessarily do this to avoid code duplication. The more important reason is to improve <em>code comprehension</em>.

It can help to look at a function's description at a very high level to understand this better:

```
fn GetItem:
    - parse json input for order id
    - get user from context
    - check user has appropriate role
    - get order from database
```

By writing short functions (which are typically 5&ndash;8 lines in Go), we can create code that reads almost as naturally as our description above:

```go
var (
    NullItem = Item{}
    ErrInsufficientPrivileges = errors.New("user does not have sufficient privileges")
)

func GetItem(ctx context.Context, json []bytes) (Item, error) {
    order, err := NewItemFromJSON(json)
    if err != nil {
        return NullItem, err
    }
    if !GetUserFromContext(ctx).IsAdmin() {
	      return NullItem, ErrInsufficientPrivileges
    }
    return db.GetItem(order.ItemID)
}
```

Using smaller functions also eliminates another horrible habit of writing code: indentation hell. <strong>Indentation hell</strong> typically occurs when a chain of `if` statements are carelessly nested in a function. This makes it <em>very</em> difficult for human beings to parse the code and should be eliminated whenever spotted. Indentation hell is particularly common when working with `interface{}` and using type casting:

```go
func GetItem(extension string) (Item, error) {
    if refIface, ok := db.ReferenceCache.Get(extension); ok {
        if ref, ok := refIface.(string); ok {
            if itemIface, ok := db.ItemCache.Get(ref); ok {
                if item, ok := itemIface.(Item); ok {
                    if item.Active {
                        return Item, nil
                    } else {
                      return EmptyItem, errors.New("no active item found in cache")
                    }
                } else {
                  return EmptyItem, errors.New("could not cast cache interface to Item")
                }
            } else {
              return EmptyItem, errors.New("extension was not found in cache reference")
            }
        } else {
          return EmptyItem, errors.New("could not cast cache reference interface to Item")
        }
    }
    return EmptyItem, errors.New("reference not found in cache")
}
```

First, indentation hell makes it difficult for other developers to understand the flow of your code. Second, if the logic in our `if` statements expands, it'll become exponentially more difficult to figure out which statement returns what (and to ensure that all paths return some value). Yet another problem is that this deep nesting of conditional statements forces the reader to frequently scroll and keep track of many logical states in their head. It also makes it more difficult to test the code and catch bugs because there are so many different nested possibilities that you have to account for.

Indentation hell can result in reader fatigue if a developer has to constantly parse unwieldy code like the sample above. Naturally, this is something we want to avoid at all costs.

So, how do we clean this function? Fortunately, it's actually quite simple. On our first iteration, we will try to ensure that we are returning an error as soon as possible. Instead of nesting the `if` and `else` statements, we want to "push our code to the left," so to speak. Take a look:

```go
func GetItem(extension string) (Item, error) {
    refIface, ok := db.ReferenceCache.Get(extension)
    if !ok {
        return EmptyItem, errors.New("reference not found in cache")
    }

    ref, ok := refIface.(string)
    if !ok {
        // return cast error on reference 
    }

    itemIface, ok := db.ItemCache.Get(ref)
    if !ok {
        // return no item found in cache by reference
    }

    item, ok := itemIface.(Item)
    if !ok {
        // return cast error on item interface
    }

    if !item.Active {
        // return no item active
    }

    return Item, nil
}
```

Once we're done with our first attempt at refactoring the function, we can proceed to split up the function into smaller functions. Here's a good rule of thumb:  If the `value, err :=` pattern is repeated more than once in a function, this is an indication that we can split the logic of our code into smaller pieces:

```go
func GetItem(extension string) (Item, error) {
    ref, ok := getReference(extension)
    if !ok {
        return EmptyItem, ErrReferenceNotFound
    }
    return getItemByReference(ref)
}

func getReference(extension string) (string, bool) {
    refIface, ok := db.ReferenceCache.Get(extension)
    if !ok {
        return EmptyItem, false
    }
    return refIface.(string)
}

func getItemByReference(reference string) (Item, error) {
    item, ok := getItemFromCache(reference)
    if !item.Active || !ok {
        return EmptyItem, ErrItemNotFound
    }
    return Item, nil
}

func getItemFromCache(reference string) (Item, bool) {
    if itemIface, ok := db.ItemCache.Get(ref); ok {
        return EmptyItem, false
    }
    return itemIface.(Item), true
}
```

As mentioned previously, indentation hell can make it difficult to test our code. When we split up our `GetItem` function into several helpers, we make it easier to track down bugs when testing our code. Unlike the original version, which consisted of several `if` statements in the same scope, the refactored version of `GetItem` has just two branching paths that we must consider. The helper functions are also short and digestible, making them easier to read.

> Note: For production code, one should elaborate on the code even further by returning errors instead of `bool` values. This makes it much easier to understand where the error is originating from. However, as these are just example functions, returning `bool` values will suffice for now. Examples of returning errors more explicitly will be explained in more detail later.

Notice that cleaning the `GetItem` function resulted in more lines of code overall. However, the code itself is now much easier to read. It's layered in an onion-style fashion, where we can ignore "layers" that we aren't interested in and simply peel back the ones that we do want to examine. This makes it easier to understand low-level functionality because we only have to read maybe 3&ndash;5 lines at a time.

This example illustrates that we cannot measure the cleanliness of our code by the number of lines it uses. The first version of the code was certainly much shorter. However, it was <em>artificially</em> short and very difficult to read. In most cases, cleaning code will initially expand the existing codebase in terms of the number of lines. But this is highly preferable to the alternative of having messy, convoluted logic. If you're ever in doubt about this, just consider how you feel about the following function, which does exactly the same thing as our code but only uses two lines:

```go
func GetItemIfActive(extension string) (Item, error) {
    if refIface,ok := db.ReferenceCache.Get(extension); ok {if ref,ok := refIface.(string); ok { if itemIface,ok := db.ItemCache.Get(ref); ok { if item,ok := itemIface.(Item); ok { if item.Active { return Item,nil }}}}} return EmptyItem, errors.New("reference not found in cache")
}
```

#### Function Signatures

Creating a good function naming structure makes it easier to read and understand the intent of the code. As we saw above, making our functions shorter helps us understand the function's logic. The last part of cleaning our functions involves understanding the context of the function input. With this comes another easy-to-follow rule: <strong>Function signatures should only contain one or two input parameters</strong>. In certain exceptional cases, three can be acceptable, but this is where we should start considering a refactor. Much like the rule that our functions should only be 5&ndash;8 lines long, this can seem quite extreme at first. However, I feel that this rule is much easier to justify.

Take the following function from [RabbitMQ's introduction tutorial to its Go library](https://www.rabbitmq.com/tutorials/tutorial-one-go.html):

```go
q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when unused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
```

The function `QueueDeclare` takes six input parameters, which is quite a lot. With some effort, it's possible to understand what this code does thanks to the comments. However, the comments are actually part of the problem&mdash;as mentioned earlier, they should be substituted with descriptive code whenever possible. After all, there's nothing preventing us from invoking the `QueueDeclare` function <em>without</em> comments:

```go
q, err := ch.QueueDeclare("hello", false, false, false, false, nil)
```

Now, without looking at the commented version, try to remember what the fourth and fifth `false` arguments represent. It's impossible, right? You will inevitably forget at some point. This can lead to costly mistakes and bugs that are difficult to correct. The mistakes might even occur through incorrect comments&mdash;imagine labeling the wrong input parameter. Correcting this mistake will be unbearably difficult to correct, especially when familiarity with the code has deteriorated over time or was low to begin with. Therefore, it is recommended to replace these input parameters with an 'Options' `struct` instead:

```go
type QueueOptions struct {
    Name string
    Durable bool
    DeleteOnExit bool
    Exclusive bool
    NoWait bool
    Arguments []interface{} 
}

q, err := ch.QueueDeclare(QueueOptions{
    Name: "hello",
    Durable: false,
    DeleteOnExit: false,
    Exclusive: false,
    NoWait: false,
    Arguments: nil,
})
```

This solves two problems: misusing comments, and accidentally labeling the variables incorrectly. Of course, we can still confuse properties with the wrong value, but in these cases, it will be much easier to determine where our mistake lies within the code. The ordering of the properties also doesn't matter anymore, so incorrectly ordering the input values is no longer a concern. The last added bonus of this technique is that we can use our `QueueOptions` struct to infer the default values of our function's input parameters. When structures in Go are declared, all properties are initialised to their default value. This means that our `QueueDeclare` option can actually be invoked in the following way:

```go
q, err := ch.QueueDeclare(QueueOptions{
    Name: "hello",
})
```

The rest of the values are initialised to their default value of `false` (except for `Arguments`, which as an interface has a default value of `nil`). Not only are we much safer with this approach, but we are also much clearer with our intentions. In this case, we could actually write less code. This is an all-around win for everyone on the project.

One final note on this: It's not always possible to change a function's signature. In this case, for example, we don't actually have control over our `QueueDeclare` function signature because it's from the RabbitMQ library. It's not our code, so we can't change it. However, we can wrap these functions to suit our purposes:

```go
type RMQChannel struct {
    channel *amqp.Channel
}

func (rmqch *RMQChannel) QueueDeclare(opts QueueOptions) (Queue, error) {
    return rmqch.channel.QueueDeclare(
        opts.Name,
        opts.Durable,
        opts.DeleteOnExit,
        opts.Exclusive,
        opts.NoWait,
        opts.Arguments, 
    )
} 
```

Basically, we create a new structure named `RMQChannel` that contains the `amqp.Channel` type, which has the `QueueDeclare` method. We then create our own version of this method, which essentially just calls the old version of the RabbitMQ library function. Our new method has all the advantages described before, and we achieved this without actually having to change any of the code in the RabbitMQ library.

We'll use this idea of wrapping functions to introduce more clean and safe code later when discussing `interface{}`.

### Variable Scope
Now, let's take a step back and revisit the idea of writing smaller functions. This has another nice side effect that we didn't cover in the previous chapter: Writing smaller functions can typically eliminate reliance on mutable variables that leak into the global scope.

Global variables are problematic and don't belong in clean code; they make it very difficult for programmers to understand the current state of a variable. If a variable is global and mutable, then by definition, its value can be changed by any part of the codebase. At no point can you guarantee that this variable is going to be a specific value... And that's a headache for everyone. This is yet another example of a trivial problem that's exacerbated when the codebase expands.

Let's look at a short example of how non-global variables with a large scope can cause problems. These variables also introduce the issue of <strong>variable shadowing</strong>, as demonstrated in the code taken from an article titled [Golang scope issue](https://idiallo.com/blog/golang-scopes):

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

What's the problem with this code? From a quick skim, it seems the `var val string` value should be printed out as `Success` by the end of the `main` function. Unfortunately, this is not the case. The reason for this lies in the following line:

```go
val, err := doComplex()
```

This declares a new variable `val` in the switch's `case 32` scope and has nothing to do with the variable declared in the first line of `main`. Of course, it can be argued that Go syntax is a little tricky, which I don't necessarily disagree with, but there is a much worse issue at hand. The declaration of `var val string` as a mutable, largely scoped variable is completely unnecessary. If we do a <strong>very</strong> simple refactor, we will no longer have this issue:

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
    return "", nil
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

After our refactor, `val` is no longer modified, and the scope has been reduced. Again, keep in mind that these functions are very simple. Once this kind of code style becomes a part of larger, more complex systems, it can be impossible to figure out why errors are occurring. We don't want this to happen&mdash;not only because we generally dislike software errors but also because it's disrespectful to our colleagues, and ourselves; we are potentially wasting each other's time having to debug this type of code. Developers need to take responsibility for their own code rather than blaming these issues on the variable declaration syntax of a particular language like Go.

On a side note, if the `// do something else` part is another attempt to mutate the `val` variable, we should extract that logic out as its own self-contained function, as well as the previous part of it. This way, instead of expanding the mutable scope of our variables, we can just return a new value:

```go
func getVal(num int) (string, error) {
    val, err := getStringResult(num)
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

### Variable Declaration 

Other than avoiding issues with variable scope and mutability, we can also improve readability by declaring variables as close to their usage as possible. In C programming, it's common to see the following approach to declaring variables:

```go
func main() {
  var err error
  var items []Item
  var sender, receiver chan Item
  
  items = store.GetItems()
  sender = make(chan Item)
  receiver = make(chan Item)
  
  for _, item := range items {
    ...
  }
}
```

This suffers from the same symptom as described in our discussion of variable scope. Even though these variables might not actually be reassigned at any point, this kind of coding style keeps the readers on their toes, in all the wrong ways. Much like computer memory, our brain's short-term memory has a limited capacity. Having to keep track of which variables are mutable and whether or not a particular fragment of code will mutate them makes it more difficult to understand what the code is doing. Figuring out the eventually returned value can be a nightmare. Therefore, to makes this easier for our readers (and our future selves), it's recommended that you declare variables as close to their usage as possible:

```go
func main() {
	var sender chan Item
	sender = make(chan Item)

	go func() {
		for {
			select {
			case item := <-sender:
				// do something
			}
		}
	}()
}
```

However, we can do even better by invoking the function directly after its declaration. This makes it much clearer that the function logic is associated with the declared variable:

```go
func main() {
  sender := func() chan Item {
    channel := make(chan Item)
    go func() {
      for {
        select { ... }
      }
    }()
    return channel
  }
}
```

And coming full circle, we can move the anonymous function to make it a named function instead:

```go
func main() {
  sender := NewSenderChannel()
}

func NewSenderChannel() chan Item {
  channel := make(chan Item)
  go func() {
    for {
      select { ... }
    }
  }()
  return channel
}
```

It is still clear that we are declaring a variable, and the logic associated with the returned channel is simple, unlike in the first example. This makes it easier to traverse the code and understand the role of each variable.

Of course, this doesn't actually prevent us from mutating our `sender` variable. There is nothing that we can do about this, as there is no way of declaring a `const struct` or `static` variables in Go. This means that we'll have to restrain ourselves from modifying this variable at a later point in the code.

> NOTE: The keyword `const` does exist but is limited in use to primitive types only.

One way of getting around this can at least limit the mutability of a variable to the package level. The trick involves creating a structure with the variable as a private property. This private property is thenceforth only accessible through other methods provided by this wrapping structure. Expanding on our channel example, this would look something like the following:

```go
type Sender struct {
  sender chan Item
}

func NewSender() *Sender {
  return &Sender{
    sender: NewSenderChannel(),
  }
}

func (s *Sender) Send(item Item) {
  s.sender <- item
}
```

We have now ensured that the `sender` property of our `Sender` struct is never mutated&mdash;at least not from outside of the package. As of writing this document, this is the only way of creating publicly immutable non-primitive variables. It's a little verbose, but it's truly worth the effort to ensure that we don't end up with strange bugs resulting from accidental variable modification. 

```go
func main() {
  sender := NewSender()
  sender.Send(&Item{})
}
```

Looking at the example above, it's clear how this also simplifies the usage of our package. This way of hiding the implementation is beneficial not only for the maintainers of the package but also for the users. Now, when initialising and using the `Sender` structure, there is no concern over its implementation. This opens up for a much looser architecture. Because our users aren't concerned with the implementation, we are free to change it at any point, since we have reduced the point of contact that users have with the package. If we no longer wish to use a channel implementation in our package, we can easily change this without breaking the usage of the `Send` method (as long as we adhere to its current function signature).

>  NOTE: There is a fantastic explanation of how to handle the abstraction in client libraries, taken from the talk [AWS re:Invent 2017: Embracing Change without Breaking the World (DEV319)](https://www.youtube.com/watch?v=kJq81Y7OEx4).

## Clean Go

This section focuses less on the generic aspects of writing clean Go code and more on the specifics, with an emphasis on the underlying clean code principles.

### Return Values

#### Returning Defined Errors

We'll start things off nice and easy by describing a cleaner way to return errors. As we discussed earlier, our main goal with writing clean code is to ensure readability, testability, and maintainability of the codebase. The technique for returning errors that we'll discuss here will achieve all three of those goals with very little effort.

Let's consider the normal way to return a custom error. This is a hypothetical example taken from a thread-safe map implementation that we've named `Store`:

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

There is nothing inherently smelly about this function when we consider it in isolation. We look into the `items` map of our `Store` struct to see if we already have an item with the given `id`. If we do, we return it; otherwise, we return an error. Pretty standard. So, what is the issue with returning custom errors as string values? Well, let's look at what happens when we use this function inside another package:

```go
func GetItemHandler(w http.ReponseWriter, r http.Request) {
    item, err := smelly.GetItem("123")
    if err != nil {
        if err.Error() == "item could not be found in the store" {
            http.Error(w, err.Error(), http.StatusNotFound)
	        return
        }
        http.Error(w, errr.Error(), http.StatusInternalServerError)
        return
    } 
    json.NewEncoder(w).Encode(item)
}
```

This is actually not too bad. However, there is one glaring problem: An error in Go is simply an `interface` that implements a function (`Error()`) returning a string; thus, we are now hardcoding the expected error code into our codebase, which isn't ideal. This hardcoded string is known as a <strong>magic string</strong>. And its main problem is flexibility: If at some point we decide to change the string value used to represent an error, our code will break (softly) unless we update it in possibly many different places. Our code is tightly coupled&mdash;it relies on that specific magic string and the assumption that it will never change as the codebase grows.

An even worse situation would arise if a client were to use our package in their own code. Imagine that we decided to update our package and changed the string that represents an error&mdash;the client's software would now suddenly break. This is quite obviously something that we want to avoid. Fortunately, the fix is very simple:

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

By simply representing the error as a variable (`ErrItemNotFound`), we've ensured that anyone using this package can check against the variable rather than the actual string that it returns:

```go
func GetItemHandler(w http.ReponseWriter, r http.Request) {
    item, err := clean.GetItem("123")
    if err != nil {
        if errors.Is(err, clean.ErrItemNotFound) {
           http.Error(w, err.Error(), http.StatusNotFound)
	        return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    } 
    json.NewEncoder(w).Encode(item)
}
```

This feels much nicer and is also much safer. Some would even say that it's easier to read as well. In the case of a more verbose error message, it certainly would be preferable for a developer to simply read `ErrItemNotFound` rather than a novel on why a certain error has been returned.

This approach is not limited to errors and can be used for other returned values. As an example, we are also returning a `NullItem` instead of `Item{}` as we did before. There are many different scenarios in which it might be preferable to return a defined object, rather than initialising it on return.

Returning default `NullItem` values like we did in the previous examples can also be safer in certain cases. As an example, a user of our package could forget to check for errors and end up initialising a variable that points to an empty struct containing a default value of `nil` as one or more property values. When attempting to access this `nil` value later in the code, the client software would panic. However, when we return our custom default value instead, we can ensure that all values that would otherwise default to `nil` are initialised. Thus, we'd ensure that we do not cause panics in our users' software.

This also benefits us. Consider this: If we wanted to achieve the same safety without returning a default value, we would have to change our code everywhere we return this type of empty value. However, with our default value approach, we now only have to change our code in a single place:

```go
var NullItem = Item{
    itemMap: map[string]Item{},
}
```
> NOTE: In many scenarios, invoking a panic will actually be preferable to indicate that there is an error check missing.

> NOTE: Every interface property in Go has a default value of `nil`. This means that this is useful for any struct that has an interface property. This is also true for structs that contain channels, maps, and slices, which could potentially also have a `nil` value.

#### Returning Dynamic Errors
There are certainly some scenarios where returning an error variable might not actually be viable. In cases where the information in customised errors is dynamic, if we want to describe error events more specifically, we can no longer define and return our static errors. Here's an example:

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

So, what to do? There is no well-defined or standard method for handling and returning these kinds of dynamic errors. My personal preference is to return a new interface, with a bit of added functionality:

```go
type ErrorDetails interface {
    Error() string
    Type() string
}

type errDetails struct {
    errtype error
    details interface{}
}

func NewErrorDetails(err error, details ...interface{}) ErrorDetails {
    return &errDetails{
        errtype: err,
        details: details,
    }
}

func (err *errDetails) Error() string {
    return fmt.Sprintf("%v: %v", err.errtype, err.details)
}

func (err *errDetails) Type() error {
    return err.errtype
}
```

This new data structure still works as our standard error. We can still compare it to `nil` since it's an interface implementation, and we can still call `.Error()` on it, so it won't break any existing implementations. However, the advantage is that we can now check our error type as we could previously, despite our error now containing the <em>dynamic</em> details:

```go
func (store *Store) GetItem(id string) (Item, error) {
    store.mtx.Lock()
    defer store.mtx.Unlock()

    item, ok := store.items[id]
    if !ok {
        return NullItem, NewErrorDetails(
            ErrItemNotFound,
            fmt.Sprintf("could not find item with id: %s", id))
    }
    return item, nil
}
```

And our HTTP handler function can then be refactored to check for a specific error again:

```go
func GetItemHandler(w http.ReponseWriter, r http.Request) {
    item, err := clean.GetItem("123")
    if err != nil {
        if errors.Is(err.Type(), clean.ErrItemNotFound) {
            http.Error(w, err.Error(), http.StatusNotFound)
	        return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    } 
    json.NewEncoder(w).Encode(item)
}
```

### Nil Values 

A controversial aspect of Go is the addition of `nil`. This value corresponds to the value `NULL` in C and is essentially an uninitialised pointer. We've already seen some of the problems that `nil` can cause, but to sum up: Things break when you try to access methods or properties of a `nil` value. Thus, it's recommended to avoid returning a  `nil` value when possible. This way, the users of our code are less likely to accidentally access `nil` values. 

There are other scenarios in which it is common to find `nil` values that can cause some unnecessary pain. An example of this is incorrectly initialising a `struct` (as in the example below), which can lead to it containing `nil` properties. If accessed, those `nil`s will cause a panic.

```go
type App struct {
	Cache *KVCache
}

type KVCache struct {
  mtx sync.RWMutex
	store map[string]string
}

func (cache *KVCache) Add(key, value string) {
  cache.mtx.Lock()
  defer cache.mtx.Unlock()
  
	cache.store[key] = value
}
```

This code is absolutely fine. However, the danger is that our `App` can be initialised incorrectly, without initialising the `Cache` property within. Should the following code be invoked, our application will panic:

```go
	app := App{}
	app.Cache.Add("panic", "now")
```

The `Cache` property has never been initialised and is therefore a `nil` pointer. Thus, invoking the `Add` method like we did here will cause a panic, with the following message:

> panic: runtime error: invalid memory address or nil pointer dereference

Instead, we can turn the `Cache` property of our `App` structure into a private property and create a getter-like method to access it. This gives us more control over what we are returning; specifically, it ensures that we aren't returning a `nil` value:

```go
type App struct {
    cache *KVCache
}

func (app *App) Cache() *KVCache {
	if app.cache == nil {
        app.cache = NewKVCache()
	}
	return app.cache
}
```

The code that previously panicked will now be refactored to the following:

```go
app := App{}
app.Cache().Add("panic", "now")
```

This ensures that users of our package don't have to worry about the implementation and whether they're using our package in an unsafe manner. All they need to worry about is writing their own clean code.

> NOTE: There are other methods of achieving a similarly safe outcome. However, I believe this is the most straightforward approach.

### Pointers in Go
Pointers in Go are a rather extensive topic. They're a very big part of working with the language&mdash;so much so that it is essentially impossible to write Go without some knowledge of pointers and their workings in the language. Therefore, it is important to understand how to use pointers without adding unnecessary complexity (and thereby keeping your codebase clean). Note that we will not review the details of how pointers are implemented in Go. Instead, we will focus on the quirks of Go pointers and how we can handle them.

Pointers add complexity to code. If we aren't cautious, incorrectly using pointers can introduce nasty side effects or bugs that are particularly difficult to debug. By sticking to the basic principles of writing clean code that we covered in the first part of this document, we can at least reduce the chances of introducing unnecessary complexity to our code.

#### Pointer Mutability
We've already looked at the problem of mutability in the context of globally or largely scoped variables. However, mutability is not necessarily always a bad thing, and I am by no means an advocate for writing 100% pure functional programs. Mutability is a powerful tool, but we should really only ever use it when it's necessary. Let's have a look at a code example illustrating why:

```go
func (store *UserStore) Insert(user *User) error {
    if store.userExists(user.ID) {
        return ErrItemAlreaydExists
    }
    store.users[user.ID] = user
    return nil
}

func (store *UserStore) userExists(id int64) bool {
    _, ok := store.users[id]
    return ok
}
```

At first glance, this doesn't seem too bad. In fact, it might even seem like a rather simple insert function for a common list structure. We accept a pointer as input, and if no other users with this `id` exist, then we insert the provided user pointer into our list. Then, we use this functionality in our public API for creating new users:

```go
func CreateUser(w http.ResponseWriter, r *http.Request) {
    user, err := parseUserFromRequest(r)
    if err != nil {
        http.Error(w, err, http.StatusBadRequest)
        return
    }
    if err := insertUser(w, user); err != nil {
      http.Error(w, err, http.StatusInternalServerError)
      return
    }
}

func insertUser(w http.ResponseWriter, user User) error {
  	if err := store.Insert(user); err != nil {
        return err
    }
  	user.Password = ""
	  return json.NewEncoder(w).Encode(user)
}
```

Once again, at first glance, everything looks fine. We parse the user from the received request and insert the user struct into our store. Once we have successfully inserted our user into the store, we then set the password to be an empty string before returning the user as a JSON object to our client. This is all quite common practice, typically when returning a user object whose password has been hashed, since we don't want to return the hashed password.

However, imagine that we are using an in-memory store based on a `map`. This code will produce some unexpected results. If we check our user store, we'll see that the change we made to the users password in the HTTP handler function also affected the object in our store. This is because the pointer address returned by `parseUserFromRequest` is what we populated our store with, rather than an actual value. Therefore, when making changes to the dereferenced password value, we end up changing the value of the object we are pointing to in our store.

This is a great example of why both mutability and variable scope can cause some serious issues and bugs when used incorrectly. When passing pointers as an input parameter of a function, we are expanding the scope of the variable whose data is being pointed to. Even more worrying is the fact that we are expanding the scope to an undefined level. We are *almost* expanding the scope of the variable to the global level. As demonstrated by the above example, this can lead to disastrous bugs that are particularly difficult to find and eradicate.

Fortunately, the fix for this is rather simple:

```go
func (store *UserStore) Insert(user User) error {
    if store.userExists(user.ID) {
        return ErrItemAlreaydExists
    }
    store.users[user.ID] = &user
    return nil
}
```

Instead of passing a pointer to a `User` struct, we are now passing in a copy of a `User`. We are still storing a pointer to our store; however, instead of storing the pointer from outside of the function, we are storing the pointer to the copied value, whose scope is inside the function. This fixes the immediate problem but might still cause issues further down the line if we aren't careful. Consider this code:

```go
func (store *UserStore) Get(id int64) (*User, error) {
    user, ok := store.users[id]
    if !ok {
        return EmptyUser, ErrUserNotFound
    }
    return store.users[id], nil
}
```

Again, this is a very standard implementation of a getter function for our store. However, it's still bad code because we are once again expanding the scope of our pointer, which may end up causing unexpected side effects. When returning the actual pointer value, which we are storing in our user store, we are essentially giving other parts of our application the ability to change our store values. This is bound to cause confusion. Our store should be the only entity allowed to make changes to its values. The easiest fix for this is to return a value of `User` rather than returning a pointer.

> NOTE: Consider the case where our application uses multiple threads. In this scenario, passing pointers to the same memory location can also potentially result in a race condition. In other words, we aren't only potentially corrupting our data&mdash;we could also cause a panic from a data race.

Please keep in mind that there is intrinsically nothing wrong with returning pointers. However, the expanded scope of variables (and the number of owners pointing to those variables) is the most important consideration when working with pointers. This is what categorises our previous example as a smelly operation. This is also why common Go constructors are absolutely fine:

```go
func AddName(user *User, name string) {
    user.Name = name
}
```

This is *okay* because the variable scope, which is defined by whoever invokes the function, remains the same after the function returns. Combined with the fact that the function invoker remains the sole owner of the variable, this means that the pointer cannot be manipulated in an unexpected manner.

### Closures Are Function Pointers

Before we get into the next topic of using interfaces in Go, I would like to introduce a common alternative. It's what C programmers know as "function pointers" and what most other programming languages call <strong>closures</strong>. A closure is simply an input parameter like any other, except it represents (points to) a function that can be invoked. In JavaScript, it's quite common to use closures as callbacks, which are just functions that are invoked after some asynchronous operation has finished. In Go, we don't really have this notion. We can, however, use closures to partially overcome a different hurdle: The lack of generics.

Consider the following function signature:

```go
func something(closure func(float64) float64) float64 { ... }
```

Here, `something` takes another function (a closure) as input and returns a `float64`. The input function takes a `float64` as input and also returns a `float64`. This pattern can be particularly useful for creating a loosely coupled architecture, making it easier to to add functionality without affecting other parts of the code. Suppose we have a struct containing data that we want to manipulate in some form. Through this structure's `Do()` method, we can perform operations on that data. If we know the operation ahead of time, we can obviously handle that logic directly in our `Do()` method:

```go
func (datastore *Datastore) Do(operation Operation, data []byte) error {
  switch(operation) {
  case COMPARE:
    return datastore.compare(data)
  case CONCAT:
    return datastore.add(data)
  default:
    return ErrUnknownOperation
  }
}
```

But as you can imagine, this function is quite rigid&mdash;it performs a predetermined operation on the data contained in the `Datastore` struct. If at some point we would like to introduce more operations, we'd end up bloating our `Do` method with quite a lot of irrelevant logic that would be hard to maintain. The function would have to always care about what operation it's performing and to cycle through a number of nested options for each operation. It might also be an issue for developers wanting to use our `Datastore` object who don't have access to edit our package code, since there is no way of extending structure methods in Go as there is in most OOP languages.

So instead, let's try a different approach using closures:

```go
func (datastore *Datastore) Do(operation func(data []byte, data []byte) ([]byte, error), data []byte) error {
  result, err := operation(datastore.data, data)
  if err != nil {
    return err
  }
  datastore.data = result
  return nil
}

func concat(a []byte, b []byte) ([]byte, error) {
  ...
}

func main() {
  ...
  datastore.Do(concat, data)
  ...
}
```

You'll notice immediately that the function signature for `Do` ends up being quite messy. We also have another issue: The closure isn't particularly generic. What happens if we find out that we actually want the `concat` to be able to take more than just two byte arrays as input? Or if we want to add some completely new functionality that may also need more or fewer input values than `(data []byte, data []byte)`?

One way to solve this issue is to change our `concat` function. In the example below, I have changed it to only take a single byte array as an input argument, but it could just as well have been the opposite case:

```go
func concat(data []byte) func(data []byte) ([]byte, error) {
  return func(concatting []byte) ([]byte, error) {
    return append(data, concatting), nil
  }
}

func (datastore *Datastore) Do(operation func(data []byte) ([]byte, error)) error {
  result, err := operation(datastore.data)
  if err != nil {
    return err
  }
  datastore.data = result
  return nil
}

func main() {
  ...
  datastore.Do(compare(data))
  ...
}
```

Notice how we've effectively moved some of the clutter out of the `Do` method signature and into the `concat` method signature. Here, the `concat` function returns yet another function. Within the returned function, we store the input values originally passed in to our `concat` function. The returned function can therefore now take a single input parameter; within our function logic, we will append it to our original input value. As a newly introduced concept, this may seem quite strange. However, it's good to get used to having this as an option; it can help loosen up logic coupling and get rid of bloated functions.

In the next section, we'll get into interfaces. Before we do so, let's take a short moment to discuss the difference between interfaces and closures. First, it's worth noting that interfaces and closures definitely solve some common problems. However, the way that interfaces are implemented in Go can sometimes make it tricky to decide whether to use interfaces or closures for a particular problem. Usually, whether an interface or a closure is used isn't really of importance; the right choice is whichever one solves the problem at hand. Typically, closures will be simpler to implement if the operation is simple by nature. However, as soon as the logic contained within a closure becomes complex, one should strongly consider using an interface instead.  

Dave Cheney has an excellent write-up on this topic, as well as a talk:

* https://dave.cheney.net/2016/11/13/do-not-fear-first-class-functions
* https://www.youtube.com/watch?v=5buaPyJ0XeQ&t=9s

Jon Bodner also has a related talk:

* https://www.youtube.com/watch?v=5IKcPMJXkKs

### Interfaces in Go

In general, Go's approach to handling `interface`s is quite different from those of other languages. Interfaces aren't explicitly implemented like they would be in Java or C#; rather, they are implicitly created if they fulfill the contract of the interface. As an example, this means that any `struct` that has an `Error()` method implements (or "fulfills") the `Error` interface and can be returned as an `error`. This manner of implementing interfaces is extremely easy and makes Go feel more fast paced and dynamic.

However, there are certainly disadvantages with this approach. As the interface implementation is no longer explicit, it can be difficult to see which interfaces are implemented by a struct. Therefore, it's common to define interfaces with as few methods as possible; this makes it easier to understand whether a particular struct fulfills the contract of the interface.

An alternative is to create constructors that return an interface rather than the concrete type:

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}

type NullWriter struct {}

func (writer *NullWriter) Write(data []byte) (n int, err error) {
    // do nothing
    return len(data), nil
}

func NewNullWriter() io.Writer {
    return &NullWriter{}
}
```

The above function ensures that the `NullWriter` struct implements the `Writer` interface. If we were to delete the `Write` method from `NullWriter`, we would get a compilation error. This is a good way of ensuring that our code behaves as expected and that we can rely on the compiler as a safety net in case we try to write invalid code.

In certain cases, it might not be desirable to write a constructor, or perhaps we would like for our constructor to return the concrete type, rather than the interface. As an example, the `NullWriter` struct has no properties to populate on initialisation, so writing a constructor is a little redundant. Therefore, we can use the less verbose method of checking interface compatibility:

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}

type NullWriter struct {}
var _ io.Writer = &NullWriter{}
```

In the above code, we are initialising a variable with the Go `blank identifier`, with the type assignment of `io.Writer`. This results in our variable being checked to fulfill the `io.Writer` interface contract, before being discarded. This method of checking interface fulfillment also makes it possible to check that several interface contracts are fulfilled:

```go
type NullReaderWriter struct{}
var _ io.Writer = &NullWriter{}
var _ io.Reader = &NullWriter{}
```

From the above code, it's very easy to understand which interfaces must be fulfilled; this ensures that the compiler will help us out during compile time. Therefore, this is generally the preferred solution for checking interface contract fulfillment.

There's yet another method of trying to be more explicit about which interfaces a given struct implements. However, this third method actually achieves the opposite of what we want. It involves using embedded interfaces as a struct property.

> <em>Wait what? &ndash; Presumably most people</em>

Let's rewind a bit before we dive deep into the forbidden forest of smelly Go. In Go, we can use embedded structs as a type of inheritance in our struct definitions. This is really nice, as we can decouple our code by defining reusable structs.

```go
type Metadata struct {
    CreatedBy types.User
}

type Document struct {
    *Metadata
    Title string
    Body string
}

type AudioFile struct {
    *Metadata
    Title string
    Body string
}
```

Above, we are defining a `Metadata` object that will provide us with property fields that we are likely to use on many different struct types. The neat thing about using the embedded struct, rather than explicitly defining the properties directly in our struct, is that it has decoupled the `Metadata` fields. Should we choose to update our `Metadata` object, we can change it in just a single place. As we've seen several times so far, we want to ensure that a change in one place in our code doesn't break other parts. Keeping these properties centralised makes it clear that structures with an embedded `Metadata` have the same properties&mdash;much like how structures that fulfill interfaces have the same methods.

Now, let's look at an example of how we can use a constructor to further prevent breaking our code when making changes to our `Metadata` struct:

```go
func NewMetadata(user types.User) Metadata {
    return &Metadata{
        CreatedBy: user,
    }
}

func NewDocument(title string, body string) Document {
    return Document{
        Metadata: NewMetadata(),
        Title: title,
        Body: body,
    }
}
```

Suppose that at a later point in time, we decide that we'd also like a `CreatedAt` field on our `Metadata` object. We can now easily achieve this by simply updating our `NewMetadata` constructor:

```go
func NewMetadata(user types.User) Metadata {
    return &Metadata{
        CreatedBy: user,
        CreatedAt: time.Now(),
    }
}
```

Now, both our `Document` and `AudioFile` structures are updated to also populate these fields on construction. This is the core principle behind decoupling and an excellent example of ensuring maintainability of code. We can also add new methods without breaking our existing code:

```go
type Metadata struct {
    CreatedBy types.User
    CreatedAt time.Time
    UpdatedBy types.User
    UpdatedAt time.Time
}

func (metadata *Metadata) AddUpdateInfo(user types.User) {
    metadata.UpdatedBy = user
    metadata.UpdatedAt = time.Now()
}
```

Again, without breaking the rest of our codebase, we've managed to introduce new functionality. This kind of programming makes implementing new features very quick and painless, which is exactly what we are trying to achieve by writing clean code.

Let's return to the topic of interface contract fulfillment using embedded interfaces. Consider the following code as an example:

```go
type NullWriter struct {
    Writer
}

func NewNullWriter() io.Writer {
    return &NullWriter{}
}
```

The above code compiles. Technically, we are implementing the interface of `Writer` in our `NullWriter`, as `NullWriter` will inherit all the functions that are associated with this interface. Some see this as a clear way of showing that our `NullWriter` is implementing the `Writer` interface. However, we must be careful when using this technique.

```go
func main() {
    w := NewNullWriter()

    w.Write([]byte{1, 2, 3})
}
```

As mentioned before, the above code will compile. The `NewNullWriter` returns a `Writer`, and everything is hunky-dory according to the compiler because `NullWriter` fulfills the contract of `io.Writer`, via the embedded interface. However, running the code above will result in the following:

> panic: runtime error: invalid memory address or nil pointer dereference

What happened? An interface method in Go is essentially a function pointer. In this case, since we are pointing to the function of an interface, rather than an actual method implementation, we are trying to invoke a function that's actually a `nil` pointer. To prevent this from happening, we would have to provide the `NulllWriter` with a struct that fulfills the interface contract, with actual implemented methods.

```go
func main() {
  w := NullWriter{
    Writer: &bytes.Buffer{},
  }

    w.Write([]byte{1, 2, 3})
}
```

> NOTE: In the above example, `Writer` is referring to the embedded `io.Writer` interface. It is also possible to invoke the `Write` method by accessing this property with `w.Writer.Write()`.

We are no longer triggering a panic and can now use the `NullWriter` as a `Writer`. This initialisation process is not much different from having properties that are initialised as `nil`, as discussed previously. Therefore, logically, we should try to handle them in a similar way. However, this is where embedded interfaces become a little difficult to work with. In a previous section, it was explained that the best way to handle potential `nil` values is to make the property in question private and create a public *getter* method. This way, we could ensure that our property is, in fact, not `nil`. Unfortunately, this is simply not possible with embedded interfaces, as they are by nature always public.

Another concern raised by using embedded interfaces is the potential confusion caused by partially overwritten interface methods: 

```go
type MyReadCloser struct {
  io.ReadCloser
}

func (closer *ReadCloser) Read(data []byte) { ... }

func main() {
  closer := MyReadCloser{}
  
  closer.Read([]byte{1, 2, 3}) 	// works fine
  closer.Close() 		// causes panic
  closer.ReadCloser.Closer() 		// no panic 
}
```

Even though this might look like we're overriding methods, which is common in languages such as C# and Java, we actually aren't. Go doesn't support inheritance (and thus has no notion of a superclass). We can imitate the behaviour, but it is not a built-in part of the language. By using methods such as interface embedding without caution, we can create confusing and potentially buggy code, just to save a few more lines.

> NOTE: Some argue that using embedded interfaces is a good way of creating a mock structure for testing a subset of interface methods. Essentially, by using an embedded interface, you won't have to implement all of the methods of the interface; rather, you can choose to implement only the few methods that you'd like to test. Within the context of testing/mocking, I can see this argument, but I am still not a fan of this approach.

Let's quickly get back to clean code and proper usage of interfaces. It's time to discuss using interfaces as function parameters and return values. The most common proverb for interface usage with functions in Go is the following:

> <em>Be conservative in what you do; be liberal in what you accept from others &ndash; Jon Postel</em>

> FUN FACT: This proverb actually has nothing to do with Go. It's taken from an early specification of the TCP networking protocol.

In other words, you should write functions that accept an interface and return a concrete type. This is generally good practice and is especially useful when doing tests with mocking. As an example, we can create a function that takes a writer interface as its input and invokes the `Write` method of that interface:

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

Let's assume that we are writing to a file when our application is running, but we don't want to write to a new file for all tests that invoke this function. We can implement a new mock type that will basically do nothing. Essentially, this is just basic dependency injection and mocking, but the point is that it is extremely easy to achieve in Go:

```go
type NullWriter struct {}

func (w *NullWriter) Write(data []byte) (int, error) {
    return len(data), nil
}

func TestFn(t *testing.T) {
    ...
    pipe := NewPipe(NullWriter{})
    ...
}
```

> NOTE: There is actually already a null writer implementation built into the `ioutil` package named `Discard`.

When constructing our `Pipe` struct with `NullWriter` (rather than a different writer), when invoking our `Save` function, nothing will happen. The only thing we had to do was add four lines of code. This is why it is encouraged to make interfaces as small as possible in idiomatic Go&mdash;it makes it especially easy to implement patterns like the one we just saw. However, this implementation of interfaces also comes with a <em>huge</em> downside.

### The Empty `interface{}`
Unlike other languages, Go does not have an implementation for generics. There have been many proposals for one, but all have been turned down by the Go language team. Unfortunately, without generics, developers must try to find creative alternatives, which very often involves using the empty `interface{}`. This section describes why these often <em>too</em> creative implementations should be considered bad practice and unclean code. There will also be examples of appropriate usage of the empty `interface{}` and how to avoid some pitfalls of writing code with it.

As mentioned in a previous section, Go determines whether a concrete type implements a particular interface by checking whether the type implements the <em>methods</em> of that interface. So what happens if our interface declares no methods, as is the case with the empty interface?

```go
type EmptyInterface interface {}
```

The above is equivalent to the built-in type `interface{}`. A natural consequence of this is that we can write generic functions that accept any type as arguments. This is extremely useful for certain kinds of functions, such as print helpers. Interestingly, this is actually what makes it possible to pass in any type to the `Println` function from the `fmt` package:

```go
func Println(v ...interface{}) {
    ...
}
```

In this case, `Println` isn't just accepting a single `interface{}`; rather, the function accepts a <em>slice</em> of types that implement the empty `interface{}`. As there are no methods associated with the empty `interface{}`, <em>all</em> types are accepted, even making it possible to feed `Println` with a slice of different types. This is a very common pattern when handling string conversion (both from and to a string). Good examples of this come from the `json` standard library package:

```go
func InsertItemHandler(w http.ResponseWriter, r *http.Request) {
    var item Item
    if err := json.NewDecoder(r.Body).Decode(&item); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    if err := db.InsertItem(item); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    w.WriteHeader(http.StatsOK)
}
```

All the less elegant code is contained within the `Decode` function. Thus, developers using this functionality won't have to worry about type reflection or type casting; we just have to worry about providing a pointer to a concrete type. This is good because the `Decode()` function is technically returning a concrete type. We are passing in our `Item` value, which will be populated from the body of the HTTP request. This means we won't have to deal with the potential risks of handling the `interface{}` value ourselves.

However, even when using the empty `interface{}` with good programming practices, we still have some issues. If we pass in a JSON string that has nothing to do with our `Item` type but is still valid JSON, we won't receive an error&mdash;our `item` variable will just be left with the default values. So, while we don't have to worry about reflection and casting errors, we will still have to make sure that the message sent from our client is a valid `Item` type. Unfortunately, as of writing this document, there is no simple or good way to implement these types of generic decoders without using the empty `interface{}` type.

The problem with using `interface{}` in this manner is that we are leaning towards using Go, a statically typed language, as a dynamically typed language. This becomes even clearer when looking at poor implementations of the `interface{}` type. The most common example of this comes from developers trying to implement a generic store or list of some sort.

Let's look at an example of trying to implement a generic HashMap package that can store any type using `interface{}`. 

```go
type HashMap struct {
    store map[string]interface{}
}

func (hashmap *HashMap) Insert(key string, value interface{}) {
    hashmap.store[key] = value
}

func (hashmap *HashMap) Get(key string) (interface{}, error) {
    value, ok := hashmap.store[key]
    if !ok {
        return nil, ErrKeyNotFoundInHashMap
    }
    return value
}
```

> NOTE: I have omitted thread safety from this example to keep it simple.

Please keep in mind that the implementation pattern shown above is actually used in quite a lot of Go packages. It is even used in the standard library `sync` package for the `sync.Map` type. So what's the problem with this implementation? Well, let's have a look at an example of using the package:

```go
func SomeFunction(id string) (Item, error) {
    itemIface, err := hashmap.Get(id)
    if err != nil {
        return EmptyItem, err
    }
    item, ok := itemIface.(Item)
    if !ok {
        return EmptyItem, ErrCastingItem
    }
    return item, nil
}
```

At first glance, this looks fine. However, we'll start getting into trouble if we add <em>different</em> types to our store, something that's currently allowed. There is nothing preventing us from adding something other than the `Item` type. So what happens when someone starts adding other types into our HashMap, like a pointer `*Item` instead of an `Item`? Our function now might return an error. Worst of all, this might not even be caught by our tests. Depending on the complexity of the system, this could introduce some bugs that are particularly difficult to debug.

This type of code should never reach production. Remember: Go does not (yet) support generics. That's just a fact that developers must accept for the time being. If we want to use generics, then we should use a different language that does support generics rather than relying on dangerous hacks.

So, how do we prevent this code from reaching production? The simplest solution is to just write the functions with concrete types instead of using `interface{}` values. Of course, this is not always the best approach, as there might be some functionality within the package that is not trivial to implement ourselves. Therefore, a better approach may be to create wrappers that expose the functionality we need but still ensure type safety:

```go
type ItemCache struct {
  kv tinykv.KV
} 

func (cache *ItemCache) Get(id string) (Item, error) {
  value, ok := cache.kv.Get(id)
  if !ok {
    return EmptyItem, ErrItemNotFound
  }
  return interfaceToItem(value)
}

func interfaceToItem(v interface{}) (Item, error) {
  item, ok := v.(Item)
  if !ok {
    return EmptyItem, ErrCouldNotCastItem
  }
  return item, nil
}

func (cache *ItemCache) Put(id string, item Item) error {
  return cache.kv.Put(id, item)
}
```

> NOTE: Implementations of other functionalities of the `tinykv.KV` cache have been omitted for the sake of brevity.

The wrapper above now ensures that we are using the actual types and that we are no longer passing in `interface{}` types. It is therefore no longer possible to accidentally populate our store with a wrong value type, and we have restricted our casting of types as much as possible. This is a very straightforward way of solving our issue, even if somewhat manually.

## Summary

First of all, thank you for making it all the way through this article! I hope it has provided some insight into clean code and how it helps ensure maintainability, readability, and stability in any codebase.

Let's briefly sum up all the topics we've covered:

- <strong>Functions</strong>&mdash;A function's name should reflect its scope; the smaller the scope of a function, the more specific its name. Ensure that all functions serve a single purpose in as few lines as possible. A good rule of thumb is to limit your functions to 5&ndash;8 lines and to only accept 2&ndash;3 arguments.

- <strong>Variables</strong>&mdash;Unlike functions, variables should assume more generic names as their scope becomes smaller. It's also recommended that you limit the scope of a variable as much as possible to prevent unintentional modification. On a similar note, you should keep the modification of variables to a minimum; this becomes an especially important consideration as the scope of a variable grows.

- <strong>Return Values</strong>&mdash;Concrete types should be returned whenever possible. Make it as difficult as possible for users of your package to make mistakes and as easy as possible for them to understand the values returned by your functions.

- <strong>Pointers</strong>&mdash;Use pointers with caution, and limit their scope and mutability to an absolute minimum. Remember: Garbage collection only assists with memory management; it does not assist with all of the other complexities associated with pointers.

- <strong>Interfaces</strong>&mdash;Use interfaces as much as possible to loosen the coupling of your code. Hide any code using the empty `interface{}` as much as possible from end users to prevent it from being exposed.

As a final note, it's worth mentioning that the notion of clean code is particularly subjective, and that likely won't ever change. However, much like my statement concerning `gofmt`, I think it's more important to find a common standard than something that everyone agrees with; the latter is extremely difficult to achieve.

It's also important to understand that fanaticism is never the goal with clean code. A codebase will most likely never be fully 'clean,' in the same way that your office desk probably isn't either. There's certainly room for you to step outside the rules and boundaries covered in this article. However, remember that the most important reason for writing clean code is to help yourself and other developers. We support engineers by ensuring stability in the software we produce and by making it easier to debug faulty code. We help our fellow developers by ensuring that our code is readable and easily digestible. We help <em>everyone</em> involved in the project by establishing a flexible codebase that allows us to quickly introduce new features without breaking our current platform. We move quickly by going slowly, and everyone is satisfied.

I hope you will join this discussion to help the Go community define (and refine) the concept of clean code. Let's establish a common ground so that we can improve software&mdash;not only for ourselves but for the sake of everyone.
