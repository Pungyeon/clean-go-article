---
TODO:
- create a section on wrapping functions as a method of loosely coupling your code with one another, rather than making direct changes to your logic.
---

```go
// HTTPClientWrapper is a wrapper for the standard http client
// which ensures that a non 2xx return code is regarded as an error
type HTTPClientWrapper struct {
	client *http.Client
	check  func(status int) bool
}

// NewHTTPClientWrapper will initialise and return a new http client wrapper
func NewHTTPClientWrapper(checker func(status int) bool) *HTTPClientWrapper {
	return &HTTPClientWrapper{
		check: checker,
		client: &http.Client{
			Transport: &http.Transport{
				MaxIdleConnsPerHost: 20,
			},
			Timeout: time.Duration(1) * time.Second,
		},
	}
}

// Get will invoke the get function on the wrapped client and check the status
// to ensure it is a 2xx return code
func (wrapper *HTTPClientWrapper) Get(url string) (*http.Response, error) {
	resp, err := wrapper.client.Get(url)
	if err != nil {
		return &http.Response{}, err
	}
	if !wrapper.check(resp.StatusCode) {
		return &http.Response{}, fmt.Errorf(
			"error on sending request to [%s] - returned with status: %s",
			url, resp.Status)
	}
	return resp, err
}

// Do will invoke the Do function on the wrapped client
func (wrapper *HTTPClientWrapper) Do(req *http.Request) (*http.Response, error) {
	return wrapper.client.Do(req)
}
```





# Clean Golang Code

## Preface

The motivation behind writing this document, is to create a resource (and eventually a reference) for the Go community, which will help developers write cleaner code. This benefits every one of us. Whether we are writing code by ourselves, or writing code in larger teams. Establishing good paradigms for writing clean code and ensuring that this is available for everyone, will help prevent many meaningless hours on trying to understand and parse others (and our own) code.  

<center style="font-style: italic">We don’t read code, we <b>decode</b> it - Peter Seibel </center>

The matter of the fact is, as Peter Seibel put it. We decode code and we honestly can't help encoding it, in some way, shapre or form. This document, will be a precursor for us, to make sure that our encoding method is effective. We want our code to be usable, readable and maintainable.

This document will start with a simple and short introduction to the fundamentals behind writing clean code and will thereafter transition into concrete refactoring examples, more specific to Go. The aim of the document is to deliver the message of how easy it is to write clean code and how easy is it to write code, when it's clean.

##### A short word on `gofmt`
I would like to take a few sentences to make my stance on `gofmt` very clear. There are extremely many things that I disagree with, when it comes to `gofmt`. I prefer snake case over camel case, I quite like my constant variables to be upper case and I also have many opinions on bracket placement. *That being said*: `gofmt` is what enables us to have a common standard for writing Go code. All Go code, will look somewhat similar and it ensures that no Go code becomes too exoteric. I think that this is overall extremely positive. I appreciate immensely, that all Go programmers are somewhat restricted to write similar code, despite being very unhappy with some of the formatting rules. In my opinion, I value homogenous code more than expressive freedom.

