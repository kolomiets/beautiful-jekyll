---
layout: post
title: "A Legacy Project - Case Study and a bit of history"
date: "2016-10-27 17:40:50 +0300"
tags: [legacy]
comments: true
---
Well, before we start, I would like to give you some context of a project I will
write about. I believe it would be much more interesting to chat about something
concrete, not about abstract matters. It is much more funny to tell you
real stories about real people and decisions we did to learn from.

For the rest of the series let us call the project we examine as Project "A"
for brevity reasons.

So, we begin with some basics to set the stage...

## Legacy Project Size
Project A is a relatively big project - here is some statistic according to
[cloc tool](https://github.com/AlDanial/cloc) (I included all build scripts,
samples, documentation, .etc - all artifacts that comprise the project and what we
need to support anyway):

|Language                 |files|blank |comment|code   |
|-------------------------|-----|------|-------|-------|
|C#                       |5671 |174540|226094 |970048 |
|C/C++ Header             |1954 |176828|242462 |426980 |
|Java                     |1939 |44943 |61836  |224214 |
|XML                      |183  |635   |165    |176949 |
|C++                      |459  |19364 |20401  |121840 |
|JavaScript               |166  |20509 |22056  |94392  |
|MSBuild script           |280  |46    |1169   |78138  |
|HTML                     |664  |1069  |58     |56765  |
|C                        |85   |5685  |10419  |33420  |
|IDL                      |29   |4144  |2      |27715  |
|WiX string localization  |66   |1244  |498    |22794  |
|XSD                      |26   |11    |0      |22774  |
|CSS                      |59   |1470  |762    |16740  |
|ASP.Net                  |99   |1301  |37     |10033  |
|F#                       |32   |1438  |713    |7213   |
|Bourne Shell             |6    |865   |1101   |6097   |
|WiX source               |36   |392   |253    |4687   |
|XAML                     |19   |63    |61     |1773   |
|Windows Resource File    |34   |464   |543    |1464   |
|Visual Basic             |17   |232   |222    |879    |
|NAnt script              |5    |187   |94     |842    |
|XSLT                     |1    |57    |84     |597    |
|JSP                      |3    |96    |20     |573    |
|DOS Batch                |19   |177   |111    |570    |
|Assembly                 |1    |24    |71     |284    |
|WiX include              |5    |44    |14     |169    |
|SAS                      |2    |38    |89     |168    |
|CMake                    |1    |5     |3      |65     |
|Visualforce Component    |5    |0     |0      |57     |
|Windows Module Definition|2    |3     |8      |56     |
|JSON                     |3    |2     |0      |55     |
|INI                      |3    |4     |0      |39     |
|**SUM**                  |**11874**|**455880**|**589346** |**2308390**|

So, it is pretty big. You may execute `cloc` tool against your project of choice to
run a basic comparison.

