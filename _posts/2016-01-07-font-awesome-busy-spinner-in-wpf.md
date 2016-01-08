---
layout: post
title: A Font Awesome Busy Spinner in WPF
comments: true
---

[Font Awesome](http://fontawesome.io) is a great source of icons for Web Apps, and it happens to work just as well in WPF Desktop apps. In particular
[fa-spinner](http://fortawesome.github.io/Font-Awesome/icon/spinner/) can be combined with WPF animations to create a simple busy spinner:

![Demo]({{ site.url }}/images/spinner.jpg)

Font Awesome's CSS style file points to a TTF font file that can be directly copied into the Visual Studio project. At the time of writing, it is available 
[here](https://maxcdn.bootstrapcdn.com/font-awesome/4.5.0/fonts/fontawesome-webfont.ttf).

fa-spinner has unicode f110 and animating it is a matter of applying a RotateTransform to a TextBlock and defining an infinite animation as part of a storyboard:

{% highlight xml %}
<Window.Resources>
    <Storyboard x:Key="WaitStoryboard">
        <DoubleAnimation
        Storyboard.TargetName="Wait"
        Storyboard.TargetProperty="(TextBlock.RenderTransform).(RotateTransform.Angle)"
        From="0"
        To="360"
        Duration="0:0:2"
        RepeatBehavior="Forever" />
    </Storyboard>
</Window.Resources>

<TextBlock Name="Wait" FontFamily="Fonts/#FontAwesome" FontSize="50" Text="&#xf110;" RenderTransformOrigin="0.5, 0.5">
    <TextBlock.RenderTransform>
        <RotateTransform Angle="0" />
    </TextBlock.RenderTransform>
</TextBlock>
{% endhighlight %}

The animation can be started (and stopped) on demand by the C# code behind:

{% highlight csharp %}
((Storyboard)FindResource("WaitStoryboard")).Begin();
{% endhighlight %}

See this [GitHub repo](https://github.com/mmaitre314/FontAwesomeWpfSpinner) for the full code.

If you are planning on making more heavy use of Font Awesome, you may consider the [FontAwesome.WPF](https://github.com/charri/Font-Awesome-WPF) NuGet package
which simplifies using those icons.

If you are looking for different styles of busy indicators, a few alternatives are available:

- [Telerik Busy Indicator](http://www.telerik.com/products/wpf/busyindicator.aspx)
- [BizzySpinner2](https://foredecker.wordpress.com/2010/01/11/bizzyspinner-2-a-wpf-spinning-busy-state-indicator-with-source/)
- [Better WPF Circular Progress Bar](http://www.codeproject.com/Articles/49853/Better-WPF-Circular-Progress-Bar)
