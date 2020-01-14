{% highlight csharp linenos %}
Target GenerateGlobalSolution => _ => _
    .DependsOn(CheckoutExternalRepositories)
    .Executes(() =>
    {
        var global = CreateSolution(
            solutions: new[] { Solution }.Concat(ExternalSolutions),
            folderNameProvider: x => x == Solution ? null : x.Name);
        global.SaveAs(GlobalSolution);
    });
{% endhighlight %}