# Custom CLI with Zaria.AI
Today I’m going to be showing you how to create your own custom CLI tool.  In this example, Let’s pretend that I am a developer who is working on building out user profile pages for a product my company is developing.  I have the table created and I’m ready to go but I find that I am constantly having to fire up SQL management studio and manipulate the data to try out various test scenarios.  I’ll be showing you how custom command lines can quickly and easily manage repetitive tasks that would otherwise be a tedious distraction.

So, what is a CLI anyways and why might you need one.

A CLI, which stands for command-line interpreter, uses a text driven prompt-and-response pattern to receive commands from a user in the form of lines of text. This provides a means of setting parameters for the environment, invoking executables and even providing information to them as to what actions they are allowed to perform. Now as you know, these days, many users rely on the menu driven interactions of graphical user interfaces, commonly referred to as GUIs, to get things done; however, some programming and maintenance tasks still work best as simple text-based input commands.  To me, an essential power of command lines over GUIs is their unique ability to be run not just as individual activities but also as pseudo-orchestrators combining multiple activities, either by using outputs of one as inputs to another or by reading data from a common source and manipulating it as needed as each task executes. 

We still see many examples of the use of CLIs for both developers and system administrators.  Products like Git, NuGet, Docker, and Azure CLI are just a few examples of products that provide a CLI as a first-class tool, if not the primary tool, for interaction.
Let’s get going.

## Initial activities
> For the purposes of this demo we will be using visual studio code.  You can follow along with me if you have your own copy.  As a first step we will create a folder for our project called carlton (that will be the name of our command line helper).  So lets use the mkdir command to do that.

> Next we simply need to navigate into that folder from the command line and type ```code .``` to open up visual studio code in the context of that folder

> Visual Studio Code has a built-in command terminal that we will be using  so lets open that up either by using the top menu or by using  ```ctrl + shift + tilde```.  Anytime you need to close the terminal for any reason you can use ```control tilde``` to do that.  We wont be needing the terminal you used to start it so we will close that.

> Now what we want to do for starters is replicate the functionality of populating and truncating data from the *SampleUsers* table and we will be doing this with a C# console application.  So we start by typing ```dotnet new console``` from the terminal. In light of what we are trying to do here you can see that even the .NET tooling usese commands to accomplish various tasks - in this case it is creating all the appropriate files and folders that you need for the console application project. 
1. Go into project file and disable nullable 

> On the explorer panel you should now see those setup.  We can test to see if all is well by using the ```dotnet build``` command.  

> And finally to run it we use ```dotnet run```  And you can see that the executable runs, resulting in "Hello World" getting printed to the console window.

> For this particular sample we will not be using top level statements, so we'll replace the code in Program.cs with normal boiler plate C# console application code.  We will also be adding an additional file for storing any global usings that we want applied accross the project and one for storing our connection string.  
```
namespace Carlton;

public class Program
{
    public static void Main(string[] args)
    {
        Console.WriteLine("Carlton Helper CLI");
    }
}
```

> We now build and test this to ensure everything is still working as intended. 

> With that done we now need to add any necessary database libraries.  The important ones are the packages for entity framework.  So we can go to nuget.org and search for EntityFrameworkCore. 

```
dotnet add package Microsoft.EntityFrameworkCore --version 7.0.2
dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 7.0.2
```
 
 From nuget.org we can see that the .NET CLI has a command we can use to add it.  Simply copy that and paste it into the terminal window.

 > While we are here, we will also be using a package called Zaria.AI to do all our command processing so lets go ahead and add that as well.  Zaria.AI provides a simple, attribute-driven, low-code API for building text based interactive conversations. It can be used to build AI BOTs, CLIs, general command processing and a whole lot more.  Once we are done we can build and run to ensure that there were no issues.
 ```
 dotnet add package Zaria.AI --version 1.0.1
 ```
> Finally lets update the global namespace with the namespaces that will be used accross all these packages
```
global using System.ComponentModel.DataAnnotations;
global using Microsoft.EntityFrameworkCore;
```

## Coding the CLI
> Let's get starting with our excercise by creating the foundations for accessing data.
1. Create a data context class
2. Add a connectionstring property to it. 
3. Create the constructor and pass connection string into it
4. Override the ```OnConfiguring``` method and in it pass the connection string into the ```UseSqlServer``` method
```
public class AppDB : DbContext
{
    public string ConnectionString{get;set;}

    public AppDB(string connection_string)
    {
        ConnectionString = connection_string;
    }

    protected override void OnConfiguring(DbContextOptionsBuilder builder)
    {
        builder.UseSqlServer(ConnectionString);
    }

   
}
```
5. Create a class representing ```SampleUsers``` - copying it in
```
public class SampleUser
{
    [Key]
    public Guid UserID { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public int Credits { get; set; }

    public bool IsActive { get; set; }
    public DateTimeOffset CreateDate { get; set; }
}
```
6. Create a DBbSet in App
```
 public DbSet<SampleUser> SampleUsers{get;set;}
```
7. Build and run to ensure all is well.
8. I'm going to refactor these into their own class now to make the code a little cleaner.

