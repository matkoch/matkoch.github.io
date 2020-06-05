---
title: Reusable Build Components with Interface Default Implementations
key: reusable-build-components-with-interface-default-implementations
tags:
- Build Automation
- NUKE
- C# 8
- .NET
mode: immersive
header:
  theme: dark
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(0, 0, 100 , .5), rgba(60, 34, 60, .4))'
    src: assets/images/2020-06-05-reusable-build-components-with-interface-default-implementations/cover.jpg
twitter_card: assets/images/2020-06-05-reusable-build-components-with-interface-default-implementations/thumbnail.jpeg
---

***TL;DR: Default implementations in interfaces are a powerful approach to extract common build infrastructure into reusable components. They allow to overcome the diamond problem that we traditionally have when just using a hierarchy of base classes. By providing a rich target definition model, NUKE ensures that predefined build steps can easily be integrated with custom build steps, or extended without losing any of the existing information.***

C# 8 introduced a lot of [new language features](https://docs.microsoft.com/dotnet/csharp/whats-new/csharp-8) that can help[^1] us to make our codebase more readable, increase performance, or lower the memory footprint. However, one of these, namely [default implementations in interfaces](https://devblogs.microsoft.com/dotnet/default-implementations-in-interfaces/), seems to be more of a niche feature:

[^1]: I've also written about [C# 8 language features in action](https://blog.jetbrains.com/dotnet/2019/04/24/indices-ranges-null-coalescing-assignments-look-new-language-features-c-8/) on our [JetBrains .NET Blog](https://blog.jetbrains.com/dotnet)

<div class="tweet" tweetID="1248589138292604929">Every time a new version of C# comes out, I quickly find ways to adopt the newly introduced features into my code. C# 8 is an exception. Question: have you used default interface members in your code yet?</div>

