###Binary
PowerArgs is available at the [Official NuGet Gallery](http://nuget.org/packages/PowerArgs).

Check out the [new homepage](http://www.powerargs.com) and [interactive demo](http://www.powerargs.com/home/demo).

###Overview

PowerArgs converts command line arguments into .NET objects that are easy to program against.  It also provides a ton of additional, optional capabilities that you can try such as argument validation, auto generated usage, tab completion, and plenty of extensibility.

Here's a simple example.
    
    // A class that describes the command line arguments for this program
    public class MyArgs
    {
        // This argument is required and if not specified the user will 
        // be prompted.
        [ArgRequired(PromptIfMissing=true)]
        public string StringArg { get; set; }

        // This argument is not required, but if specified must be >= 0 and <= 60
        [ArgRange(0,60)]
        public int IntArg {get;set; }
    }

    class Program
    {
        static void Main(string[] args)
        {
            try
            {
                var parsed = Args.Parse<MyArgs>(args);
                Console.WriteLine("You entered string '{0}' and int '{1}'", parsed.StringArg, parsed.IntArg);
            }
            catch (ArgException ex)
            {
                Console.WriteLine(ex.Message);
                Console.WriteLine(ArgUsage.GetUsage<MyArgs>());
            }
        }
    }
    
###Metadata Attributes 
These can be specified on argument properties.
 
    [ArgPosition(0)]                                    // This argument can be specified by position (no need for -propName)
    [ArgShortcut("n")]                                  // Let's the user specify -n
    [ArgDescription("Description of the argument")]
    [ArgExample("example text", "Example description")]
    [DefaultValue("SomeDefault")]                       // Specify the default value
    [ArgIgnore]                                         // Don't populate this property as an arg
    [StickyArg]                                         // Use the last used value if not specified
    [Query(typeof(MyDataSource))]                       // Easily query a data source
    [TabCompletion]                                     // Enable tab completion for parameter names (Can be customized)
    
###Validator Attributes
These can be specified on argument properties.  You can create custom validators by implementing classes that derive from ArgValidator.

    [ArgRequired(PromptIfMissing=bool)] 
    [ArgExistingFile]
    [ArgExistingDirectory]
    [ArgRange(from, to)]
    [ArgRegex("MyRegex")]               // Apply a regular expression validation rule
    [UsPhoneNumber]                     // A good example of how to create a custom data type

###Latest Features

Access your parsed command line arguments from anywhere in your application.

    MyArgs parsed = Args.GetAmbientArgs<MyArgs>();
    
This will get the most recent insance of type MyArgs that was parsed on the current thread.  That way, you have access to things like global options without having to pass the result all throughout your code.

###Styled Usage

Enhancements to auto-generated usage documentation.  Here is some sample output.
    
![Sample styled output](https://github.com/adamabdelhamed/PowerArgs/blob/master/StyledUsageExampleOutput.PNG?raw=true "Sample Output")


Here's the code that generates that output.

    using System;
    using PowerArgs;

    namespace HelloWorld
    {
        [TabCompletion]
        [ArgExample("HelloWorld -s SomeString -i 50 -sw", "Shows how to use the shortcut version of the switch parameter")]
        public class MyArgs
        {
            [ArgDescription("Description for a required string parameter")]
            public string StringArg { get; set; }
    
            [ArgDescription("Description for an optional integer parameter")]
            public int IntArg { get; set; }
    
            [ArgDescription("Description for an optional switch parameter")]
            public bool SwitchArg { get; set; }
    
            [ArgDescription("Shows the help documentation")]
            public bool Help { get; set; }
        }
    
        class Program
        {
            static void Main(string[] args)
            {
                try
                {
                    var parsed = Args.Parse<MyArgs>(args);
                    if (parsed.Help)
                    {
                        ArgUsage.GetStyledUsage<MyArgs>().Write();
                    }
                    else
                    {
                        Console.WriteLine("You entered StringArg '{0}' and IntArg '{1}', switch was '{2}'", parsed.StringArg, parsed.IntArg, parsed.SwitchArg);
                    }
                }
                catch (ArgException ex)
                {
                    Console.WriteLine(ex.Message);
                    ArgUsage.GetStyledUsage<MyArgs>().Write();
                }
            }
        }
    }

### Secure String Arguments

Support for secure strings such as passwords where you don't want your users' input to be visible on the command line.

Just add a property of type SecureStringArgument.

    public class TestArgs
    {
        public SecureStringArgument Password { get; set; }
    }
    
Then when you parse the args you can access the value in one of two ways.  First there's the secure way.

    TestArgs parsed = Args.Parse<TestArgs>();
    SecureString secure = parsed.Password.SecureString; // This line causes the user to be prompted

Then there's the less secure way, but at least your users' input won't be visible on the command line.

    TestArgs parsed = Args.Parse<TestArgs>();
    string notSecure = parsed.Password.ConvertToNonsecureString(); // This line causes the user to be prompted

###Tab Completion

Get tab completion for your command line arguments.  Just add the TabCompletion attribute and when your users run the program from the command line with no arguments they will get an enhanced prompt (should turn blue) where they can have tab completion for command line argument names.

    [TabCompletion]
    public class TestArgs
    {
        [ArgRequired]
        public string SomeParam { get; set; }
        public int AnotherParam { get; set; }
    }

Sample usage:

    someapp -some  <-- after typing "-some" you can press tab and have it fill in the rest of "-someparam"

You can even add your own tab completion logic in one of two ways.  First there's the really easy way.  Derive from SimpleTabCompletionSource and provide a list of words you want to be completable.

    public class MyCompletionSource : SimpleTabCompletionSource
    {
        public MyCompletionSource() : base(MyCompletionSource.GetWords()) {}
        private static IEnumerable<string> GetWords()
        {
            return new string[] { "SomeLongWordThatYouWantToEnableCompletionFor", "SomeOtherWordToEnableCompletionFor" };
        }
    } 
 
 Then just tell the [TabCompletion] attribute where to find your class.
 
    [TabCompletion(typeof(MyCompletionSource))]
    public class TestArgs
    {
        [ArgRequired]
        public string SomeParam { get; set; }
        public int AnotherParam { get; set; }
    }
 
 There's also the easy, but not really easy way if you want custom tab completion logic.  Let's say you wanted to load your auto completions from a text file.  You would implement ITabCompletionSource.
 
    public class TextFileTabCompletionSource : ITabCompletionSource
    {
        string[] words;
        public TextFileTabCompletionSource(string file)
        {
            words = File.ReadAllLines(file);
        }

        public bool TryComplete(bool shift, string soFar, out string completion)
        {
            var match = from w in words where w.StartsWith(soFar) select w;

            if (match.Count() == 1)
            {
                completion = match.Single();
                return true;
            }
            else
            {
                completion = null;
                return false;
            }
        }
    }
    
If you expect your users to sometimes use the command line and sometimes run from a script then you can specify an indicator string.  If you do this then only users who specify the indicator as the only argument will get the prompt.

    [TabCompletion("$")]
    public class TestArgs
    {
        [ArgRequired]
        public string SomeParam { get; set; }
        public int AnotherParam { get; set; }
    }


###Data Source Queries

Easily query a data source such as an Entity Framework Model (Code First or traditional) using Linq.

    // An example Entity Framework Code First Data Model
    public class DataSource : DbContext
    {
        public DbSet<Customer> Customers{get;set;}
    }

    public class TestArgs
    {
        public string OrderBy { get; set; }
        [ArgShortcut("o-")]
        public string OrderByDescending { get; set; }
        public string Where { get; set; }
        public int Skip { get; set; }
        public int Take { get; set; }

        [Query(typeof(DataSource))]
        [ArgIgnore]
        public List<Customer> Customers { get; set; }
    }   
 
 That's it!  PowerArgs will make the query for you using the query arguments.  It's all done by naming convention.  
 
 Now just consume the data in your program.
    
    // Sample command that queries the Customers table for newest 10 customers  
    // <yourapp> -skip 0 -take 10 -where "item.DateCreated > DateTime.Now - TimeSpan.FromDays(1)" -orderby item.LastName
    
    var parsed = Args.Parse<TestArgs>(args);
    var customers = parsed.Customers;
    
###Programs with actions
This example shows various metadata and validator attributes.  It also uses the Action Framework that lets you separate your program into distinct actions that take different parameters.  In this case there are 2 actions called Encode and Clip.  It also shows how enums are supported.

        [TabCompletion, ArgExceptionBehavior(ArgExceptionPolicy.StandardExceptionHandling), ArgExample("superencoder encode fromFile toFile -encoder Wmv", "Encode the file at 'fromFile' to an AVI at 'toFile'")]
        public class VideoEncoderArgs
        {
            [ArgActionMethod]
            public void Encode(EncodeArgs args)
            {
                Console.WriteLine("Encode called");
            }

            [ArgActionMethod]
            public void Clip(ClipArgs args)
            {
                Console.WriteLine("Clip called");
            }
        }

        public enum Encoder
        {
            Mpeg,
            Avi,
            Wmv
        }

        public class EncodeArgs
        {
            [ArgRequired, ArgExistingFile, ArgPosition(1)]
            [ArgDescription("The source video file")]
            public string Source { get; set; }

            [ArgPosition(2)]
            [ArgDescription("Output file.  If not specfied, defaults to current directory")]
            public string Output { get; set; }

            [DefaultValue(Encoder.Avi)]
            [ArgDescription("The type of encoder to use")]
            public Encoder Encoder { get; set; }
        }

        public class ClipArgs : EncodeArgs
        {
            [ArgRange(0, double.MaxValue)]
            [ArgDescription("The starting point of the video, in seconds")]
            public double From { get; set; }

            [ArgRange(0, double.MaxValue)]
            [ArgDescription("The ending point of the video, in seconds")]
            public double To { get; set; }
        }
    
    class Program
    {
        static void Main(string[] args)
        {
             Args.InvokeAction<VideoEncoderArgs>(args);
        }
    }
    
###Custom Revivers
Revivers are used to convert command line strings into their proper .NET types.  By default, many of the simple types such as int, DateTime, Guid, string, char,  and bool are supported.

If you need to support a different type or want to support custom syntax to populate a complex object then you can create a custom reviver.

This example converts strings in the format "x,y" into a Point object that has properties "X" and "Y".

    public class CustomReviverExample
    {
        // By default, PowerArgs does not know what a 'Point' is.  So it will 
        // automatically search your assembly for arg revivers that meet the 
        // following criteria: 
        //
        //    - Have an [ArgReviver] attribute
        //    - Are a public, static method
        //    - Accepts exactly two string parameters
        //    - The return value matches the type that is needed

        public Point Point { get; set; }

        // This ArgReviver matches the criteria for a "Point" reviver
        // so it will be called when PowerArgs finds any Point argument.
        //
        // ArgRevivers should throw ArgException with a friendly message
        // if the string could not be revived due to user error.
      
        [ArgReviver]
        public static Point Revive(string key, string val)
        {
            var match = Regex.Match(val, @"(\d*),(\d*)");
            if (match.Success == false)
            {
                throw new ArgException("Not a valid point: " + val);
            }
            else
            {
                Point ret = new Point();
                ret.X = int.Parse(match.Groups[1].Value);
                ret.Y = int.Parse(match.Groups[2].Value);
                return ret;
            }
        }
    }

###Auto-Generated Usage
You can get an auto generated usage string by passing your argument type to the GetUsage<T> function like this:

    string usage = ArgUsage.GetUsage<VideoEncoderArgs>();

Sample Output:

    Usage: superencoder <action> options

    EXAMPLE: superencoder encode fromFile toFile -encoder Wmv
    Encode the file at 'fromFile' to an AVI at 'toFile'

    Global options:

       OPTION         TYPE     ORDER   DESCRIPTION                       
       -whatif (-w)   Switch           Simulate the encoding operation   

    Actions:

    Encode - Encode a new video file

       OPTION          TYPE      ORDER   DESCRIPTION                                                    
       -source (-s)    String*   1       The source video file                                          
       -output (-o)    String    2       Output file.  If not specfied, defaults to current directory   
       -encoder (-e)   Encoder           The type of encoder to use                                     


    Clip - Save a portion of a video to a new video file

       OPTION          TYPE      ORDER   DESCRIPTION                                                    
       -source (-s)    String*   1       The source video file                                          
       -output (-o)    String    2       Output file.  If not specfied, defaults to current directory   
       -from (-f)      Double            The starting point of the video, in seconds                    
       -to (-t)        Double            The ending point of the video, in seconds                      
       -encoder (-e)   Encoder           The type of encoder to use  

This was the code that resulted in the generated usage string.

    [ArgExample("superencoder encode fromFile toFile -encoder Wmv", "Encode the file at 'fromFile' to an AVI at 'toFile'")]
    public class VideoEncoderArgs
    {
        [ArgRequired]
        [ArgPosition(0)]
        [ArgDescription("Either encode or clip")]
        public string Action { get; set; }

        [ArgDescription("Encode a new video file")]
        public EncodeArgs EncodeArgs { get; set; }
        [ArgDescription("Save a portion of a video to a new video file")]
        public ClipArgs ClipArgs { get; set; }

        [ArgDescription("Simulate the encoding operation")]
        public bool WhatIf { get; set; }

        public static void Encode(EncodeArgs args)
        {
            // TODO - Implement Encode Action
        }

        public static void Clip(ClipArgs args) 
        {
            // TODO - Implement Clip action
        }
    }

    public enum Encoder
    {
        Mpeg,
        Avi,
        Wmv
    }

    public class EncodeArgs
    {
        [ArgRequired]
        [ArgExistingFile]
        [ArgPosition(1)]
        [ArgDescription("The source video file")]
        public string Source { get; set; }

        [ArgPosition(2)]
        [ArgDescription("Output file.  If not specfied, defaults to current directory")]
        public string Output { get; set; }

        [DefaultValue(Encoder.Avi)]
        [ArgDescription("The type of encoder to use")]
        public Encoder Encoder { get; set; }
    }

    public class ClipArgs : EncodeArgs
    {
        [ArgRange(0, double.MaxValue)]
        [ArgDescription("The starting point of the video, in seconds")]
        public double From { get; set; }
            
        [ArgRange(0, double.MaxValue)]
        [ArgDescription("The ending point of the video, in seconds")]
        public double To { get; set; }
    }                                   