> Now that we are all set lets go ahead and add the function for truncating the database table.  
```
    static void ClearUsers()
    {
        Console.WriteLine($"Removing users...");
        AppDB database = new AppDB(PrivateStuff.ConnectionString);
        database.SampleUsers.RemoveRange(database.SampleUsers);
        database.SaveChanges();
    }
```

> Next we can create the method for adding users into the system.  
```
    static void PopulateUsers()
    {
        Console.WriteLine($"Populating users...");
        AppDB database = new AppDB(PrivateStuff.ConnectionString);
        int count = 10;
        foreach(var index in Enumerable.Range(0, count))
        {
            database.SampleUsers.Add(new SampleUser
            {
                UserID = Guid.NewGuid(),
                FirstName = $"john{index}",
                LastName = $"doe{index}",
                Credits = 0,
                IsActive = true,
                CreateDate = DateTimeOffset.UtcNow,
            });
        }
        database.SaveChanges();
        Console.WriteLine($"{count} users added.");
    }
```
> In the main method of the console
Once we are done we build and run this and we should now see that the 5 records that were previously in the database are now gone.

> But what if we wanted to add users but not clear them, or clear users without adding them.  Or what if we wanted to add more or less than 10 users?  the current approach is limiting because it requires a compilation for every change we make to the tool - and the variability is bound to the code of the tool as opposed to being injected into it by the user.  This is were Zaria.AI comes in.  With it, we can modify our code base to allow for comamnds to be passed in that correspond to the activity we want performed.  Lets start by updating the code to allow us to decide whether we want to clear or populate user.
1. Update the global.cs to include the namespaces for Zaria
```
...
global using Zaria.AI;
```

2. Add a pattern attribute to clear users and populate user. 
3. Zaria requires that the method being decorated with the attribute returns a bool so modify the return types for both methods to bool.  We'll talk about the meaning of the returned value in a little bit.
```
    [Pattern("clear users")]
    static bool ClearUsers()
    {
        Console.WriteLine($"Removing users...");
        AppDB database = new AppDB(PrivateStuff.ConnectionString);
        database.SampleUsers.RemoveRange(database.SampleUsers);
        database.SaveChanges();
        return true;
    }

    [Pattern("populate users")]
    static bool PopulateUsers()
    {
        Console.WriteLine($"Populating users...");
        AppDB database = new AppDB(PrivateStuff.ConnectionString);
        int count = 10;
        foreach(var index in Enumerable.Range(0, count))
        {
            database.SampleUsers.Add(new SampleUser
            {
                UserID = Guid.NewGuid(),
                FirstName = $"john{index}",
                LastName = $"doe{index}",
                Credits = 0,
                IsActive = true,
                CreateDate = DateTimeOffset.UtcNow,
            });
        }
        database.SaveChanges();
        Console.WriteLine($"{count} users added.");
        return true;
    }
```
4. Remove ClearUsers and PopulateUsers calls from Main and instead add 
```
        var processor = CommandProcessor.Initialize();
        processor.Execute(args);
```
5. build
6. Add a new terminal (DOS not powershell) and here navigate to the location of the carlton binary
which should be under ```bin\debug\<your .net version>\```
7. Type ```carlton``` and hit enter.  

> The program should run normally but when it actually executes you will note that the prompt that tells you how many users were created is not showing.  What's happening?  Well since we no longer directly call those functions we will need to call them by passing in the appropriate text as arguments into the *carlton* executable.   So instead, run 
```
carlton clear users
```
From the database we can see that all users have been cleared.  To populate users we can use
```
carlton populate users
```

> This is a great start but still lacks the level of variability we need.  Although we can clear and populate users, the number of users we can create is locked to 10.  We can change that by passing in the *count* value.  **Zaria.AI** supports adding placeholders in patterns.  To indicate that a token is a placeholder simply add the **@** symbol in front of it.  For any placeholder, the value will be passed into a parameter on the handler method that has the same name.  So we would add a count paramter to ```PopulateUsers``` and then add a ```@count``` placeholder that connects it to the user input.  Zaria will process the input text and match the appropriate token to the variable then pass that value into it when the handler is invoked. Lets modify ```PopulateUser``` to take advantage of this.  The following types can be passed in as paramters to a Pattern hander:
- int
- string
- Guid
- bool
- decimal
- CommandContext
> CommandContext contains a context into what was entered by the user.  It can be used to access the pipeline state (in scenarios where more than one command is passed into the pipeline), and to traverse the input text that was passed in.  Its value is not passed in by the user, it is implicitly passed in by the framework when the parameter is included in the handler method's parameters.


```
    [Pattern("populate @count users")]
    static bool PopulateUsers(int count = 10)
    {
        Console.WriteLine($"Populating {count} users...");
        AppDB database = new AppDB(PrivateStuff.ConnectionString);

        foreach(var index in Enumerable.Range(0, count))
        {
            database.SampleUsers.Add(new SampleUser
            {
                UserID = Guid.NewGuid(),
                FirstName = $"john{index}",
                LastName = $"doe{index}",
                Credits = 0,
                IsActive = true,
                CreateDate = DateTimeOffset.UtcNow,
            });
        }
        database.SaveChanges();
        Console.WriteLine($"{count} users added.");
        return true;
    }
