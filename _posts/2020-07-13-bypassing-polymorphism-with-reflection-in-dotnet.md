---
title: Non-Virtual Invocations in .NET
tags:
- .NET
- C#
mode: immersive
header:
  theme: dark
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(0, 0, 100 , .5), rgba(60, 34, 60, .4))'
    src: assets/images/2020-07-13-bypassing-polymorphism-with-reflection-in-dotnet/cover.jpg
twitter_card: assets/images/2020-07-13-bypassing-polymorphism-with-reflection-in-dotnet/thumbnail.jpeg
image_author_name: JJ Shev
image_author_link: https://unsplash.com/@skjev5280
---

While implementing [support for interface default implementations](2020-06-05-reusable-build-components-with-interface-default-implementations.md) we've added the new fluent methods `Base` and `Inherit`. In order to let an overridden target inherit from its base declaration (or a re-implemented target with default implementation), we needed to **make non-virtual calls using reflection**. This wasn't exactly easy, since even when reflecting on a member through its declaring type, it will follow the principle of [polymorphism](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/polymorphism) and call the member virtual. With [a bit of Stack Overflnugetow](https://stackoverflow.com/a/14415506/568266), we ended up implementing the following generic method:

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
{% endhighlight %}

Using the `GetValueNonVirtual` method, we can use any property or method reflected from a base type, and **bypass their overrides**:

{% highlight csharp linenos %}
var method = typeof(object).GetMethod("ToString").Single();
var value = method.GetValueNonVirtual(dto);
{% endhighlight %}

I'm glad that we have reflection! 🤓
