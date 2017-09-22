---
layout: post
title: Slack Outgoing Webhooks
subtitle: "Receive webhooks from Slack.com and respond with a message if you want"
published: true
---

## The aim of the game
I built the [Slack.Webhooks](http://github.com/nerdfury/Slack.Webhooks) library so that you could post messages into your Slack account easily, but I wanted to demonstrate how to use it to receive message from slack (and optionally respond with a message).

## Required packages for this demo

- [Slack.Webhooks](http://github.com/nerdfury/Slack.Webhooks)
- [NancyFX](http://github.com/nancyfx/nancy) I chose to use self hosted nancy but feel free to use MVC, WebForms or whatever.

## The process

Start a new console app (this could just as easily have been a empty asp.net application)

### Install dependencies

```
PM> Install-Package Slack.Webhooks
PM> Install-Package Nancy.Hosting.Self
```

### Setup nancy

Listen for requests on port 1234

```csharp
static void Main()
{
    JsConfig.EmitLowercaseUnderscoreNames = true;
    JsConfig.IncludeNullValues = false;
    JsConfig.PropertyConvention = JsonPropertyConvention.Lenient;
    using (var host = new NancyHost(new Uri("http://localhost:1234")))
    {
        host.Start();
        Console.ReadLine();
    }
}
```

Create a module to handle requests

```csharp
public class WebhookModule : NancyModule
{
    public WebhookModule()
    {
        Post["/"] = _ =>
            {
                var model = this.Bind<HookMessage>();
                var message = string.Empty;
                if (model.Text.ToLower().StartsWith("testbot: hello"))
                    message = string.Format("@{0} Hello", model.UserName);
                if(!string.IsNullOrWhiteSpace(message))
                    return new SlackMessage {Text = message, Username = "testbot", IconEmoji = Emoji.Ghost};
                return null;
            };
    }
}
```

and the Model

```csharp
public class HookMessage
{
    public string Token { get; set; }
    public string TeamId { get; set; }
    public string ChannelId { get; set; }
    public string ChannelName { get; set; }
    public string UserId { get; set; }
    public string UserName { get; set; }
    public string Text { get; set; }
    public string TriggerWord { get; set; }
}
```

I may move this model into the library, but it requires a bit of faffing to deserialize the payload from Slack, so I may just keep it seperate for now.

They payload from Slack looks a little like this:

```
token=XXXXXXXXXXXXXXXXXX
team_id=T0001
team_domain=example
channel_id=C2147483705
channel_name=test
timestamp=1355517523.000005
user_id=U2147483697
user_name=Steve
text=googlebot: What is the air-speed velocity of an unladen swallow?
trigger_word=googlebot:
```

in order to bind the 

```
channel_id
```

style field names to 

```
ChannelId
```

model properties you can create a new implementation of Nancy's IFieldNameConverter interface. I did this using the String.ToTitleCase extension method from ServiceStack.Text in order to get the result I wanted.

```csharp
public class TitleCaseFieldNameConverter : IFieldNameConverter
{
    public string Convert(string fieldName)
    {
        return fieldName.ToTitleCase();
    }
}
```

And then register this new type in a new nancy bootstrapper

```csharp
public class Bootstrapper : DefaultNancyBootstrapper
{
    protected override void ApplicationStartup(Nancy.TinyIoc.TinyIoCContainer container, Nancy.Bootstrapper.IPipelines pipelines)
    {
        container.Register<IFieldNameConverter, TitleCaseFieldNameConverter>();
        base.ApplicationStartup(container, pipelines);
    }
}
```

At this point the app should be good to go, all that is left to do is configure a new Outgoing webhook integration to deliver messages containeing trigger word(s) (e.g. testbot:) to your new application.

If you want to do this on your local dev machine I can't recommend [ngrok](https://ngrok.com/) enough for routing external traffic without having to mess about with routers, dns and firewalls too much.