```
> When we build and run this we see that we are now able to pass values into the hander.
Note that if we attempt to pass in ```carlton populate users``` now it fails.  This is because it has to properly match in order to succeed.  **Zaria.AI** also provides the ability to declare a method as a catch-all for any patterns that are not found by decorating it with the ```NotFound``` attribute.  This way the user can be trained on what they can and can't do with the CLI.  Before we test our changes lets add a new method and decorate it with that.
```
[NotFound]
static bool CatchAll(CommandContext context)
{
    Console.ForegroundColor = ConsoleColor.Yellow;
    Console.WriteLine($"\nCommand Not Found");
    Console.ResetColor();
    return true;
}
```
Only 1 NotFound attribute is required.  The first handler that satisfies that criteria will be taken and all others ignored.  With that done we can build and test both scenarios.  

> The framework also allows you to run multiple commands in succession by seperating them with a semicolor.  So to clear all users and the populate 25 the caller would type 
```
carlton clear users; populate 25 users
``` 
There is no limit to the number of commands that can be passed in this manner so one could theoretically input
```
carlton clear users; populate 25 users; populate 12 users; populate 3 users;
``` 
and from this 40 users would get created.  This lends itself to the reason for the bool return type and what value should be passed back.  As long as you return true, the next step in the pipeline will be executed.  Returning false stops the flow after the currently executing command completes.

> Next lets update the user population process by allowing the caller to also pass in the user's first and last name.  We dont want to force the caller to do this all the time though, so we will be using to more capabilities of **Zaria.AI** to limit this.  **Optional Tokens** and **Groups**.  For any text in the input pattern that is not a placeholder (does not work with placeholders) you can place a questionmark after it to make it options.  So if we wanted the caller to have the ability to omit the *users* part of the ```populate users``` command we would change it to 
```
populate @count users?
```  
with this 
both ```populate 10 users```  and    ```populate 10``` would trigger the hander.

> you can place things in groups by enclosing them in square brackets.  There are a few rules to groups:
1. A group can have 1 or more words in it.
2. Groups can appear at any point in the input string after the last static word in the string. 
3. A group cannot be started with a placeholder
4. If the first word of a group is optional, omitting it means that all words in that group are also ommitted

For the following pattern ```populate @count [users] [now]``` here are the input strings that match

Input | result
---| ---
populate 10 users now | yes
populate 10 now users | yes

Every other variation will fail

For the following pattern ```populate @count [users?] [now?]``` here are the input strings that match

Input | result
---| ---
populate 10 | yes
populate 10 users | yes
populate 10 now | yes
populate 10 users now | yes
populate 10 now users | yes

Every other variation will fail.

> 

With all this in mind lets update the PopulateUsers table to allow for passing in names.
```
[Pattern("populate @count users [with? first name @base_firstname] [and? last name @base_lastname]")]
static bool PopulateUsers(CommandContext context, int count = 10, string first_name= "John", string last_name= "Doe")
{
    Console.WriteLine($"Populating {count} users...");
    AppDB database = new AppDB(PrivateStuff.ConnectionString);
    
        if (context.HasToken("with"))
        first_name = context.GetToken("with").Skip(3).Text;

    if (context.HasToken("and"))
        last_name = context.GetToken("and").Skip(3).Text;

    foreach(var index in Enumerable.Range(0, count))
    {
        Console.Write(".");
        database.SampleUsers.Add(new SampleUser
        {
            UserID = Guid.NewGuid(),
            FirstName = $"{first_name}_{index}",
            LastName = $"{last_name}_{index}",
            Credits = 0,
            IsActive = true,
            CreateDate = DateTimeOffset.UtcNow,
        });
    }
    database.SaveChanges();
    Console.WriteLine($"{count} users added.");
    return true;
}
```
The following input text will all match

```
carlton populate 10 users
carlton populate 10 users with first name lebron
carlton populate 10 users and last name james
carlton populate 10 users with first name lebron and last name james
carlton populate 10 users and last name james with first name lebron 
```
