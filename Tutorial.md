---
Title: "Elsa Workflow Tutorial: Simple Document Approval"
Description: Learn how to configure workflow using Elsa Workflow library programatically and via Visual Designer.
Date: 10/11/2021

---

# Tutorial: Elsa Workflow Simple Document Approval

This tutorial shows how to configure workflow by using Elsa Workflow library both programmatically and using Visual Designer.

In this tutorial, you learn how to:

> * Install Elsa package and other dependancy
> * Setup Elsa Core and Server
> * Link Dashboard and Visual Designer to Elsa server
> * Configure Elsa workflow programmatically (C#)
> * Use docker for SMTP testing
> * Trigger an API from Postman
> * Configure Elsa workflow using Visual Designer

To understand the concept of Elsa workflow, you can read about it here https://elsa-workflows.github.io/elsa-core/docs/next/concepts/concepts-workflows

## Create new project

Create a new, empty ASP.NET Core project called `DocumentApproval.Web`:
```
dotnet new web -n "DocumentApproval.Web"
```

CD into the created project folder:
```
cd DocumentApproval.Web
```

And add the following packages:
```
dotnet add package Elsa
dotnet add package Elsa.Activities.Email
dotnet add package Elsa.Activities.Http
dotnet add package Elsa.Activities.Temporal.Quartz
dotnet add package Elsa.Persistence.EntityFramework.Sqlite
dotnet add package Elsa.Server.Api
dotnet add package Elsa.Designer.Components.Web
```
This is the package for:
- Elsa core package
- Elsa package for send/receive email activity
- Elsa package for HTTP realated activity
- Elsa package for using time-based activities eg:`Timer` activity
- Elsa package to use SQLite Persistence provider
- Elsa package for using Elsa designer dashboard

## Startup
Open `Startup.cs` and replace its contents with the following:
```
using Elsa;
using Elsa.Persistence.EntityFramework.Core.Extensions;
using Elsa.Persistence.EntityFramework.Sqlite;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

namespace DocumentApproval.Web
{
    public class Startup
    {
        public Startup(IWebHostEnvironment environment, IConfiguration configuration)
        {
            Environment = environment;
            Configuration = configuration;
        }

        private IWebHostEnvironment Environment { get; }
        private IConfiguration Configuration { get; }

        public void ConfigureServices(IServiceCollection services)
        {
            var elsaSection = Configuration.GetSection("Elsa");
            
            services
            
                .AddElsa(elsa => elsa
                    .UseEntityFrameworkPersistence(ef => ef.UseSqlite())
                    .AddConsoleActivities()
                    .AddHttpActivities(elsaSection.GetSection("Server").Bind)
                    .AddEmailActivities(elsaSection.GetSection("Smtp").Bind)
                    .AddQuartzTemporalActivities()
                    .AddWorkflowsFrom<Startup>()
                );

            services.AddElsaApiEndpoints();
            services.AddRazorPages();
        }

        public void Configure(IApplicationBuilder app)
        {
            app
                .UseStaticFiles()
                .UseHttpActivities()
                .UseRouting()
                .UseEndpoints(endpoints =>
                {
                    endpoints.MapControllers();
                    endpoints.MapFallbackToPage("/_Host");
                });
        }
    }
}
```

*Notice that we're accessing a configuration section called "Elsa". We then use this section to retrieve sub-sections called "Server" and "Smtp". Let's 
update `appsettings.json` with these sections next:*

## Appsettings.json
Open `appsettings.json` and add the following section:
```
{
  "Elsa": {
    "Server": {
      "BaseUrl": "https://localhost:5001"
    },
    "Smtp": {
      "Host": "localhost",
      "Port": "2525",
      "DefaultSender": "noreply@acme.com"
    }
  }
}
```

*The reason we are setting a "base URL" is because the HTTP activities library provides an absolute URL provider that can be used by activities and workflow expressions. Since this absolute URL 
provider can be used outside the context of an actual HTTP request (for instance, when a timer event occurs), we cannot rely on e.g. `IHttpContextAccessor`, since there won't be any HTTP context.*

## _Host.cshtml

Notice that the application will always fall back to serve the `_Host.cshtml` page, which we will create next.

- Create a new folder called `Pages`.
- Inside `Pages`, create a new file called `_Host.cshtml`. This will render and include the necessary HTML, scripts and styles for the Elsa Dashboard UI.
- Add the following content to `_Host.cshtml`:

```
@page "/"
@{
    var serverUrl = $"{Request.Scheme}://{Request.Host}";
}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <title>Elsa Workflows</title>
    <link rel="icon" type="image/png" sizes="32x32" href="/_content/Elsa.Designer.Components.Web/elsa-workflows-studio/assets/images/favicon-32x32.png">
    <link rel="icon" type="image/png" sizes="16x16" href="/_content/Elsa.Designer.Components.Web/elsa-workflows-studio/assets/images/favicon-16x16.png">
    <link rel="stylesheet" href="/_content/Elsa.Designer.Components.Web/elsa-workflows-studio/assets/fonts/inter/inter.css">
    <link rel="stylesheet" href="/_content/Elsa.Designer.Components.Web/elsa-workflows-studio/elsa-workflows-studio.css">
    <script src="/_content/Elsa.Designer.Components.Web/monaco-editor/min/vs/loader.js"></script>
    <script type="module" src="/_content/Elsa.Designer.Components.Web/elsa-workflows-studio/elsa-workflows-studio.esm.js"></script>
</head>
<body>
<elsa-studio-root server-url="@serverUrl" monaco-lib-path="_content/Elsa.Designer.Components.Web/monaco-editor/min">
    <elsa-studio-dashboard></elsa-studio-dashboard>
</elsa-studio-root>
</body>
</html>
```

At this point, you should have a functioning Elsa Server application that can execute workflows and serve the Elsa Dashboard.

## Document Approval Workflow: Builder API

Next we will programatically build the workflow. Some of the keypoints are:
- Create a new workflow named DocumentApprovalWorkflow
- Add activity sequences of the workflow
- Fork workflow to 3 branches, `Approve`, `Reject` and `Remind`
- Execute `Signal received` and `Timer` activity 

Create a new file called `DocumentApprovalWorkflow.cs` and add the following contents:
```
using System.Net;
using System.Net.Http;
using Elsa.Activities.ControlFlow;
using Elsa.Activities.Email;
using Elsa.Activities.Http;
using Elsa.Activities.Http.Extensions;
using Elsa.Activities.Http.Models;
using Elsa.Activities.Primitives;
using Elsa.Activities.Temporal;
using Elsa.Builders;
using NodaTime;

namespace DocumentApproval.Web
{
    public class DocumentApprovalWorkflow : IWorkflow
    {
        public void Build(IWorkflowBuilder builder)
        {
            builder
                .WithDisplayName("Document Approval Workflow")
                .HttpEndpoint(activity => activity
                    .WithPath("/v1/documents")
                    .WithMethod(HttpMethod.Post.Method)
                    .WithReadContent())
                .SetVariable("Document", context => context.GetInput<HttpRequestModel>()!.Body)
                .SendEmail(activity => activity
                    .WithSender("workflow@acme.com")
                    .WithRecipient("josh@acme.com")
                    .WithSubject(context => $"Document received from {context.GetVariable<dynamic>("Document")!.Author.Name}")
                    .WithBody(context =>
                    {
                        var document = context.GetVariable<dynamic>("Document")!;
                        var author = document!.Author;
                        return $"Document from {author.Name} received for review.<br><a href=\"{context.GenerateSignalUrl("Approve")}\">Approve</a> or <a href=\"{context.GenerateSignalUrl("Reject")}\">Reject</a>";
                    }))
                .WriteHttpResponse(
                    HttpStatusCode.OK,
                    "<h1>Request for Approval Sent</h1><p>Your document has been received and will be reviewed shortly.</p>",
                    "text/html")
                .Then<Fork>(activity => activity.WithBranches("Approve", "Reject", "Remind"), fork =>
                {
                    fork
                        .When("Approve")
                        .SignalReceived("Approve")
                        .SendEmail(activity => activity
                            .WithSender("workflow@acme.com")
                            .WithRecipient(context => context.GetVariable<dynamic>("Document")!.Author.Email)
                            .WithSubject(context => $"Document {context.GetVariable<dynamic>("Document")!.Id} Approved!")
                            .WithBody(context => $"Great job {context.GetVariable<dynamic>("Document")!.Author.Name}, that document is perfect."))
                        .ThenNamed("Join");

                    fork
                        .When("Reject")
                        .SignalReceived("Reject")
                        .SendEmail(activity => activity
                            .WithSender("workflow@acme.com")
                            .WithRecipient(context => context.GetVariable<dynamic>("Document")!.Author.Email)
                            .WithSubject(context => $"Document {context.GetVariable<dynamic>("Document")!.Id} Rejected")
                            .WithBody(context => $"Nice try {context.GetVariable<dynamic>("Document")!.Author.Name}, but that document needs work."))
                        .ThenNamed("Join");

                    fork
                        .When("Remind")
                        .Timer(Duration.FromSeconds(60)).WithName("Reminder")
                        .SendEmail(activity => activity
                                .WithSender("workflow@acme.com")
                                .WithRecipient("josh@acme.com")
                                .WithSubject(context => $"{context.GetVariable<dynamic>("Document")!.Author.Name} is waiting for your review!")
                                .WithBody(context =>
                                    $"Don't forget to review document {context.GetVariable<dynamic>("Document")!.Id}.<br><a href=\"{context.GenerateSignalUrl("Approve")}\">Approve</a> or <a href=\"{context.GenerateSignalUrl("Reject")}\">Reject</a>"))
                            .ThenNamed("Reminder");
                })
                .Add<Join>(join => join.WithMode(Join.JoinMode.WaitAny)).WithName("Join")
                .WriteHttpResponse(HttpStatusCode.OK, "Thanks for the hard work!", "text/html");
        }
    }
}
```

Before we try out the workflow, let's setup an SMTP host. The easiest way to do so is by running `Smtp4Dev` using Docker:
```
docker run -p 3000:80 -p 2525:25 rnwood/smtp4dev:linux-amd64-3.1.0-ci0856
```

When Smtp4Dev has started, you'll be able to navigate to its dashboard at http://localhost:3000/ and inspect the emails the workflow will send.

## First Run

Run the project and send the following HTTP request (using e.g. Postman):
```
POST /v1/documents HTTP/1.1
Host: localhost:5001
Content-Type: application/json

{
    "Id": "3",
    "Author": {
        "Name": "John",
        "Email": "john@gmail.com"
    },
    "Body": "This is sample document."
}
```
The response should look like this:
```
<h1>Request for Approval Sent</h1>
<p>Your document has been received and will be reviewed shortly.</p>
```
When you launch the `Smtp4Dev` dashboard, you should be seeing this:

![smptp1](https://user-images.githubusercontent.com/93765959/141693214-86a60894-3d3a-4cf2-b895-92660b1c3252.JPG)

And after around every 50 seconds, reminder email messages will come in:

![smtp2](https://user-images.githubusercontent.com/93765959/141693232-b6f03560-81fc-41b5-8329-5571759da29f.JPG)

This will continue until you either click the `Approve` or `Reject` link.

## Long Running Workflows

Due to the fact that the workflow is blocked on `Timer` and `SignalReceived` activities, the workflow will be persisted. 
In this demo, we use the Entity Framework Core persistence provider and Sqlite as the database engine.
![suspended](https://user-images.githubusercontent.com/93765959/141693291-9c013703-7223-45db-8ede-ddf0ab3ecfc6.JPG)

This means that even when you stop the application and then start it again, Elsa will pick up where it has left off: the Timer activity will keep resuming the workflow. 
Similarly, clicking the Accept or Reject link from the email messages will resume the workflow too.

![smtp3](https://user-images.githubusercontent.com/93765959/141712864-6952b8bd-1668-4442-857f-76f8a820985e.JPG)

Go ahead and click either the Approve or Reject link. This should trigger a final email message, clear the Timer activity (due to the Join activity), 
and finish the workflow.

This is the outcomes when use click the Approve link.

- HTTP response
![Response](https://user-images.githubusercontent.com/93765959/141713026-ce077c57-a814-4f21-a54a-5b2f6d06d072.JPG)

- Final email
![smptp4](https://user-images.githubusercontent.com/93765959/141714604-d1f3a470-80cc-45d4-8793-30be8c1cd919.JPG)

- Workflow instance finished
![finished](https://user-images.githubusercontent.com/93765959/141714980-6fbfdb95-f1b0-4423-9a9b-cb7fb62520ff.JPG)

Alternatively, we can create another workflow to send the Respond signal to the 1st workflow, instead of clicking on the link in email. 
This time we will configure the workflow using visual Designer.

## Create Workflow using Visual Designer

With the Elsa Dashboard in front of you, navigate to the `Workflow Definitions` page and click the `Create button`. You should now see an empty canvas with just 
a `Start` button and a cog wheel to configure workflow settings.

![create wf](https://user-images.githubusercontent.com/93765959/141716114-4dc0a05e-8c11-46ce-9286-2ab1c3d7b799.JPG)

Let's do that first: click the cog wheel and specify the following:
```
Name: SendApproveSignal
Display Name: Send Approve Signal
Click Save.
```

### HTTP Endpoint
Now click the Start button and look for the `HTTP Endpoint activity` and select it. 
Configure it with the following settings:
```
Path: /send-approve-signal
Methods: GET
Read Content: true (checked)
```
![http2](https://user-images.githubusercontent.com/93765959/141716783-8c979490-71cc-4718-ae86-ca795e2b9127.JPG)

### Send Signal
Click the `Done` outcome button on the previous activity and look for the `Send Signal activity` and configure it as follows:
```
Signal: Approve
```
![send signal](https://user-images.githubusercontent.com/93765959/141716865-4059b323-d74e-400c-a11a-2a7d63acf8ba.JPG)


### HTTP Response: Document Received
Click the `Done` outcome button on the previous activity and look for the `HTTP Response activity` and configure it as follows:
```
Status Code: OK
Content: <h1>Request for Approval Sent</h1><p>Your document has been received and will be reviewed shortly.</p>
Content Type: text/html
```
![http6](https://user-images.githubusercontent.com/93765959/141716829-02b15942-0474-4d8c-847e-4494e7416834.JPG)

## Second Run
Make sure to publish your changes and then issue the following HTTP request:
```
GET /send-approve-signal
Host: localhost:5001
Content-Type: application/json
```

As you'll see, it works exactly the same as with the programmatic workflow created earlier.

## Error Handling

As for now,  the implementation for error handling is still in implemtation. But from the forum, the idea is maybe we can leave error handling up to each 
activity itself: if an activity can fault, it could simply do a try/catch block themselves, and return the appropriate outcome (e.g. "Faulted").
