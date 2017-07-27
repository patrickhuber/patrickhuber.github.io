---
layout: default
title: Avoid Secrets in DotNet Core Tests.
---

# Motivation

I like the idea of removing user secrets from my applications during development as pointed out in the following [microsoft documentation](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets). This techneque works great for web application development, but what about tests? I often will create integration tests that contain sensetive information to test out connecting to an external service, so why not do the same?

# Setup


## Create the project 

First, Create a .NET core unit test project. 

## Install NuGet Packages

Next, add the following nuget package

```
Install-Package "Microsoft.Exensions.Configuration.UserSecrets"
```

## Add the UserSecretId

Similar to the setup in [this article](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets), right click on the project and add the following line. 

```Xml
<PropertyGroup>
    <UserSecretsId>{test project identifier}-{guid}</UserSecretsId>
</PropertyGroup>
```

Replace the {test project identifier} with the name of your test project. This name is just to keep the secret store unique, it can really be anything. 

Replace the {guid} with a unique identifier. You can generate one here: https://www.guidgenerator.com/online-guid-generator.aspx

Here is an example:

```Xml
<PropertyGroup>
    <UserSecretsId>myproj-integration-tests-bf4e2e40-30e1-42e6-87ca-9f88390a414c</UserSecretsId>
</PropertyGroup>
```

## Add the tooling reference

Also in the same csproj file add the following reference

```Xml
<ItemGroup>
    <DotNetCliToolReference Include="Microsoft.Extensions.SecretManager.Tools" Version="1.0.1" />
</ItemGroup>
```

## Create your first secret

Open the command prompt to the directory of the .csproj file. Add a secret by running:

```
dotnet user-secrets set ApiUserName someguy
dotnet user-secrets set ApiPassword abc123
```

## Setup the test to read the secret

Now that we have a secret in the secret store, we can read it using the configuration libraries. I'm using MSTest but the same should apply for any testing framwork. 

```CSharp
using Microsoft.Extensions.Configuration;
using Microsoft.VisualStudio.TestTools.UnitTesting;
using System.Collections.Generic;
using System.IO;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text;
using System.Threading.Tasks;

namespace MyAwesomeLibrary.Tests.Integration
{
    [TestClass]
    public class HttpClientTests
    {
        IConfiguration Configuration { get; set; }

        public HttpClientTests()
        {
            // the type specified here is just so the secrets library can 
            // find the UserSecretId we added in the csproj file
            var builder = new ConfigurationBuilder()
                .AddUserSecrets<HttpClientTests>();

            Configuration = builder.Build();
        }

        [TestMethod]
        public async Task HttpClientShouldAllowBasicAuthentication()
        {            
            var username = Configuration["ApiUsername"];
            var password = Configuration["ApiPassword"];

            // this test url just validates the parameters you pass in the uri,
            // normally you wouldn't do this
            var requestUri = $"http://httpbin.org/basic-auth/{username}/{password}";

            HttpClient client = new HttpClient();
            using (var request = new HttpRequestMessage(HttpMethod.Get, requestUri))
            {
                var credential = $"{username}:{password}";
                var credentialBytes = Encoding.ASCII.GetBytes(credential);
                var credentialBase64 = Convert.ToBase64String(credentialBytes);

                request.Headers.Authorization = new AuthenticationHeaderValue("Basic", credentialBase64);

                var response = await client.SendAsync(request);

                Assert.AreEqual(200, (int)response.StatusCode);
            }
        }
    }
}
```