In the context of build automation and specifically [NUKE](https://nuke.build), they have been [on my radar](https://youtu.be/SVD70QYvQ6I?t=2741) for some time already. After all, they 
Available time made it hard, but thanks to [Thomas Unger](https://github.com/tunger) this dream has [finally come true](https://github.com/nuke-build/nuke/pull/427)! 👏

## Writing Build Components

Utilizing different [strategies for build sharing](https://www.nuke.build/docs/sharing-builds/fundamentals.html) can greatly **reduce the maintenance effort** for our build infrastructure when different projects/repositories need to be build the same way. So they can help developers working in teams, but could also be used in the OSS community. With interface default implementations, we're extending the already existing strategy of **sharing infrastructure via NuGet packages**. But how exactly will this look like? Let's dive right into writing some build components:

{% highlight csharp linenos %}
interface IHasSolution
{
    [Solution]
    Solution Solution => GetInjectionValue(() => Solution);
}

interface IBuild : IHasSolution
{
    [Parameter]
    Configuration Configuration => GetInjectionValue(() => Configuration);

    Target Restore => _ => _
        .Executes(() =>
        {
            DotNetRestore(_ => _
                .SetProjectFile(Solution)
        });

    Target Compile => _ => _
        .DependsOn(Restore)
        .Executes(() =>
        {
            DotNetBuild(_ => _
                .SetProjectFile(Solution)
                .EnableNoRestore()
                .SetConfiguration(Configuration)
        });
}
{% endhighlight %}

With the `IHasSolution` interface, we are defining that our build has a `Solution` property that provides [access to our solution model](https://www.nuke.build/docs/authoring-builds/solutions-and-projects.html). The `IBuild` interface extends on that information and introduces two targets `Restore` and `Compile` that actually build the solution with a given configuration that can be passed as [parameter](https://www.nuke.build/docs/authoring-builds/parameter-declaration.html). In our build class, we only need to inherit those interfaces:

{% highlight csharp linenos %}
class Build : NukeBuild, IBuild
{
    public static void Main() => Execute<Build>();
}
{% endhighlight %}

Moving forward, we could also define one more interface `IPublishNuGet` that takes care of packing and pushing NuGet packages:

{% highlight csharp linenos %}
interface IPublishNuGet : IBuild
{
    AbsolutePath PackageDirectory { get; }

    Target Pack => _ => _
        .DependsOn(Compile)
        .Executes(() =>
        {
            DotNetPack(_ => _
                .SetProject(Solution)
                .EnableNoBuild()
                .SetConfiguration(Configuration)
                .SetOutputDirectory(PackageDirectory)
                .SetVersion(GitVersion.NuGetVersionV2)
        });

    [Parameter]
    string Source => GetInjectionValue(() => Source) ?? "https://api.nuget.org/v3/index.json";

    [Parameter]
    string ApiKey => GetInjectionValue(() => ApiKey);

    Target Publish => _ => _
        .DependsOn(Pack)
        .Executes(() =>
        {
            var packages = PackageDirectory.GlobFiles("*.nupkg");

            DotNetNuGetPush(_ => _
                    .SetSource(Source)
                    .SetApiKey(ApiKey)
                    .CombineWith(packages, (_, v) => _
                        .SetTargetPath(v)),
                degreeOfParallelism: 5,
                completeOnFailure: true);
        });
}
{% endhighlight %}

Again, we are extending our process on an existing interface: the `Pack` target from our new `IPublishNuGet` interface depends on the `Compile` target defined in the `IBuild` interface. Also note, how the interface now requires to actually implement a property `PackageDirectory`, which is used by the default implementation:

{% highlight csharp linenos %}
class Build : NukeBuild, IBuild, IPublishNuGet
{
    public static void Main() => Execute<Build>();

    string PackageDirectory => RootDirectory / "output" / "packages";
}
{% endhighlight %}

But what if we want to **extend the build pipeline in between**? For instance, to validate packages, clean our repository, or to announce a new release? With NUKE, this is rather easy using its [target definition model](https://nuke.build/docs/authoring-builds/fundamentals.html):

{% highlight csharp linenos %}
class Build : NukeBuild, IBuild, IPublishNuGet
{
    public static void Main() => Execute<Build>();
    
    Target Clean => _ => _
        .Before<IBuild>(x => x.Restore)
        .DependentFor<IPublishNuGet>(x => x.Publish)
        .Executes(() => { /* */ });

    Target ValidatePackages => _ => _
        .DependsOn<IPublishNuGet>(x => x.Pack)
        .DependentFor<IPublishNuGet>(x => x.Publish)
        .Executes(() => { /* */ });

    Target Announce => _ => _
        .TriggeredBy<IPublishNuGet>(x => x.Publish)
        .Executes(() => { /* */ });
}
{% endhighlight %}

We've added a `Clean` target that will be executed as a dependency for `Publish`, but also _before_ the `Restore` target. Another target `ValidatePackages` is executed right before `Publish`, while `Announce` is triggered by `Publish` and will execute right after.

## Provoking Diamond Problems

So far, our build implementation doesn't really justify the demand for interfaces yet. We could have also used a **hierarchy of base classes** along the way. However, for our next step, we'll actually need the flexibility of [multiple inheritance](https://en.wikipedia.org/wiki/Multiple_inheritance). Suppose we want to extract the `Announce` target in its own interface and also make it more general purpose, so that it can be used when publishing a website or a mobile application. Think of it as following the [single-responsibility principle](https://en.wikipedia.org/wiki/Single-responsibility_principle) with build components. Let's introduce the `IAnnounce` interface:

{% highlight csharp linenos %}
interface IAnnounce
{
    Target Announce => _ => _
        .Executes(() => { /* */ });

    string Message { get; }
}
{% endhighlight %}

Firstly, we've introduced a `Message` property, so that every individual build class can specify, what message should be sent. Secondly, we've removed `IPublishNuGet` as a base interface and `Publish` as a trigger dependency. This information goes into the build class as well:

{% highlight csharp linenos %}
class Build : NukeBuild, IBuild, IPublishNuGet, IAnnounce
{
    public static void Main() => Execute<Build>();

    Target IAnnounce.Announce => _ => _
        .Inherit<IAnnounce>(x => x.Announce)
        .TriggeredBy(Publish);

    string IAnnounce.Message => "New Release Out Now!";
}
{% endhighlight %}

Here, we're redefining the `Announce` target and calling `Inherit<T>` to let it derive from the default target definition in the `IAnnounce` interface. The target will keep its original actions, but will be extended with a trigger from `Publish`. For better understanding, we can always create a visual representation of our dependency graph using `nuke --plan`:

![Build Graph](/assets/images/2020-06-05-reusable-build-components-with-interface-default-implementations/plan.png){:width="700px" .shadow}

A similar trick as with default implementations and `Inherit`, we can use in a hierarchy of build classes that defines `virtual` targets:

{% highlight csharp linenos %}
class BaseBuild : NukeBuild
{
    public virtual Target ComplexTarget => _ => _
        .Executes(() => { /* complex logic */ };
}
class Build : BaseBuild
{
    public static void Main() => Execute<Build>();

    public override Target ComplexTarget => _ => _
        .Base()
        .Requires(() => AdditionalParameter)
        .Executes(() => { /* additional logic */ };
}
{% endhighlight %}

Here, we're actually overriding an existing target, and call `Base` to execute the original target definition. This essentially mimics a `base.<Method>` call in the realm of target definitions.

<!--
## Calling Members Non-Virtual

For the new fluent methods `Base` ad `Inherit`, there was one interesting issue that we had to solve. In order to let an overridden target inherit from its base declaration (or a re-implemented target with default implementation), we needed to **make non-virtual calls using reflection**. This wasn't exactly easy, since even when reflecting on a member through its declaring type, it will follow the principle of [polymorphism](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/polymorphism) and call the member virtual. With a [little bit of help from Stack Overflow](https://stackoverflow.com/a/14415506/568266), we ended up implementing the following generic method:

{% highlight csharp linenos %}
public static TResult GetValueNonVirtual<TResult>(this MemberInfo member, object obj, params object[] arguments)
{
    ControlFlow.Assert(member is PropertyInfo || member is MethodInfo, "member is PropertyInfo || member is MethodInfo");
    var method = member is PropertyInfo property
        ? property.GetMethod
        : (MethodInfo) member;

    var funcType = Expression.GetFuncType(method.GetParameters().Select(x => x.ParameterType)
        .Concat(method.ReturnType).ToArray());
    var functionPointer = method.NotNull("method != null").MethodHandle.GetFunctionPointer();
    var nonVirtualDelegate = (Delegate) Activator.CreateInstance(funcType, obj, functionPointer)
        .NotNull("nonVirtualDelegate != null");

    return (TResult) nonVirtualDelegate.DynamicInvoke(arguments);
}
{% endhighlight %}-->

## Conclusion

Default implementations in interfaces will probably not make it to every codebase out there. For the purpose of defining reusable build components and integrating them into specific build pipelines though, they're the perfect fit. We're no longer coupled to use a strict hierarchy of build targets, but instead we can compose our build from multiple independent targets and connect them as needed. Most importantly, we reduce the maintenance cost when different projects need to be built the same way.
