{% highlight csharp linenos %}
[Parameter] readonly bool UseSSH;

AbsolutePath ExternalRepositoriesDirectory => RootDirectory / "external";
AbsolutePath ExternalRepositoriesFile => ExternalRepositoriesDirectory / "repositories.yml";

Target CheckoutExternalRepositories => _ => _
    .Executes(async () =>
    {
        var repositories = YamlDeserializeFromFile<string[]>(ExternalRepositoriesFile)
            .Select(x => GitRepository.FromUrl(x));
        
        foreach (var repository in repositories)
        {
            var repositoryDirectory = ExternalRepositoriesDirectory / repository.GetGitHubName();
            var origin = UseSSH ? repository.SshUrl : repository.HttpsUrl;
        
            if (!Directory.Exists(repositoryDirectory))
                Git($"clone {origin} {repositoryDirectory} --branch {await repository.GetDefaultBranch()} --progress");
            else
            {
                SuppressErrors(() => Git($"remote add origin {origin}", repositoryDirectory));
                Git($"remote set-url origin {origin}", repositoryDirectory);
            }
        }
    });
{% endhighlight %}