## Context
* [Introduction to Clean Code](#Introduction-to-Clean-Code)
    * [Test Driven Development](#Test-Driven-Development)
    * [Naming](#Naming)
    * * [Comments](#Comments)
    	* [Function Naming](#Function-Naming)
    	* [Variable Naming](#Variable-Naming)
    * [Cleaning Functions](#Cleaning-Functions)
      * [Function Length](#Function-Length)
      * [Function Signatures](#Function-Length)
    * [Variable Scope](#Variable-Scope)
    * [Variable Declaration](#Variable-Declaration)
    
* [Clean Golang](#Clean-Golang)
		* [Return Values](#Return Values) 
      * [Returning Defined Errors](#Returning-Defined-Errors)
      * [Returning Dynamic Errors](#Returning-Dynamic-Errors)
      * [Returning Other Values](#Returning-Other-Values)
    * [Pointers in Go](#Pointers-in-Go)
    * [Using `goto` in Go](#Using-`goto`-in-Go)
    * [Closures are Function Pointers](#Closures-are-Function-Pointers)
    * [Interfaces in Go](#Interfaces-in-Go)
    * [The empty `interface{}`](#The-empty-`interface{}`)
* [Go Code Generation](#Go-Code-Generation)
* [Balancing Performance with Cleanliness](#Balancing-Performance-with-Cleanliness)
    * Immutability - Variable Scope Continue 
* [Index](#Index)

## Introduction to Clean Code

Clean Code, is the pragmatic concept of ensuring readable and maintanable code. Clean Code establishes trust in the codebase and will steer developers away from introducing bugs. Clean Code will also establish much more stability in development speed, which typically will take a nose dive in the later stages of projects, due to higher risk of increasing bugs when introducing changes, as the codebase expands. 

### Test Driven Development

The core of creating clean code stems from creating good tests. Writing good tests helps create clean code, as it invites developers to think about the outcomes and test coverage of functions / functionality. It's easier to test a function that is only 4 lines, rather than a function, which is 40. In the same manner, a function which is 4 lines, is typically easier to understand than a function of 40 lines. Therefore, when using test driven development, the resulting code is much more likely to be of a cleaner nature. 

The next important part of test driven development, which is very closely related to clean code, is the TDD cycle:

1. Write a test which fails
2. Make the test pass
3. Refactor code
4. Repeat

Step three of the cycle, ensures that we can refactor our code as we are writing it. The tests ensure that our refactor doesn't change the outcome of our functions and we can therefore, essentially, go crazy refactoring our code to be as clean as possible. As we go along, and our codebase expands, we will still have our tests, to make sure that our refactoring will not affect the outcome of our functions.

###  Naming

#### Comments
First things first: I want to address the topic of comments. Unecessary comments are the biggest indicator of code smell. Comments are usually added into a code base, because something is so unclear, that it's deemed necessary to explain with a comment. Of course, this is not always the case. In Go, all according to `gofmt` all public variables and functions, should be annotated. I think this is absolutely fine, as this makes documentation easy. However, I therefore always want to distinguish between comments which enable auto-generated documentation and all other comments. Annotation comments, for documentation, should be written like documentation. They should be high level and concern the logical implementation as little as possible, other than on the highest abstraction level. 

The reasoning behind this, is that there are other ways to explain code and ensure that code is being written comprehensibly and expressively. If the code is neither of the two, some people find it acceptable to replace this, with a comment explaining the logic. The matter of the fact is, that most people will not read comments, because it's very intrusive to the experience of reading code. However, let's start from the beginning. Bad comments:

```go
// iterate over the range 0 to 9 
// and invoke the doSomething function
// for each iteration
for i := 0; i < 10; i++ {
  doSomething(i)
}
```

This comment, is what I call a tutorial comment. It's pretty common in tutorials, which explain the low level functionality of a language (or programming on a more general level). The matter of the fact is, that these comments are absolutely useless in production code. Hopefully, we aren't collaborating with other programmers, who don't understand that principles behind the language we have chosen to write in, or even worse, don't understand simple principles of programming. As programmers, we don't have to read the comment, we know this is happening, by reading the code. Hence the proverb:

<center style="margin: 0 100px 20px 100px; font-style: italic">"Document why, not how." - Venkat Subramaniam</center>

Following this logic, we can now change our comment, to explain why we are iterating from the range zero to nine:

```go
// instatiate 10 threads to handle upcoming work load
for i := 0; i < 10; i++ {
  doSomething(i)
}
```

Now we can understand why we are iterating and we can tell what we are doing, by reading the code. The worrying part about this comment is, that this probably should be necessary to express in prose. We can quite easily express this directly in our code instead:

```go
for worker_id := 0; worker_id < 10; worker_id++ {
  instantiateThread(worker_id)
}
```

With just a few changes of our variable and function names, we have established explanations of our action, directly in our code. This is much clearer for the reader, because the reader won't have to read the comment and then map the prose comment to the code. Instead, they can read the code and immediately understand everything which is going on. 

Of course, this example, was relatively easy. It's unfortunately not always this easy and writing clear and expressive code becomes increasingly difficult, together with code complexity. So let's have a look at some methods, which will help make this task easier. 

#### Function Naming

We will start by discussing the naming of our functions. The general rule for function naming is really simple: The more specific the function, the more general the name. In other words, this means that we want to start with a very broad and short function name, such as `Run` or `Parse`, which describes thes general functionality. Let's imagine that we are creating a configuration parser. Following this naming convention, our top level of abstraction might look something like the following:

```go
func main() {
    configpath := flag.String("config-path", "", "configuration file path")
    flag.Parse()

    config, err := configuration.Parse(*configpath)
    
    ...
}
```

Our focus being on the naming of the `Parse` function. Despite this function name being very short and general, it's actually quite clear what this function attempts to achieve. When we go one layer deeper, our function naming will become slightly more specific:

```go
func Parse(filepath string) (Config, error) {
    switch getFileExtension(filepath) {
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

More specific, but not much, just appropriately. It's still clear what the difference is between the parent function and the sub-functions, without being overly specific. This enables each sub-function to appear clear on it's own, whereas if we had named the `parseJSON` function `json` instead. This would not have been the case. 

Notice that `getFileExtension` is actually a little more specific. However, this is because the functionality of this function is, in fact, quite specific:

```go
func getFileExtension(filepath string) string {
    segemnts := strings.Split(filepath, ".")
    return segments[len(segments)-1]
}
```

This kind of logical progression in our function names, makes the code easier to follow and will make the code much easier to read. When we think about the opposite approach to function naming, it becomes even more clear why. If our highest level of abstraction becomes too specific, we will end up with a function name such as `DetermineFileExtensionAndParseConfigurationFile`. This is horrendously difficult to read and just adds confusion, more than anything else. We are trying to be too specific too quickly and therefore we end up being confusing, despite our intention of trying to be clear. 

#### Variable Naming
Rather interestingly, the opposite is true for variables. Unlike functions, our variable naming should progress from more to less specific. 

<center style="margin: 0 100px 20px 100px; font-style: italic">"You shouldn’t name your variables after their types for the same reason you wouldn’t name your pets 'dog' or 'cat'." - Dave Cheney</center>

The reason why we want to become less and less specific with our variables, is the fact that it becomes clearer and clearer for the reader, what the variable represents, the smaller the scope of the variable is. In the example of the previous function `getFileExtension`, the naming of the variable `segments`, could even be shortened to `s`, if we wanted to. The context of the variable is so clear, it is unecessary to explain our code further, with longer variable names. Another good example of this, would be in nested for loops. 


```go
func PrintBrandsInList(brands []BeerBrand) {
    for _, b := range brands { 
        fmt.Println(b)
    }
}
```

The reason why this is true, is because of the scope of the variable, rather than the abstraction layer, which is the guideline we would use for our function naming. The smaller the scope of a variable, the less important the actual naming is. In the above example, the `b` variable scope is so short, that we don't need to spend brain power on remembering what it represents. However, because the scope `brands` is slightly larger, when reading the code, we will use more brain power on remembering what these represent. When expanding the variable scope in the function below, it becomes even more apparent:

```go
func BeerBrandListToBeerList(beerBrands []BeerBrand) []Beer {
    var beerList []Beer
    for _, brand := range beerBrandList {
        for _, beer := range brand {
            beerList = append(beerList, beer)
        }
    }
    return beerList
}
```

Now, let's imagine that we apply the opposite logic, to see what this looks like:

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

Even though the function might still be readable, due to it's brevity, there is a strange off-putting feeling, when reading through the function. Should the scope of the variables or the logic of the function expand, this off-putting feel, becomes even worse and could potentially spiral into complete confusion. However, while on the topic of functions and their brevity, let's dive into the next topic of writing clean code.

### Cleaning Functions
#### Function Length

In the words of Robert C. Martin: 

<center style="margin: 0 100px 20px 100px; font-style: italic">"How small should a function be? Smaller than that!"</center>

When writing clean code, our primary goal is to make our code easily digestable. The most effective way to do this, is to make our functions as small as possible. It's important to understand, that this is not necessarily to avoid code duplication. The more promenant reason for this is to heighten the code comprehension. Another way of explaining this, is to look at a function description: 

```
fn GetItem:
    - parse json input for order id
    - get user from context
    - check user has appropriate role
    - get order from database
```

When using small functions (typically 5-8 lines in Go), we can create a function that reads almost as easily as our description:

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
    if !GetUserFromContext(ctx).IsAdmin() {
	      return NullItem, ErrInsufficientPrivliges
    }
    return db.GetItem(order.ItemID)
}
```

Using smaller functions also has a side-effect of eliminating another horrible habit of writing code: indentation hell. Indentation hell, typically occurs when a chain of if statements are clumsily inserted into a function. This makes the code very, very difficult to parse (for human beings) and should be eliminated whenever spotted. This is particularly common when working with `interface{}` and using type casting:

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

Not only can this kind of code result in a really bad experience for other programmers, who will have to fight to understand the flow of the code. Should the logic in our `if` statements expand, it becomes exponentially more difficult to figure out which statement returns what. It is unfortunately not uncommon to find this kind of implementation in code. I have even bumped into examples of the beginning `if` statement of a correspending `else` statement, was on another page of my monitor. Having to scroll up and  down a page, while trying to figure out what a function does, is not ideal. Even though, we don't have to scroll on our page to see the corresponding `if else` statements in the above code sample, we are still scrolling with our eyes and maintaining state in our brain. Most programmers can quite easily contain this state for the function above, or worse examples. However, we have forced readers of our code, to use unecessary brain power. This may result in reader fatigue, should we repeat this mistake throughout our code. Constantly having to parse code like the above, will make reading the code more and more difficult, which we of course, want to avoid.

So, how do we clean this function? Foruntately, it's actually quite simple. On our first iteration, we will try to  ensure that we are returning an error as soon as we can. Instead of nested the `if` and `else` statements, we want to "push our code to the left". This is handled by returning from our function, as soon as we possibly can.  

```go
func GetItem(extension string) (Item, error) {
    refIface, ok := db.ReferenceCache.Get(extension)
    if !ok {
        return EmptyItem, errors.New("reference not found in cache")
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

Once we are done with this, we can split up our function into smaller functions as mentioned previously. A good rule of thumb being:  If the `value, err :=` pattern is repeated more than once in a function, this is an indication that we can split the logic of our code into smaller functions.

```go
func GetItem(extension string) (Item, error) {
    if ref, ok := getReference(extension) {
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
> For production code, one should elaborate on the code even further, by returning errors instead of a `bool` values. This makes it much easier to understand where the error is originating from. However, as these are just example functions, the `bool` values will suffice for now. Examples of returning errors more explicitly will be explained in more detail later.

The resulting clean version of our function, has resulted in a lot more lines of code. However, the code is so much easier to read. It's layered in an onion-style fashion, where we can ignore code details that we aren't interested in and dive deeper into the functions that we wish to know the workings behind. When we do deep-dive into the lower level functionality, it will be extremely easy to comprehend, because we will only have to understand 3-5 lines in this case. This example illustrates, that we cannot score the cleaniless of our code from the line count of our functions. The first function iteration was much shorter. However, it was artificially short and very difficult to read. In most cases cleaning code will, to begin with, expand the already existing code base, in terms of lines of code. However, the benefit of readability is far preferred. If you are ever in doubt about this, think of how you feel about the following function, which does the same:

```go
func GetItemIfActive(extension string) (Item, error) {
    if refIface,ok := db.ReferenceCache.Get(extension); ok {if ref,ok := refIface.(string); ok { if itemIface,ok := db.ItemCache.Get(ref); ok { if item,ok := itemIface.(Item); ok { if item.Active { return Item,nil }}}}} return EmptyItem, errors.New("reference not found in cache")
}
```

While we are on the topic. There are also a bunch of other side-effects that come along when writing in this style of code. Rather obviously, it makes our code much easier to test. It's much easier to get 100% code coverage on a function that is 4 lines (written by a sane person), than a function which is 400 lines. That's common sense. 

#### Function Signatures

Creating good function naming structure, makes it easier to read and understand the intent of code. Making our functions shorter, helps with the understanding of the content of the function logic.  The last part of cleaning our functions, will be to understand the context of the function input. With this, comes another easy to follow rule. Function signatures, should only contain one or two input parameters. On certain exceptional occasions, three can be acceptable, but this is where we should start considering a refactor. Much like the rule that our function should only be 5-8 lines long, this can seem quite extreme at first. However, I feel that this rule is more immediately demonstrably true. 

As an example, take the following function from the RabbitMQ introduction tutorial, to their Go library:

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

The function `QueueDeclare` takes six input parameters, which is quite extreme. The above code is somewhat possible to understand, because of the comments, but as mentioned earlier: Comments should be substituted with descriptive code. One good reason for this, is that there is nothing preventing us from invoking the `QueueDeclare` function without comments, making it look like this:

```go
q, err := ch.QueueDeclare("hello", false, false, false, false, nil)
```

Now, without looking at the previous code, try to remember what the fourth and fifth `false` represent. It's impossible, and it's inevitable that we will forget at some point. This can lead to costly mistakes, and bugs that are difficult to correct. The mistakes might even occur through incorrect comments. Imagine labelling the wrong input parameter. Correcting this mistake, will be unbearably difficult to correct, especially when familiarity with the code has deteriorated over time or was low to begin with. Therefore, it is recommended to replace these input parameters, with an 'Options' `struct` instead:

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

This solves both the problem of omitting comments or accidentally labelling the variables incorrectly. Of course, we can still confuse properties with the wrong value, but in these cases, it will be much easier to determine where our mistakes lies within the code. The ordering of the properties also do not matter anymore and therefore incorrectly ordering the input values, is no longer a worry. The last added bonus of this technique, is that we can use our Option `struct`, to infer default values of our functions input parameters. When structures in Go are declared, all properties are initialised to their default value. This means, that our `QueueDeclare` option, can actually be invoked in the following way:

```go
q, err := ch.QueueDeclare(QueueOptions{
    Name: "hello",
})
```

The rest of the values are by initialised to their default `false` values (except for `Arguments`, which, as an interface has a default value of `nil`). Not only are we much safer, we are more clear with our intentions and in this case, we could actually write less code. This is an all around win.

A last note on this, is that it's not always possible to change the function signatures. As in this case, we don't have control of our `QueueDeclare` function signature, since this is from the RabbitMQ library. It's not our code, we can't change it. However, we can wrap these functions, to suit our purposes:

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

Basically, we create a new structure `RMQChannel` which contains the `amqp.Channel` type, which has the `QueueDeclare` method. We then create our own version of this method, which essentially just calls the old version of the RabbitMQ library function. Our new method has all the advantages described before and we achieved this, without actually having access to changing any code in the RabbitMQ library.

We will use the idea of wrapping functions to introduce more clean and safe code later when discussing the `interface{}`.

// TODO : Add link?

### Variable Scope
// TODO : the sentence below is now out of place.

Another nice side-effect of writing smaller functions. Is that it can typically eliminate using longer lasting mutable variables. Writing code with global variables, is a pratice of the past, it doesn't belong in clean code. Now, why is that? Well, the problem with using global variables is that we make it very difficult for programmers to understand the current state of a variable. If a variable is global and mutable, then, by definition, it's value can be changed by any part of the codebase. At no point can you guarantee that this variable is going to be a specific value... This is a headache for everyone. This is yet another example of a trivial problem, which is exasterbate, when the codebase expands. Let's, look at a short example of how even larger scoped (not global) variables can cause problems. 

Larger scoped variables, also introduce the issue of variable shadowing as shown int he code taken from an article named: [`Golang scope issue - A feature bug: Shadow Variables`](https://idiallo.com/blog/golang-scopes):

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

This declares a new variable `val` in the the switch case `32` scope and has nothing to do with the variable declared in the first line of `main`. Of course, it can be argued that the Go syntax is a little tricky, which I don't necessarily disagree with, but there is a much worse issue at hand. The declaration of `var val string` as a mutable largely scoped variable, is completely unecessary. If we do a **very** simple refactor, we will no longer have this issue:

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

After our refactor, `val` is no longer mutated and the scope has been reduced. Again, keep in mind that these functions are very simple. Once this kind of code style becomes a part of larger more complex systems, it can be impossible to figure out, why errors are happening. We don't want this to happen. Not only because we generally dislike errors happening in software, but it is also disrespectful to our colleagues, and ourselves, that we are potentially wasting each others live's, having to debug this type of code. Let's take responsibility ourselves, rather than blaming the variable decalaration syntax in Go.

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

### Variable Declaration 

// TODO : I'm not sure this is all that great of a chapter. The *bad* example isn't particularly expressive of the actual issue which we are trying to display

Other than avoiding variable scope and mutability, we can also improve readability but keeping our variable declaration close to the logic. In C programming, it's common to see the following method for declaring variables:

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

This suffers from the same symptom as described in variable scope. Even though that these variables might not actually be re-assigned at any point, this kind of style, will keep the readers on their toes, in all the wrong ways. Much like computer memory, our brain has a limited amount to allocate from. Having to keep track of which variables could be mutated and whether or not something will mutate these items, will only make it more difficult to get a good overview of what is happening in the code. Figuring out the eventually returned value, can be a nightmare. Therefore, to makes this easier for our readers, which could potentially be a future version of ourselves, it is good practice to declare variables as close to their usage as possible:

```go
func main() {
  var sender chan Item
  sender = make(chan Item)
  
  go func() {
   	for {
    	select {
      case item := <- sender:
        // do something
      }
  	} 
  }()
}
```

However, we can do even better than this, by invoking the function directly on declaration. 

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

And coming full circle, we can omit the anonymous function and make it a named function instead:

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

Just like in the previous section, we are now declaring our function and tying it with it's logic, immediately. This makes it easier to traverse code and understand the responsibility of each variable.

Of course, this doesn't actually limit us from mutating our `sender` variable. There is nothing that we can do about this, as there is no way of declaring a `const struct` or `static` variables in Go. This means, that we will have to restrain ourselves from mutating this variable at a later point in the code.

> NOTE: The keyword `const` does exist, but are limited for use on primitive types. 

One way of getting around this, which at least will limit the mutability of a variable to a package level. Is to create a structure, with the variable as a private property. This private property is, thenceforth, only accesible through other methods of this wrapping structure. Expanding on our channel example, this would look something like the following:

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

We have now ensured, that the `sender` property of our `Sender` struct, is never mutated. At least not, from outside of the package. As of writing this document, this is the only way of creating publicly immutable non-primitive variables. It's a little verbose, but it's truly worth the effort, to ensure that we don't end up with strange bugs, that could be the outcome of mutating properties of our structure. 

```go
func main() {
  sender := NewSender()
  sender.Send(&Item{})
}
```

Looking at the exampe above, it's clear how this also simplifies the usage of our package. This way of hiding the implementation, is not only beneficial for the maintainers of the package, but also the users of the package. Now, when initialising and using the `Sender` structure, there is no concern of the implementation. This opens up, for a much looser architecture. Because our users aren't concerned with the implementation, we are free to change it at any point, since we have reduced the point of contact users of the package have. If we no longer wish to use a channel implementation in our package, we can easily change this, without breaking the usage of the `Send` method (as long as we adhere to it's current function signature).

>  NOTE: There is a fantastic explanation of how to handle the abstraction in client libraries, taken from the talk [AWS re:Invent 2017: Embracing Change without Breaking the World (DEV319)](#https://www.youtube.com/watch?v=kJq81Y7OEx4)

## Clean Golang

This section will describe some less generic aspects of writing clean golang code, but rather be discussing aspects that are very go specific. Like the previous section, there will still be a mix of generic and specific concepts being discussed, however, this section marks the start of the document, where the document changes from a generic description of clean code with golang examples, to golang specific descriptions, based on clean code principles.

### Return Values

#### Returning Defined Errors
We will be started out nice an easy, by describing a cleaner way to return errors. Like discussed earlier, our main goals with writing clean code, is to ensure readibility, testability and maintanability of the code base. This error returning method will improve all three aspects, with very little effort.

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

There is nothing inherently smelly about this function, in its isolation. We look into the `items` map of our `Store` struct, to see if we already have an item with this `id`. If we do, we return the item, if we don't, we return an error. Pretty standard. So, what is the issue with returning custom errors like this? Well, let's look at what happens, when we use this function, from another package:

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

This is actually not too bad. However, there is one glaring problem with this. Errors in golang, are simply just an `interface` which implements a function (`Error()`) which returns a string. Therefore, we are now hardcoding the expected error code into our code base. This isn't too great. Mainly, because if  the error message value changes, our code breaks (softly). Our code is too closely coupled, meaning that we would have to change our code in, possibly, many different places. Even worse would be, if a client used our package to write this code. Their software would inexplicably break all of a sudden after a package update, should we choose to change the message of the returnig error. This is quite obviously something that we want to avoid. Fortunately, the fix is very simple. 

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

Returning default `Null` values like the previous examples, can also be more safe, in certain caes. As an example, a user of our package could forget to check for errors and end up initialising a variable, pointing to an empty struct containing a default value of `nil` as one or more property values. When attempting to access this `nil` value later in the code, this could cause a panic in thier code. However, when we return our custom default value instead, we can ensure that all values, which otherwise would default to `nil`, are initialised and thereby ensure that we do not cause panics in our users / clients software. This is also beneficial for ourselves, as if we wanted to achieve the same safety, without returning a default value, we would have to change our code, every place in which we return this type of empty value. However, with our default value appraoch, we now only have to change our code in a single place:

```go
var NullItem = Item{
    itemMap: map[string]Item{},
}
```
> NOTE: In many scenarios, invoking the panic will actually be preferable. To indicate that there is an error check missing.

> NOTE: Every interface property in golang, has a default value of `nil`. This means that this is useful, for any struct, which has an interface property. This is also true for structs which contain channels, maps and slices, which could potentially also have a `nil` value.

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

This new data structure still works as our standard error. We can still compare it to `nil` since it's an interface implementation and we can still call `.Error()` on it, so it won't break any already existing implementations. However, the advantage is that we can now check our error type as we could previously, despite our error now containing the *dynamic* details:

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
            http.Error(w, err.Error(), http.StatusNotFound)
	        return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    } 
    json.NewEncoder(w).Encode(item)
}
```

#### Returning Other Values

// TODO: this section must be refactored. It's awful

This section isn't going dive tremendously into the idea of returning values and how to ensure clean code in doing so. However, it's a topic will act as a nice lead-up to the next section of this article. As mentioned many times before, a big part of the *why* of writing clean code, is to ensure readability. Readability is obviously something that is somewhat subjective, however, despite this the following seems to be indisputable. In order to maximize readability, the code we write, should look similar, if the functionality is similar. This makes it easy to identify the functionality of functions and thereby enabling developers to read / skim code efficiently. 

Take a look at the ending result from [Cleaning Functions](#Cleaning_Functions). These smaller functions all look and behave the same, in turn, making them very easily parseable. 

```go
func getItemByReference(extension string) (string, bool) {
    refIface, ok := db.ReferenceCache.Get(extension)
    if !ok {
        return EmptyItem, false
    }
    return refIface.(string)
}

func getActiveItemByReference(reference string) (Item, bool) {
    item, ok := getItemByReference(reference)
    if !item.Active || !ok {
        return EmptyItem, false
    }
    return item, true
}
```

Now, let's change the code a little and throw a curve ball, for absolutely no reason:

```go
func getItemByReference(extension string) (string, bool) {
    refIface, ok := db.ReferenceCache.Get(extension)
    if !ok {
        return EmptyItem, false
    }
    return refIface.(string)
}

func getActiveItemByReference(reference string) (item Item, ok bool) {
    item, ok = getItemByReference(reference)
    if !item.Active || !ok {
        return
    }
    return
}
```

We haven't changed the code logic, much. The only difference is that we are initialising the variables `item` and `ok`, in the function return signature. The `return` statement is now equivalent to:

```go
return item, ok
```

However, this is not apparent at first glance and the style of the code is not congruent with the rest of our functions / code. We are also making mutable variables, which will definitely open up for possible shadow variable issues further down the line. 

The first time I saw that this was possible to do in golang, I really thought that this was a super neat little trick. However, now I am not particularly excited about the idea of returning values in this way. When returning values, we should make sure that the receiver of this value, is safe when receiving the value. As of now, we are returning a value of type `Item`. What does this default to? Presumably, it is an empty initialised value: `Item{}`. This is not too bad. However, if we are working with something less concrete, such as an `interface` or a `slice`, it is entirely possible that we end up returning `nil` or even worse, return a variable of some weird format.

This is why, as you might have noticed, that in all the code examples, when an error has ocurred an `EmptyItem` has been returned. The essence of this, is to make clear that we are returning an empty variable together with our error. If we omit this, we can get unexpected behaviour, which might end up being very difficult to debug.

The code above, actually presents us with this exact issue. In our original clean function, the if either of the boolean operators `!item.Active || !ok` were true, we would return a "ok = false" response. The invoker of the function could check that the result returned was bad / not ok and could then handle this. However, the dirty function doesn't actually mirror this behaviour. If `item.Active == true`, we are now actually returning "ok = true" and therefore, instead of returning an error-like value, we are returning a inactive item, which wasn't the intention of the function. 

If you noticed that we introduced this bug. Well done! However, as before, please keep in mind that this is very simple code and on a larger scale, these kind of errors could be very difficult to catch. That being said, even on a smaller scale, these errors can be very difficult to catch. 

It is my opinion, that you should avoid using this golang functionality. It's very rarely benificial. That being said, there are certain scenarios where it gives a slight benefit to the developer, it rarely justifies the introduced complexity this change brings with it. 

A very common usage of this golang technique, is together with unmarshalling functions or database calls:

```go
func NewItemFromJSON() (item Item, err error) {
    if err = json.Unmarshal(&item); err != nil {
        return
    }
    return 
}
```

This is of course, instead of using the following logic:

```go
func NewItemFromJSON() (Item, error) {
    var item Item
    if err = json.Unmarshal(&item); err != nil {
        return EmptyItem, err
    }
    return item, nil
}
```

The second version may seem 'uglier', but in fact, it is the clean version. The biggest problem with the first function, is that it is returning the item, which is being mutated by the `json.Unmarshal` function. If an error occurs, we have no idea what the state of this variable is. The fact that we are sending a mutated version of our variable back, could lead to some unexpected results. We want to stay as far away from introducing undefined behaviour as we possibly can and therefore the second option is much preferred.

// TODO: Seems like something is missing here

### Nil Values 
A controversial aspect of Go, is the addition of `nil`. This value corresponds to the value `NULL` in C and is essentially an uninitialised pointer. A lot of other languages have omitted this from their language, prioritising safety and therefore the equivalent of `nil` is in fact a type, rather than an actual null pointer. The reason for this, is that null pointers can cause a lot of trouble. We explained which troubles they can cause in the sections [Returning Defined Errors](#Returning-Defined-Errors) and [Returning Other Value](#Returning-Other-Values), but to some up: Things break, when you try to access null pointers. 

Luckily, there is an easy way to combat this. One method of doing this, is by hiding our concrete struct behind an interface. The concrete type can only be initialised via. functions, which ensure that all possible `nil` values are initialised and we hereby ensure that these values will never be accessed.

```go
type Cache interface {
    Add(string, string) error
}

type kvCache struct {
    cache map[string]string
}

func (c *kvCache) Add(key, value string) error {
    ...
}

func NewKVCache() Cache {
    return &kvCache{
        cache map[string]string{},
    }
}
```

The above code works and fixes the immediate issue, ensuring that the `map` in our key value cache, is initialised with our constructor. Since it's a private struct, then there is no way of initialising it directly outside of our package and we can therefore conclude, that the `cache` map property, will never be accessed as a null poitner. However, using this can be quite tedious, as we will have to create both interface and struct methods for every single property that we wish to access. Don't get me wrong, this is a completely legitimate way of writing code, but in some cases, I don't think that it is the most suitable / straightforward solution to solve this issue. 

Instead, we can turn our `KVCache` into a public struct, making it fully accessible outside of package. Instead of hiding the properties behind an interface, we can make the properties which default to `nil` private and create methods for accessing them. These methods can then ensure that these values are not `nil` before returning them.

```go
type KVCache struct {
    cache map[string]string
}

func (c *KVCache) Cache() map[string]string {
	if c.cache == nil {
		c.cache = map[string]string{}
	}
	return c.cache
}
```

This way, we have still fixed the issue that we started out with, but we are now free to add other non-nil default valued properties to our struct and make them publically accessible. This is nice and saves us the trouble of having to implement a bloated interface, for accessing our various properties. We aren't writing Java, so we want to avoid looking like we enjoy writing getters and setters.

This method works for all values default to `nil`: slices, interfaces, maps, channels etc. and is a good way of prioritising safety, without annoying the users of our package, by denying them access to the different properties of our structures.

### Pointers in Go
Pointers in go are rather a large topic. They are a very big part of working with the language, so much so, that it is essentially impossible to write go, without some knowledge of pointers and their workings in go. Pointers in themselves are a large topic, so I will refrain from explaining them in detail, but rather just explain their quirks and how to handle these quirks in go.

Pointers add complexity, however, as mentioned, it's almost impossible to avoid them when writing go. Therefore, it is important to understand how to use pointers, without adding unecessary complexity and thereby keeping your codebase clean. Without restraining oneself, the incorrect use of pointers can introduce nasty side-effects, introducing bugs that are particularly difficult to debug. Of course, when sticking the basic principles in the first part of this article, we limit our exposure of introducing this complexity, but pointers are a particular case, which can still undo all of our previous hard work to make our code clean. 

#### Pointer Mutability
I have already used the word mutability more than once in this article, as a negative. I will keep doing this, in hope that it become indoctrinating in your mind. Mutability is obviously not a clear-cut bad thing and I am by no means an advocate for writing 100% pure functional programs. Mutability is a powerful tool, but we should really only ever use it, when it's necessary. Let's have a look at a code example illustrating why:

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

At first glance, this doesn't seem too bad. In fact, it might even seem like a rather simple insert function for a common list structure. We accpet a pointer as input and if no other users with this `id` exist, then we insert the user pointer into our list. Now, we use this functionality in our public API for creating new users:

```go
func CreateUser(w http.ResponseWriter, r *http.Request) {
    user, err := parseUserFromRequest(r)
    if err != nil {
        http.Error(w, err, http.StatusBadRequest)
        return
    }
    if err := store.Insert(user); err != nil {
        http.Error(w, err, http.StatusInternalServerError)
        return
    }
    user.Password = ""
    if err := json.NewEncoder(w).Encode(user); err != nil {
        http.Error(w, err, http.StatusInternalServerError)
    }
}
```

Once again, at first glance everything looks fine. We parse the user from the received request and insert the user struct into our store. Once we have inserted our user into the store successfully, we then set the password to nothing, before returning the user as a json object to our client. This is all quite common practice, typically when returning a user object, where the password has been hashed, we don't want to return the hashed password.

However, this code is horribly flawed. If we check in our user store, see that the change we made to the users password in the http handler function, also affected the object in our store. This is because the pointer address returned by `parseUserFromRequest`, is what we populated our store with, rather than an actual value. Therefore, when making changes to the dereferenced password value, we end up changing the value of the object we are pointing to in our store.

This is a great example of why both mutability and variable scope, can cause some serious issues and bugs, when used incorrectly. The fix for this particular bug, is rather simple:

```go
func (store *UserStore) Insert(user User) error {
    if store.userExists(user.ID) {
        return ErrItemAlreaydExists
    }
    store.users[user.ID] = &user
    return nil
}
```

Instead of passing a pointer to a `User` struct, we are now passing in a copy of a `User`. We are still storing a pointer to our store, however, instead of storing the pointer from outside of the function, we are storing the pointer to the copied value, which scope is inside the function. This fixes the immediate problem, but might still cause issues further down the line, if we aren't careful.

```go
func (store *UserStore) Get(id int64) (*User, error) {
    user, ok := store.users[id]
    if !ok {
        return EmptyUser, ErrUserNotFound
    }
    return store.users[id], nil
}
```

Again, a very standard very simple implementation of a getter function for our store. However, this is still bad. We are once again expanding the scope of our pointer, which may end up causing unexpected side-effects. When returning the actual pointer value, which we are storing in our user store, we are essentially giving other parts of our application the ability to change our store values. This is bad, because it's bound to ensure confusion. Our store should be the only entity enabled to make changes to the values stored there. The easiest fix available for this, is to either return a value of `User` rather than returning a pointer. 

Please keep in mind, that there is intrinsically nothing wrong with returning pointers, however, the expanded scope and number of owners of the variables is the important aspect, which makes the previous example a smelly operation. As an example, there is nothing wrong with the following example:

```go
func AddName(user *User, name string) {
    user.Name = name
}
```

The reason why this is *ok*, is that the scope, which is defined by whomever invokes the functions, remains the same after the function returns. This combined with the fact, that the ownership is also handled in a more straightforward manner. This functions will have an ownership something like:

```
ParentFn -> AddName (adds name) -> ParentFn
```

In other words the ownership is (logically) moved to the `AddName` function as is returned to whichever parent function, when the function returns. In contrast, our store returning pointers looked more like this:

```
HttpHandler -> store.Insert ->  HttpHandler -> 
                                store.users ->
ParentFn    -> store.Get    ->  ParentFn    ->
```

Rather than having a linear ownership hand over, we have a tree of ownership instead. With many different parts of our applicaton, that can mutate our variable in a potentially global scope.

### Using `goto` in Go
Just don't

// TODO : Despite wonderful comedic effect, this should probably be elaborated upon

### Closures are Function Pointers

So, before we go to the next topic of using interfaces in Go. I would like to introduce the commonly overseen alternative, which is what C programmers know as 'function pointers' and most other programmers refer to as 'closures'. Closure are quite simple. They are an input parameter for a function, which act like any other parameter, except for the fact that they are a function. In Javascript, it is very common to use closures as callbacks, which is typically used in scenarios where upon we want to invoke a function after an asynchronous operation has finished. In Go, we don't really have this issue, or at the very least, we have other, much nicer, ways of solving this issue. Instead, in Golang, we can use closures to solve a different hurdle: The lack of generics. 

Now, don't get too excited. We aren't going to substitute the lack of generics. We are simply going to solve a subset of the lack of generics with the use of closures. Consider the following function signature:

```go
func something(closure func(float64) float64) float64 { ... }
```

This function takes in a function, which returns a float64 and then returns a float64. This pattern can be particularly useful, for creating a loosely coupled architecture, making it easier to to add functionality, without affecting other parts of the code. An example use case of this, could be that we had a structure, containing some data. Through this structures `Do()` method, we can perform operations on this data. If we know the operation ahead of time, we can approach problem this by placing the logic for handling the different operations, directly in our `Do()` method:

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

As we can imagine, this function will perform a predetermined operation on the data contained in the `Datastore` struct. However, we can also imagine, that at some point we would want to add more operations. Over a longer period of time, this might end up being quite a lot of different operations, making  our `Do` method bloated and possibly even hard to maintain. It might also be an issue for people wanting to use our `Datastore` object, who don't have access to edit our package code. Keeping in mind, that there are no way of extending structure methods as there is in most OOP languages, this could also become an issue for developers wanting to use our package. 

So instead, let's try a different approach, using closures instead:

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

However, other than this being a very messy function signature, we also have another issue with this. This function isn't particularly generic. What happens, if we find out that we actually want the `concat` function needs to be able to take multiple byte arrays as input? Or if want to add some completely new functionality, that may also need more or less input values than `(data []byte, data []byte)` ?

One way to solve this issue, is to change our concat function. In the example below, I have changed it to only take a single byte array as input argument, but it could just as well have been the opposite case. 

```go
func concat(data []byte) func(data []byte) func(data []byte) ([]byte, error) {
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

Notice how we have added some of the clutter from the `Do` method signature. The way that we have accomplished this, is by having our `concat` function return a function. Within the returned function, we are storing the input values originally passed in to our `concat` function. The returned function can therefore now take a single input parameter, and within our function logic, we will append it, with our original input value. As a newly introduced concept, this is quite strange, however, getting used to having this as an option can indeed help loosen up program coupling and help get rid of bloated functions. 

In the next section, we will talk about interfaces, but let's take a short moment to talk about the difference between interfaces and closures. The problems that interfaces solve, definitely  overlap with the problems solved by closures.  The implementation of interfaces in Go makes the distinction of when to use one or the other, somewhat difficult at times. Usually, whether an interface or a closure is used, is not really of importance and whichever solves the problem in the simplest manner, is the right choice. Typically, closures will be simpler to implement, if the operation is simple by nature. However, as soon as the logic contained within a closure becomes complex, one should strongly consider using an interface instead.  

// TODO : Still not particularly happy the ending of this chapter, still seems unfinished

Dave Cheney has an excellent write up on this topic, and a talk on the same topic:

* https://dave.cheney.net/2016/11/13/do-not-fear-first-class-functions
* https://www.youtube.com/watch?v=5buaPyJ0XeQ&t=9s

Jon Bodner also has a talk about this topic

* https://www.youtube.com/watch?v=5IKcPMJXkKs

### Interfaces in Go

In general, the go method for handling `interface`'s is quite different from other languages. Interfaces aren't explicitly implemented, like they would be in Java or C#, but are implicitly implemented if they fulfill the contract of the interface. As an example, this means that any `struct` which has an `Error()` method, implements / fullfills the `Error` interface and can be returned as an `error`. This has it's advantages, as it makes golang feel more fast-paced and dynamic, as implementing an interface is extremely easy. There are obviously also disadvantages with this approach to implementing interfaces. As the interface implementation is no longer explicit, it can difficult to see which interfaces are implemented by a struct. Therefore, the most common way of defining interfaces, is by writing interfaces with as few methods a possible. This way, it will be easier to understand whether or not a struct fulfills the contract of an interface.

There are other ways of keeping track of whether your structs are fulfilling the interface contract. One method, is to create constructors, which return an interface, rather than the concrete type:

```go
// package io 
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

The above function ensures, that the `NullWriter` struct implements the `Writer` interface. If we were to delete the `Write` method for the `NullWriter` we would get a compile error, where we to try and build the solution. This is a good way of ensuring our code behaves in the way that we expect and we can use the compiler as a safety net to ensure that we aren't producing invalid code. 

There is another way of trying to be more explicit about which interfaces a given struct implements. However, this method is, achieves the opposite of what we wish to achieve. The method being, using embedded interfaces, as a struct property.

<center style="font-style: italic">"Wait what?" - Presumably most people</center>

So, let's rewind a little, before we dive deep into the forbidden forest of smelly Go. In Go, we can use embedded structs, as a type of inheritance in our struct definitions. This is really nice as we can decouple our code, by defining reusable structs.

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

Above, we are defining a `Metadata` object, which will provide us with property fields that we are likely to use on many different types. This is great and by all means great for improving our code cleanliness. This is mainly, because we can use this, decouple our code and thereby update our Metadata object, without breaking other parts of our code. An example of this, let's create some constructors: 

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

At a later point in time, we find out, that we would also like a `CreatedAt` field on our `Metadata` object. This is now easily achieveable, by simply updating our `NewMetadata` constructor:

```go
func NewMetadata(user types.User) Metadata {
    return &Metadata{
        CreatedBy: user,
        CreatedAt: time.Now(),
    }
}
```

Now, both our `Document` and `AudioFile` structures are updated, to also populate these fields on construction. This is the core principle behind decoupling and an excellent example of ensuring maintanability of code. We can also add new methods, without breaking our code:

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

Again, without breaking the rest of our code base, we are implementing new functionality to our already existing structures. This kind of programming, makes implementing new features very quick and very painless, which is exactly what we are trying to achieve by making our code clean. 

Now, I am sorry to break this streak of happiness, because now we return to the smelly forbidden forest of Go. Let's get back to our interfaces and how to show explicitly which interfaces are being implemented by a structure. Instead of embedding a struct, we can embed an interface:

```go
type NullWriter struct {
    Writer
}

func NewNullWriter() io.Writer {
    return &NullWriter{}
}
```
The above code compiles. This is bad. Technically, we are implemnting the interface of `Writer`, because we are embedding the interface and "inheriting" the functions which are associated with this interface. Some see this as a clear way of showing that our `NullWriter` is implementing the `Writer` interface. However, we have to be careful using this technique, as we can no longer rely on the compiler to save us:

```go
func main() {
    w := NewNullWriter()

    w.Write([]byte{1, 2, 3})
}
```

As mentioned before, the above code will compile. The `NewNullWriter` returns a `Writer` and everything is honky-dori, according to the compiler, because `NullWriter` fulfills the contract of `io.Writer`, via. the embedded interface. However, running the code above will result in the following:

> panic: runtime error: invalid memory address or nil pointer dereference

The explanation being, that an interface method in Go, is essentially a function pointer. In this case, since we are pointing the function of an interface, rather than an actual method implementation, we are trying to invoke a function, which is in actuality a nil pointer. Oops! Personally, I think that this is a massive oversight in the Go compiler. This code **should not** compile... but while this is being fixed (if it ever will be), let's just promise each other, never to implement code in this way. In an attempt to be more clear with our implementation, we have ended up shooting ourselves in the foot and bypassing compiler checks. 

> NOTE: Some people argue that using embedded interfaces, is a good way of creating a mock structure, for testing a subset of interface methods. Essentially, by using an embedded interface, you won't have to implement all of the methods of an interface, but instead only implement the few methods that you wish to be tested. In my opinion, this is just laziness combined with congnitive dissonance. 

Let's quickly get back to clean code and quickly get back to using interfaces the proper way in Go. Let's talk about using interfaces as function parameters and return values. The most common proverb for interface usage with functions in Go is:

<center style="font-style: italic; margin: 0 100px 0 100px">"Be consdervative in what you do, be liberal in what you accept from others" - Jon Postel</center>

> FUN FACT: This proverb originally has nothing to do with Go, but is actually taken from an early specification of the TCP networking protocol.

In other words, you should write functions that accept an interface and return a concrete type. This is generally good practice, and becomes super beneficial when doing tests with mocking. As an example, we can create a function which takes a writer interface as input and invokes the `Write` method of that inteface.

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

func (w *NullWriter) Write(data []byte) (int, error) {
    return len(data), nil
}

func TestFn(t *testing.T) {
    ...
    pipe := NewPipe(NullWriter{})
    ...
}
```

> NOTE: there is actually already a null writer implementation built into the ioutil package named `Discard`

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

In this case, we aren't only accepting a single `interface{}` but rather, a slice of types. These types can be of any type and can even be of different types, as long as they implement the empty `interface{}`, which funnily enough, they 100% will. This is a super common pattern when handling string conversation (both from and to string). The reason being, this is the only way in golang to implement generic methods. Good examples of this, come from the `json` standard library package:

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

The reason why this is a good way of using the empty `interface{}`. We are containing all the *messy* code inside our `Decode()` function, so that the developer using this functionality won't have to worry about reflection or casting of types. We just have to worry about providing a pointer to a concrete type. This is good, because the `Decode()` function is, essentially, returning a concrete type. However, even when using the empty `interface{}` with good practices, we still have some issues. If we pass a `JSON` string that has nothing to do with our `Item` type, we still won't receive an error. Out `item` variable will just be left with the default values. So, while we don't have to worry about reflection and casting errors, we will still have to make sure that the message sent from our client is a valid `Item` type. However, as of writing this document, there is no simple / good way to implement these type of generic decoders, without using the empty `interface{}` type. 

The problem with this, is that we are leaning towards using golang (a statically typed languagae) as a dynamically typed langauge. This becomes even clearer, when looking at poor implementations of the `interface{}` type. The most common example of this, comes from developers trying to implement a generic store / list of some sort. Let's look at an example, trying to implement a generic HashMap package, which can store any type, using the `interface{}`. 

```go
type HashMap struct {
    store map[string]interface{}
}

func (hashmap *HashMap) Insert(key string, value interface{}) {
    hashmap.store[key] = value
}

func (hashmap *HashMap) Get(id string) (interface{}, error) {
    value, ok := hashmap.store[key]
    if !ok {
        return nil, ErrKeyNotFoundInHashMap
    }
    return value
}
```

> NOTE: I have omitted thread-safety from the example for simplicity

Please keep in mind that the implementation pattern used above, is used in quite a lot of golang packages. It is even used in the standard library `sync` package, for the `sync.Map` type. So, what is the big problem with this implementation? Well, let's have a look at an example of using this package.

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

On first glance, this looks fine. However, like mentioned previously. What happens when the code base changes? As of right now, there is nothing limiting us from adding something other than the `Item` type. So what happens when someone starts adding other types into our HashMap? Our function now might return an error. This might even be a small change like someone else in the code base wanting to store a pointer `*Item` instead of an `Item`. Worst of all, this won't even be caught by our tests. We are simply checking that our function is working as expected and it does. The developer inserting the `*Item`, should of course add tests that will fail, but depending on the complexity of the system, might actually have a difficult time debugging why this test is failing. 

This type of code, should never reach production. The matter of the fact is, that golang does not support generics as of now and as golang programmers, we should accept this. If we want to use generics, then we should use a different language which does support generics, rather than trying hack our way out of this. This is my biggest gripe with golang. A language that is otherwise quite good at forcing their idioms on their users, it baffles me that the use of `interface{}` is not more restricted. If anything, I feel that the return of an `interface{}` type should flag for a warning. I actually believe that it's that bad of a practice. If anything, it's much worse practice than using snake case instead of came case. 

So, how do we prevent this code from reaching production? Well, let's borrow a principle from the Rust language. Generics are fully supported by Rust, which is actually quite strange, considering the language architecture. However, upon further insight, the way that Rust handles this, is very simple and very straight forward. Rust does not do runtime reflection, instead Rust does compiletime reflection. On compile, Rust will map out all the different possible variations of a generic function and compile a variant of this function, using concrete types rather than generic types. Referenced from: [this article](https://gist.github.com/Kimundi/8391398).

So, in other words rust is generating our functions for us. If we stick to golang code, that would be the equivalent of having, the following code:

```go
func SomeFunction(value interface{}) interface{} { ... }

func main() {
    SomeFunction(42)
    SomeFunction("hello there")
}
```

Which would generate the following code: 

```go
func SomeFunction_Int(value int) int { ... }
func SomeFunction_String(value string) string { ... }
```

Unfortunately, there is no way in golang to directly translate this. However, we can actually get very very close to producing something similar with go code generation. 

### Go Code Generation
Let me start out by saying, that go code generation is nothing fancy. It really, really isn't. At it's most basic, it's simply just templatling (much like HandleBars.js in JavaScript and Jinja in Python). However, it works really well, especially when it comes to writing generic code, that we don't want to write over and over again. Keep in mind, that it doesn't solve all of our problems and there are definitely situations in which actual generic lanauge support would be preferred. However, it is still greatly preferred to using empty `interface{}` 

The following shows some basic go code generation, which will be able to create very simple, but thread-safe, HashMap for primitive types. This is an example of how to do this, and should give enough inspiration as how to expand on the example and create more elaborate code generation templates.

Our project has the following structure:

```
.
|-- generate
|   `-- hashmap.go
|-- main.go
```

The `hashmap.go` file consists of a hashmap template and some logic, which will iterate over command line arguments to generate different types specified by the arguments. From the root of our project, we can run the following command:

> go run generate/hashmap.go int64 string uint16

This will generate files which implement a `HashMap` type for `int64` `string` and `uint16`. So the project now looks like the following:

```
.
|-- generate
|   `-- main.go
|-- hashmap
|   |-- hashmap_int64.go
|   |-- hashmap_string.go
|   `-- hashmap_uint16.go
|-- main.go
```

> NOTE: The full code can be found in the index of this document

We can now use these hashmap types in the rest of our codebase. These HashMaps are now no longer using `interface{}` but instead specifying concrete types for both input and output. As a short example, our get function now looks like this: 

```go
func (hashmap *HashMap_int64) get(key string) (int64, error) {
	value, ok := hashmap.store[key]
	if !ok {
		return Emptyint64, Errint64NotFoundInHashMap
	}
	return value
}
```

If we think about what was mentioned in the previous chapter, we can see why this is advantage for us. We are no longer having to do any type casting and we can be 100% sure that our functions are returning the correct concrete type. We can also be sure, that if breaking changes are introduced by changes the generated code, we will get compile time or test errors immediately. Not only does this make it easier to maintain our codebase, but it's also easier to understand the codebase and the effects of changing the codebase might have.

Of course, this doesn't come without certain disadvantages. We add a comment a top of every generated file:

> // Code generated by hashmap.go -Type int64 DO NOT EDIT.

This line, will tell certain IDE's to show a warning, that the file is not to be edited. However, nothing is actually preventing users from editing these files. Typically though, code generation is used along with a build script tool (such as a Makefile), so that all generated files are re-generated on build. This will, hopefully, prevent any users from making unwanted changes to the generated code. Of course, this also means that there is no room for tweaking our generated code type by type, but I feel this to be more of an advantage, than anything.

## Index

### Generated Code
#### generate/hashmap.go
```go
package main

import (
	"bytes"
	"io/ioutil"
	"os"
	"path"
	"strings"
	"text/template"
)

type genValues struct {
	Type string
}

func main() {
	if err := clean(); err != nil {
		panic(err)
	}

	if err := generate(os.Args[1:]); err != nil {
		panic(err)
	}
}

func clean() error {
	if err := os.RemoveAll("hashmap"); err != nil {
		if !os.IsNotExist(err) {
			return err
		}
	}
	return os.Mkdir("hashmap", 0755)
}

func generate(args []string) error {
	tmpl, err := template.New("hashmap").Parse(hashmap)
	if err != nil {
		return err
	}
	for _, arg := range args {
		if err := generateHashMap(arg, tmpl); err != nil {
			return err
		}
	}
	return nil
}

func generateHashMap(arg string, tmpl *template.Template) error {
	buf := &bytes.Buffer{}
	if err := tmpl.Execute(buf, genValues{Type: arg}); err != nil {
		return err
	}
	return ioutil.WriteFile(createFilename(arg), buf.Bytes(), 0755)
}

func createFilename(arg string) string {
	return path.Join(
		"hashmap",
		"hashmap_"+strings.ToLower(arg)+".go",
	)
}

var hashmap = `// Code generated by hashmap.go -Type {{.Type}} DO NOT EDIT.

package hashmap

import (
	"errors"
	"sync"
)

var (
	Empty{{.Type}} = {{.Type}}

	Err{{.Type}}NotFoundInHashMap = errors.New("{{.Type}} was not found in hash map")
)

type HashMap_{{.Type}} struct {
	store map[string]{{.Type}}
	mtx sync.RWMutex
}

func NewHashMap_{{.Type}}() *HashMap_{{.Type}} {
	return &HashMap_{{.Type}}{
		store: map[string]{{.Type}}{},
	}
}

func (hashmap *HashMap_{{.Type}}) Get(key string) ({{.Type}}, error) {
	hashmap.mtx.RLock()
	defer hashmap.mtx.RUnlock()

	return hashmap.get(key)
}

func (hashmap *HashMap_{{.Type}}) get(key string) ({{.Type}}, error) {
	value, ok := hashmap.store[key]
	if !ok {
		return Empty{{.Type}}, Err{{.Type}}NotFoundInHashMap
	}
	return value
}

func (hashmap *HashMap_{{.Type}}) Insert(key string, value {{.Type}}) {
	hashmap.mtx.Lock()
	defer hashmap.mtx.Unlock()

	hashmap.store[key] = value
}

func (hashmap *HashMap_{{.Type}}) Delet(key string) {
	hashmap.mtx.Lock()
	defer hashmap.mtx.Unlock()

	delete(key, hashmap.store)
}
`
```

#### hashmap/hashmap_int64.go
```go
// Code generated by hashmap.go -Type int64 DO NOT EDIT.

package hashmap

import (
	"errors"
	"sync"
)

var (
	Emptyint64 = int64

	Errint64NotFoundInHashMap = errors.New("int64 was not found in hash map")
)

type HashMap_int64 struct {
	store map[string]int64
	mtx sync.RWMutex
}

func NewHashMap_int64() *HashMap_int64 {
	return &HashMap_int64{
		store: map[string]int64{},
	}
}

func (hashmap *HashMap_int64) Get(key string) (int64, error) {
	hashmap.mtx.RLock()
	defer hashmap.mtx.RUnlock()

	return hashmap.get(key)
}

func (hashmap *HashMap_int64) get(key string) (int64, error) {
	value, ok := hashmap.store[key]
	if !ok {
		return Emptyint64, Errint64NotFoundInHashMap
	}
	return value
}

func (hashmap *HashMap_int64) Insert(key string, value int64) {
	hashmap.mtx.Lock()
	defer hashmap.mtx.Unlock()

	hashmap.store[key] = value
}

func (hashmap *HashMap_int64) Delet(key string) {
	hashmap.mtx.Lock()
	defer hashmap.mtx.Unlock()

	delete(key, hashmap.store)
}
```

