# About Zaria.AI 

This framework provides a simple, attribute-driven, low-code API for building text based interactive dialogs. It Can be used to build AI BOTs, 
CLIs, general command processing and much more! 

___
## Updates from Alpha/Beta

> Updated nuget reference from alpha/beta naming to final product name **Zaria.AI**.  If you used Thinkmine.Zaria.Commander please note that namespaces and type 
names have changed in the final release. 

> Types that can be used as parameters in a handler method have been locked down to:
  - int
  - string
  - bool
  - guid
  - double
  - CommandContext

    Any other type used as a parameter in a handler method will cause that method to be ignored.

> Reference variables directly in input commands. To reference a variable use the ```$``` sumbol before it.
So if there was a variable set in code ```context.Pipeline.Store("age", 25);``` then it could be referenced when the user is
inputing commands as follows ```Set age to $age```.  This value will be replaced by the value of the variable (25) so that the
resulting input string will be ```Set age to 25```.  It is important to note that this is what will be processed, not the original string.

> Changed ```CommandContext.Pipeline.Add``` to ```CommandContext.Pipeline.Set```

> Changed ```CommandContext.Pipeline.Get``` to return a tuple ```(bool Exists, object Value)``` where the boolean value indicates
whether there was a key found in the pipeline state.  This change was made to remove ambiguity as to the meaning of a default value such as 
*null* getting returned (since a default value can mean both that the variable does not exist **or** that the value *does* exist but is null).  Using this
approach retrieving state can be accomplished as follows:

```
[Pattern("display @var")]
bool Display(string var, CommandContext context)
{
    var value1 = context.Pipeline.Get<int>(var);
    if (value1.Exists)
        Console.WriteLine($"Your answer is {value1.Value}");
    else
        Console.WriteLine("Variable does not exist");
    return true;
}
```

> Added ```File``` property to *Pattern*.  You can specify a fully qualified path to a .pidgin file.  This is simply a
text file with command seperated into individual lines.  This approach supports all the same capabilities as all other approaches
to declaring patterns however this does not work with injected patterns.  The sample below creates a handler that gets its patterns from a file called arithmetic-commands.pidgin

```
[Pattern(File = @"C:\CommandTestHarness\arithmetic-commands.pidgin")]
bool ProcessPatternsFromFile(int x, int y)
{
    int answer = 0;
    string operand = "";
    switch (_context.InputTokens[0].Text)
    {
        case "add":
            operand = "+";
            answer = x + y;
            break;
        case "subtract":
            operand = "-";
            answer = y - x;
            break;
    }
    Console.WriteLine($"{x} {operand} {y} = {answer}");
    return true;
}
```

> Added ```URL``` property to *Pattern*.  You can specify a url to an endpoint that returns a string with command seperated into individual lines.  
The command processor will use an HTTP Get to attempt to retrieve the pattern data.  This approach supports all the same capabilities as all other approaches to 
declaring patterns however this does not work with injected patterns.  The following sample gets its patterns from a uri on the cloud. The endpoint must return the 
patterns as plain text with one pattern on each line.

```
[Pattern(URL = @"http://<some service>/api/Function1")]
bool ProcessPatternsFromURL(int x, int y)
{
    int answer = 0;
    string operand = "";
    switch (_context.InputTokens[0].Text)
    {
        case "multiply":
            operand = "*";
            answer = x * y;
            break;
        case "divide":
            operand = "/";
            answer = x / y;
            break;
    }
    Console.WriteLine($"{x} {operand} {y} = {answer}");
    return true;
}
```


> Added *Provider* property to Pattern.  Any public type that derives from *IPatternProvider* can be used as a value for this property.  The 
type exposes a function which can be used to retrive or generate patterns that will be associated with the given hander declaring the attribute.

```
public class TestPatternProvider : IPatternProvider
{
    public List<string> GetPatterns()
    {
        //go to database and retrieve value
        return new List<string>
        {
            "sample1",
            "sample2",
        };
    }
}
```

The provider is declared as follows:

```
[Pattern(Provider = typeof(TestPatternProvider))]
bool Test(CommandContext context)
{
    Console.WriteLine(context.InputTokens[0].Text);
    return true;
}
```

