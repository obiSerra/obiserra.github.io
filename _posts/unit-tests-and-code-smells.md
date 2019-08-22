---
layout: post
title:  "Unit tests and code smells"
---

# Unit tests and code smells


I am a massive fan of unit tests, expecially when working on big project with a massive code base for two main reasons:

1. they simplify your job when check how some feature behave in different scenarios by removing all the condition to generate a certain case
1. writing tests can often highligh bad pattern inside your code

Few days ago I was working on "module(@?)" inside a C# project. The module job was to fetch some data from multiple providers, modify them and arrange them
into a data structure that was then used by anoter module to generate a json. My job was to add a new section inside the json.

The code was organized something like this:

``` C#
    public class MyModule
    {
        private JsonSerializerModule _serializer;
        private Content _moduleContent;
    
        public MyModule(JsonSerializerModule serializer, ... // a bunch of other dependencies needed by the partial modules)
        {
         _serializer = serializer;
         _moduleContent = new ModuleContent();
        }

        public void Run ()
        {
          GenerateTopSection();
          GenerateMiddleSection();

         _serializer.Serialize(_moduleContent);
        }
    }

    public partial class MyModule.Top
    {
        private void GenerateTopSection()
        {
            var topSection = ...; // here some dependencies are used
            _moduleContent.AddContent(topSection);
        }
    }

    public partial class MyModule.Middle
    {
        private void GenerateMiddleSection()
        {
            var middleSection = ...; // here other dependencies are used
            _moduleContent.AddContent(middleSection);
        }
    }

    ...

``` 

The all class wasn't tested, so after adding my `MyModule.Bottom`, I started to write few tests at least for my new section: I wanted to check that everything worked fine
on different responses from the data provider I was using.

I stopped trying after few hours: it was impossible to mock or tu instantiate all the modules needed by the main class, even if I wanted to test only the small part I had developed.
So I ended up refatoring the code into few single classes containg the logic for each section and returning a value that was then used by a main class to generate the data structure to serialize.


```C#

    public class MyModuleMain
    {
        private JsonSerializerModule _serializer;
        private TopProvider _topProvider;
        private Content _moduleContent;
        ...
    
        public MyModule(JsonSerializerModule serializer, TopProvider topProvider, ... // all the section subclasses 
        {
         _serializer = serializer;
         _topProvider = topProvider;
         _moduleContent = new ModuleContent();
        }

        public void Run ()
        {
          _moduleContent.AddContent(GenerateTopSection());
          ...

         _serializer.Serialize(_moduleContent);
        }
    }

    public class TopProvider
    {
        private Section GenerateTopSection(... // only the deplendecies for this module)
        {
            var topSection = ...; // here some dependencies are used
            return topSection;
        }
    }

    public class MiddleProvider
    {
        private Section GenerateMiddleSection(... // only the dependencies for this module)
        {
            ...
        }
    }

    ...

```


The need for testable code brought us to improve the overall code, applying the The Single Responsibility Principle and The Principle of Least knowledge.

_Tip:_ Using the Dipendency Injection and Invertion of Control the couplings between each class was further reduced.

_Bonus Point:_ after few days we were able to easily reuse some of the classes in another main class to generate a slightly differnt json.
