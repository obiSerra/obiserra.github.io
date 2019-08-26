---
layout: post
title:  "Unit tests and code smells"
date:   2019-08-26 17:10:22 +0200
categories: refactoring unit-tests C#
---

# Unit tests and code smells

I am a massive fan of unit tests, especially when working on a big project with a massive codebase for two main reasons:

 1. they simplify your job when checking how some feature behaves in different scenarios by removing all the condition to generate a certain case
 1. writing tests can often highlight bad pattern inside your code

A few days ago I was working on a module inside a C# project. The module job was to fetch some data from multiple providers, modify them and arrange them into a data structure that was then used by another module to generate a JSON. My job was to add a new section inside the JSON.

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

 The whole class wasn't tested, so after adding my `MyModule.Bottom`, I started to write a few tests at least for my new section: I wanted to check that everything worked fine 
 on different responses .
 
 I stopped trying after a few hours: it was impossible to mock or to instantiate all the modules needed by the main class, even if I wanted to test only the small part I had developed.
 So I ended up refactoring the code into a few single classes containing the logic for each section and returning a value that was then used by the main class to generate the data structure to serialize.

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
The need for testable code brought us to improve the overall code, applying The Single Responsibility Principle and The Principle of Least Knowledge.

_Tip:_ Using the Dependency Injection and Inversion of Control the couplings between each class was further reduced.

_Bonus Point:_ after a few days we were able to easily reuse some of the classes in another main class to generate a slightly different JSON.