As you can see, there is a zoo of languages involved. And, as you may rightfully
expect, there are some obsolete parts that most likely garbage (Bourne Shell
files??? In C#/C++ project? What are they for god's sake?)

Still, this analysis shows that Project A is mostly .NET project with some big
chunks of unmanaged C++ code. Java is the third most common language presented here
and rightfully so - project A has a Java support, but we can ignore it
for now - it is a whole different story.

## Project Structure
Now we briefly understand the size of the codebase, next question is how it is
structured?

I started to work on project A when it had the following structure:

 * Two main Visual Studio 2008 solutions, with **100+ projects** in common
 * A number of auxiliary VS solutions with components that we change rarely and
 build only when needed. Another **20+ VS2008 projects**
 * Numerous samples, installers, documentation builds, etc.

All in all, I would estimate the whole product as **~200 projects** of various types
(C++, C#, Wix, Sandcastle, etc.)

All these projects lie on file system in *semi-structured* hierarchy. It was semi-structured,
because it *was* structured originally, but then order was broken and project folders
started to pop up everywhere. At the end you may only *guess* original intention
looking at piles of files and folders without clear naming convention and structure.

Well, at least build procedure knows how to glue all this stuff together, right?

## Build procedure
Build of the Project A was pretty complex and consisted of NANT build scripts that
performed the following main steps:

 * Update versions / copyright of the components
 * Build C++/C# sources
 * Obfuscate and digital sign the assemblies
 * Build / obfuscate / minify JavaScript sources
 * Run unit tests
 * Build demos and installers

Looks like a traditional workflow, right? Nothing too impressive, even NANT could
be forgiven. Devil is in small details though...

It happened that Project A had to be compiled in different configurations:

 * Both x86 and x64 architecture had to be supported (that's pretty typical
 if you have native C++ dependencies)
 * Both .NET Framework 2.0 and .NET Framework 4.0 had to be supported

All in all it gives you four (4) configurations to build:

|                       |x86|x64|
|-----------------------|---|---|
|**.NET Framework 2.0** | + | + |
|**.NET Framework 4.0** | + | + |

I mentioned that source code was mainly organized into VS2008 solutions. That is
clearly historical - I saw remnants of VS2005 project files that were upgraded to
VS2008 at some point when new IDE came out. The problem with VS2008 is that it
**can not** be used to target .NET Framework 4.0. It is just not possible - you
have to use newer versions of IDE.

To workaround this *problem*, NANT scripts and some evil imagination worked together
to perform the following transformations *at build time*:

 * VS2008 solution was converted to VS2010 format:

```xml
  <exec program="devenv.com"
    commandline="/upgrade &quot;${Dir.Source}\ProjectA.sln&quot;" />
```

 * C# projects (.csproj files) were updated to target 4.0 framework:

```xml
  <xmlpoke file="${filename}" value="v4.0"
    xpath="//x:Project/x:PropertyGroup[1]/x:TargetFrameworkVersion"
    failonerror="false" >
    <namespaces>
      <namespace prefix="x"
        uri="http://schemas.microsoft.com/developer/msbuild/2003" />
    </namespaces>
  </xmlpoke>
```
 * Of course, Managed C++ projects should be updated in completely different manner
 and another piece of NANT was used to do that (too large to include it here)
 * After such an *upgrade* there was a number of broken references that had to be
 fixed. Of course, NANT magic was responsible for that as well
 * As with any rule, in real projects there are some exceptions - not all projects
 should be upgraded, so there was a list hardcoded in (you guessed it already)
 NANT scripts

Similar method was employed to *convert* VS2008 solution to target x64 architecture.
With even more exceptions and tricks along the way.

All in all it resulted in 3k lines of pretty brittle and elaborate NANT build
script that was a nightmare to read and almost impossible to support. The worst
thing you may have missed is that it all happened *at build time*, therefore
there were no easy way to build x64 or .NET 4.0 flavor of the product locally.
Developers usually worked with original VS2008 solution that targeted .NET2.0 + x86
and *hoped* that the same code would work for other configurations as well.

Well, you may say, probably there is a bright side - what about unit tests? If
you have extensive collection of tests, you may be relatively safe if you execute
them against all four configurations, right?

## Unit tests
Unit tests is actually a great safety net for legacy projects. If you have some,
consider yourself lucky and try as best as you could to keep them alive.

We *were* lucky - there were about 4k unit tests. That's a big number. And these
were not exactly *unit* tests, more like integration ones, since they worked with
real data, read and wrote files from network shares, etc. Bad thing about them is
that you had to spend about an hour to execute them all. **For one configuration**.
That is right - you had to wait **4 hours** to check all flavors of the product.

Bottom line is that *nobody* executed all unit tests locally, ever - it was
impractical. Much easier was to run tests for single component/configuration and
hit commit button. The hope was that build server is good enough to run all the
tests and report promptly about the problems. Vicious practice that waited for
disaster...

## Continuous integration
Continuous integration for Project A was built with
[Cruise Control .NET](http://www.cruisecontrolnet.org/) server that ran on
physical box under someone's desk. There were another 4-5 physical slave machines
that were used to run the builds.

The team tried to solve the problem of long-running tests and ran builds of different
project configurations in parallel, on different builds machines. By itself it was
a good idea, that allowed to significantly reduce build time. However, in this particular
case it was implemented in the following way:

 * Each build configuration started on its own, designated build machine
 * Build scripts put artifacts of the build in special folder that was shared
 across the network
 * Once all four configuration builds were completed, installer build started
 * Installer (Wix based) used network shares to gather artifacts and bundle them
 together into single .msi package

There were **tons** of things that may (and did) break along the way. Network is
not a super reliable transport, especially if you copy large amounts of data
(and they did). Build machines they used were *different* (mix of Windows XP and
Windows Server 2003 boxes) and some tests passed on XP but failed on Server
(or vice versa). CC.NET configuration was so fragile that only one person truly
understood it, and when he left the company... Well, complete disaster.

## Discipline

When I joined the project, there could be **days** if not **weeks** of broken
builds. It means that there were no such thing as commit discipline - why should
you bother and thoroughly verify and test your change if build is down for several
days already? Someone else will surely fix it (eventually (right?)).

It was especially ugly if you need to deliver planned release, but you can't even
properly build the project! Desperate attempts to fix the build and meet the
deadlines resulted in product that was built on a dev box, without execution of
unit tests **at all**. Yes, Project A definitely saw dark days...

## Documentation

Ah, that is the worst thing. I hate bad documentation and Project A had a plenty
of that :)

Project A documentation consists of two main parts:

 * API Reference
 * Conceptual docs (aka Dev Guide)

First of all, due to the build degradation issues mentioned above, API Reference
part of the build actually **died**. Build stopped to produce accurate
documentation that reflected the project state. It produced *something else*
instead, which is the worst thing that can happen with docs.

Conceptual docs suffered differently. According to changed corporate policy,
build of the docs passed to Tech Doc team which did not have any product knowledge
and any desire to acquire it (no offense here - they had tons of work to do).
Therefore, PDFs that were produced were *unreadable* for the following reasons:

 * Just too long to be manageable - Dev Guide was **700 pages long** and I doubt
 that our customers ever read it (I once did, during a transatlantic flight, and
 I can say that I'd never wanted to sleep so badly)
 * **No code highlighting** - is it only me, but why they always tell me that it
 is *impossible* to add code highlighting in PDF files? In 21st century? Really?
 * Tons of errors - both grammatic and factual. Not a surprise, considering length
 of the document and associated cost of proofreading

## Conclusion
All in all, Project A was in a bad shape. Like a former runner who spent several years
in programmer's chair eating fast food and doing no exercises, all "limbs" of the
project felt rusty and clumsy. It just could not run anymore. It barely walked.

So what would you do with such project? Team is not motivated to change anything,
there are not obvious good parts that you can rely on. You can't stop all
activities and start restructuring - you have to deliver fixes and releases,
according to schedule.

What would you do?
