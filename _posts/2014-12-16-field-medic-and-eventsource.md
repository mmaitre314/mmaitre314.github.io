---
layout: post
title: Field Medic and EventSource
comments: true
---

A previous [blog]({{ site.url }}/2014-12-16-field-medic-and-eventsource.md) covered how to use [Field Medic](http://www.windowsphone.com/en-us/store/app/field-medic/73c58570-d5a7-46f8-b1b2-2a90024fc29c) to capture event logs on Windows Phone. This is only half of the story though: something needs to generate those events in the first place. This is where [EventSource](http://blogs.msdn.com/b/vancem/archive/2012/08/13/windows-high-speed-logging-etw-in-c-net-using-system-diagnostics-tracing-eventsource.aspx) comes into play. It enables quickly adding performance events (markers for operation beginning and end, etc.) and unstructured traces (aka printf) to .NET apps:

{% highlight csharp %}
Log.Events.ProcessImageStart();
Log.Write("About to process file {0}", filename);
await ProcessImageAsync(filename);
Log.Events.ProcessImageStop();
{% endhighlight %}

![WPA]({{ site.url }}/images/2014-12-16-WPA.PNG)

EventSource does so with fairly little supporting code:

{% highlight csharp %}
using System.Diagnostics.Tracing;

[EventSource(Name = "Company-Product-Component")]
sealed class Log : EventSource
{
    public static Log Events = new Log();

    // Unstructured traces

    [NonEvent]
    public static void Write(string message) 
    { 
        Events.Message(message); 
    }

    [NonEvent]
    public static void Write(string format, params object[] args) 
    {
        if (Events.IsEnabled())
        {
            Events.Message(String.Format(format, args));
        }
    }

    [Event(1, Level = EventLevel.Verbose)]
    private void Message(string message) { Events.WriteEvent(1, message); }

    // Performance markers

    public class Tasks
    {
        public const EventTask ProcessImage = (EventTask)1;
    }

    [Event(2, Task = Tasks.ProcessImage, Opcode = EventOpcode.Start, Level = EventLevel.Informational)]
    public void ProcessImageStart() { Events.WriteEvent(2); }

    [Event(3, Task = Tasks.ProcessImage, Opcode = EventOpcode.Stop, Level = EventLevel.Informational)]
    public void ProcessImageStop() { Events.WriteEvent(3); }
}
{% endhighlight %}

Field Medic supports capturing events generated by EventSource using typical [WPRP](http://msdn.microsoft.com/en-us/library/windows/hardware/hh448223.aspx) config files, with one caveat: the event provider name must be prefixed by '*'.

{% highlight xml %}
<EventProvider Id="EventProvider_Company_Product_Component" Name="*Company-Product-Component" Level="5"/>
{% endhighlight %}

See [Component.wprp]({{ site.url }}/download/Component.wprp) for a full WPRP config file.