> Added Pipeline state to process. Pipeline state allows the user to store state to the context such that it can be used by future commands down the pipe.
This is for scenarios where multiple commands need to be processed in order to complete a given task (and the state needs to be passed between the command
handlers. The command processor provides the state but all management must be handled by consumers of this library.  This state **does not** persist beyond
a given instance of the command processor.  The following sample illustrates:

```
[Pattern("var @name =  @value")]
bool SetVariable(string name, int value, CommandContext context)
{
    context.Pipeline.Store(name, value);
    return true;
}
```

This can be used in a subsequent example as follows:

```
[Pattern("display @var")]
bool Display(string var, CommandContext context)
{
    var value1 = context.Pipeline.Get<int>(var);
    Console.WriteLine($"Your answer is {value1}");
    return true;
}
```

This command handler basically prints the given pipeline variable to the console.  

You can build a very simple calculator
by combining these two command patterns with another for the actual arithmetic work as shown below.

```
[Pattern("calc @name1 @operand @name2 into @var")]
bool Calculate(string name1, string name2, string operand, string var, CommandContext context)
{
    var x = context.Pipeline.Get<int>(name1);
    var y = context.Pipeline.Get<int>(name2);
    int answer = 0;
            

    switch (operand)
    {
        case "+":                    
            answer = x + y;
            break;
        case "-":
            answer = x - y;
            break;
        case "*":
            answer = x * y;
            break;
        case "/":
            answer = x / y;
            break;
    }

    context.Pipeline.Add(var, answer);
    return true;
}
```

> Quotes are now processed verbatim so case is maintained for them.  Everything else is case insensitive

> Both versions of Execute now return ```bool```.  This is done to help with the transition from single use to CLI.  If args are passed
the default behavior is to return false (as the expectation is that this is a single run).  It is up to the user and designer of the command 
interface to decide.
___

## Usage

### Creating patterns
For any given class - 
1. For each command pattern you want to process add *static* method with a *bool* return type.  Each of these will serve as a handler for when the user's input matches the pattern this method will advertise.
2. Now decorate the method with **PatternAttribute** attribute. 
3. As a catch-all for any input that does not match a pattern you advertise you can optionally provide another method (with a similar signature as described above) and decorate this one with **NotFoundAttribute** instead. Note that only 1 method, the first encountered with the proper attribute, will be used for the fallback route even if you decorate multiple methods with it.  

**PatternAttribute** accepts a string as an argument. This string represents the pattern that must be matched for the method to be called.  This string can contain any sequence of characters except those with special meaning listed below:

char | Description
--- | ---
  **@**   | Used to define placeholders in the pattern.  When a user types in thier input, the parts that match the criteria for the placeholder will be passed into the method hosting the attribute. To add placeholders to the pattern use the *@* symbol in front of the placeholder name.
  **[**   | Both square brackets are used to denote a section of your pattern that the user can place anywhere in the input string
  **]**   | Both square brackets are used to denote a section of your pattern that the user can place anywhere in the input string
  **?**   | Indicates that the given text is optional.  It is meant to be placed at the end of the item being matched (so the pattern ```hello world?``` will trigger the underlying method if the user passes in *hello* or *hello world*.  Placeholders cannot be optional. If a placeholder has the *?* character after it, the request will be ignored or will generate an error.

Some points to note:
---
The parsing system does a token (word) for token match on the input string using the pattern defined in the PatternAttribute as a psuedo-schema.  By default, word positions are respected when matching in this way.
This approach is somewhat rigid from a users perspective however; forcing them to enter commands in an exact order can be challenging. To address this, Pidgin also includes a mechanism to that allows for sections of the 
overall pattern to be entered without respect to position. While position is still maintained within the section, the resulting sections as a whole can be keyed-in in any order.

- To indicate that a section of the pattern is not tied to a position, encase that section in square brackets (*[]*).  
- Note that you cannot start a pattern with square brackets (the first part of your pattern maintain its position).
- Note that you cannot start any part of your pattern with placeholders.  Additionally, in all scenarios placeholders must come **after** static text. This is also true for all sections of the pattern; meaning if you elect to break the pattern up into sections then those sections will also need text before placeholders.  
- You cannot put sections inside of sections. This means you can't place square brackets inside of a block enclosed by square brackets.  **This will cause a runtime error**.  
- The parsing process is sensitive to white spaces, if the user's intent is to pass in a parameter with white spaces they must use **double quotes** (*"*) to surround that text.  This means a pattern like ```hello @message``` can be
triggered with the text ```hello john``` but can also be triggered with ```hello "john the baptist"```
- Be careful where you put placeholders in the pattern. A good rule of thumb is to have static text between your variables and any *]* character.

---


A pattern like ```The [quick @fox_color fox jumped] @jump_direction over the lazy dog``` will produce three sections.
- The
- quick @fox_color fox jumped
- __@jump_direction over the lazy dog__


A pattern like ```the [@quick @fox_color fox jumped] jump_direction over the lazy dog``` will also produce three sections.
- The
- __@quick @fox_color fox jumped__
- jump_direction over the lazy dog
---

As can be seen, during pattern evaluation sections of the Pidgin may get resolve as starting with a variable.  To prevent this, **never** put a placeholder directly after a partitioned block.  


## Types

### CommandProcessor
Provides functions that can be used to initiaize the processor and execute commands on it.  Also contains
properties that can be used to inspect the input string/array that is being executed

### PipelineContext
Represents the context of the overarching process flow driving the commands entered by the user.  All state management is controlled through this type.

### PatternAttribute
Use this attribute to decorate any methods you want to be used to process a command pattern.

### NotFound
Use this attribute to decorate the method you want to be called when there is no command pattern found

### InputToken
Represents a token (word) in the input string/array that is passed into the processor.

### PatternRepository
Collection where dynamic commands are stored.  Dynamic commands are the commands that are injected into the CommandLineProcessor as opposed to getting
discovered through the initialization process.

### CommandContext
Context of an executing command.  This class can be passed into any command handler (including in NotFound scenarios).
Provides a view into the parameters that were passed in.  You can use the type to walk back and forth through
the text that was passed into your tool token (word) by token.  Note that quoted text is treated as one token

### CommandPatternException
Represents an exception that occurs while parsing the input text

### IPatternProvider
Classes that implement this interface can be used to provide patterns to a given *PatternAttribute*. The following illustrates 
a sample of this:


```
public class TestPatternProvider : IPatternProvider
{
    public List<string> GetPatterns()
    {
        //go to database and retrieve value
        return new List<string>
        {
            "sample1",
            "sample2",
        };
    }
}
```

The provider is declared as follows:

```
[Pattern(Provider = typeof(TestPatternProvider))]
bool Test(CommandContext context)
{
    Console.WriteLine(context.InputTokens[0].Text);
    return true;
}
```

Examples
---

In the following example a command pattern handler is created
```
[Pattern("say [-what @something] [-to @someone]")]
static bool SaySomething(string something, string someone, int i)
{
    Console.WriteLine($"You said {something} to {someone}");
    return true;
}
 ```
This method will be invoked whenever the following pattern is passed to the interpreter ```say -what [...] -to [...] ```.
Any placeholders in the pattern must match the parameter names on the method that it decorates (similar to Web API).  
If placeholders are provided but the method does not have paramters (or they dont match) then the parameters will hold their default
value when the method is run.  This is also true in the reverse scenario; if no placeholders are included the method will be 
called with the default values of any parameters that exist. The return value is not used by the processor; rather, it is meant to be used
in REPL scenarios to indicate whether the loop should continue.  THe convension is to return *true* to continue the program and *false* to exit it.

In the following example we pass the following into a command-line that has the code above deployed:  

```say -what "hello world" -to "the man next door" ``` 

*Note: Quotes are required when spaces exist between your placeholder tokens (if not displayed in your browser)*

The result of this would be: 

**You said hello world to the man next door**


A single method may be decorated with more than one command pattern (thus allowing it to handle multiple scenarios).
In the following example the same *SayHello* method is used to handle three different patterns.  
```
[Pattern("test @arg")]
[Pattern("test")]
[Pattern("hello there")]
static bool SayHello(string arg)
{
    Console.WriteLine($"Calling hello there command with {arg}");
    return true;
}
```

This method will be invoked if any of the following patterns is passed into the processor.
- *test*
- *test 123*
- *hello there*

You can also represent these different patterns on a given handler by using 1 Pattern attribute with multiple patterns as follows

```
[Pattern("test","test @arg","hello there")]
static bool SayHello(string arg)
{
    Console.WriteLine($"Calling hello there command with {arg}");
    return true;
}
```

### Running the Processor

To initialize the processor use the following code:

```
CommandProcessor.Initialize();
```

This will traverse the current assembly and any referenced assemblies and identify all the command patterns that have been
specified.  A command pattern is declared by creating a method with a *boolean return type* and decorating it with the
**PatternAttribute** attribute.  Initialize must always be called first, and must be called again each time a new assembly is added to the 
AppDomain it was initially called in. To limit the commands patterns used to a smaller scope you can use *Initialize* with a type parameter
representing the class who's methods will be inspected for command patterns.  In the following case we are limiting the call to only
initialize methods in the *SamplePatternHost* class.

```
CommandProcessor.Initialize<SamplePatternHost>();
```


The processor can be used as follows:

```CommandProcessor.Execute(string[]);```

This version of the execute method accepts an array of strings.  This can be the arguments passed into the console itself (but it really could be anything).
No special processing is performed on the array other than turning it into a space-delimited string.  So if the array:
```[hello,world,i,like,to,code]``` it would be converted to **```hello world i like to code```**.  This version of the method returns void.

For creating something like a REPL (Read-evaluate-print loop), or for use in generic command processing, the other version of the method is more ideal.

```CommandProcessor.Execute(string);```

This version of the method takes the entire string as input and processes it with no additional manipulation.  This version also returns a bool that can be used
to determine whether it should be called again.

The following shows how a simple console REPL might look.

```
static void Main(string[] args)
{
    Console.WriteLine("Pidgin REPL Sample");

    //call initialization
    var processor = CommandProcessor.Initialize();

    while (true)
    {
        Console.Write("> ");
        var command_line = Console.ReadLine();
        var continue_to_run = processor.Execute(command_line);

        if (!continue_to_run)
            break;
    }

}
```

As can be seen from the sample, the Execute method returns a boolean value that is then used
to determine whether the loop continues to run.  The alternate method does not have a return type.

```
static void Main(string[] args)
{
    var input_pattern = //pattern from args array...
    var processor = CommandProcessor.Initialize();
    processor.Execute(input_pattern);
}
```

There will be scenarios where a procedural approach is more advanageous to you.  For instance, if a quick and easy
command is needed, it might not make sense to create an entire method for it.  In those situations you can use the ```Pattern``` property
of CommandProcessor.  It allows you to inject pattern evaluators directly into the CommandLineProcessor without having 
to create a class method.

```
processor.Pattern["reset @parameter"] = (placeholders) =>
{
    foreach(var placeholder in placeholders.Keys)
        Console.WriteLine($"{placeholder} = {placeholders[placeholder]}");
    return true;
};
```
One key difference between the two approaches is that with this, the placeholders are not automatically converted and mapped to the 
method's parameters.  With this approach a dictionary(of string) is passed into the lambda.  If multiple patterns need to be mapped to one
lambda they can be passed comma separated.  An example of multiple dynamic patterns using the same lamda is shown below.

```
processor.Pattern
[
    "run @code_name [override? @role]",
    "stop @application"
] = (placeholders, context) =>
{
    var is_run = context.HasToken("run");
    if (is_run)
    {
        var allow_override = context.HasToken("override");
        if (allow_override)
        {
            Console.WriteLine($"*** Running {placeholders["code_name"]} with [{placeholders["role"]}] access ***");
        }
        else
            Console.WriteLine($"Running {placeholders["code_name"]} with normal access");
    }
    else
    {
        Console.WriteLine($"Stopping {placeholders["application"]}");
    }

    return true;
};

```

Please note that this collection **Cannot** be updated or iterated through.  This is mainly a mechanism for
injecting additional patterns with their associated handlers into the processor.  Also note that patterns declared
here will be processed **before** any patterns statically declared (declared with method and attribute notation).
In this way they can override the intended functionality of a given pattern so be careful with them.
