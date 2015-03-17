---
layout: post
title: Creating Debugger Visualizers for Windows/Windows Phone Store apps
comments: true
---

When debugging image-processing code, being able to view bitmaps directly in the debugger greatly helps understand what the code is actually doing.

![BitmapVisualizer](http://matthieumaitre.info/images/BitmapVisualizer.png)

[Image Watch](https://visualstudiogallery.msdn.microsoft.com/e682d542-7ef3-402c-b857-bbfba714f78d) is the reference debugger extension when it comes to debugging C++ apps. For C# Desktop apps Visual Studio [Visualizer APIs](https://msdn.microsoft.com/en-us/library/zayyhzts.aspx) help build such debugger extensions. Those APIs do not currently support Store apps, but with a bit more work similar visualizers can be created to debug those apps too.

This blog gives an overview of the steps required to create such visualizers. For details see the full code in this [GitHub repo](https://github.com/mmaitre314/BitmapVisualizer).

# Create the VS package

First, we need to create a VSIX package which will let users install the debugger extension. Install the [Visual Studio SDK](https://msdn.microsoft.com/en-us/library/bb166441.aspx) and create a new solution using the solution template under 'Templates > Extensibility > Visual Studio Package'. In the VSIX manifest, add an asset of type Microsoft.VisualStudio.MefComponent using the main project in the solution as source. This tells Visual Studio that we want to [extend the editor](https://msdn.microsoft.com/en-us/library/dd885013.aspx).

Also add a few references which will help interact with the editor:

- Microsoft.VisualStudio.CoreUtility
- Microsoft.VisualStudio.Text.Data
- Microsoft.VisualStudio.Text.UI
- Microsoft.VisualStudio.Text.UI.Wpf
- PresentationCore
- PresentationFramework
- System.ComponentModel.Composition
- System.Xaml
- WindowsBase

# Create the adornment UI

Next we need some UI to display the bitmaps. A basic XAML user control with an `<Image>` element will do:

{% highlight xml %}
<UserControl x:Class="MMaitre.BitmapVisualizer.BitmapVisualizerControl"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             Cursor="Arrow">
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <Border Grid.Row="0" Background="Black" >
            <Image Name="Preview" Width="300" Stretch="Uniform" Margin="2"/>
        </Border>
        <Polygon Grid.Row="1" Points="0,0 20,0, 10,10" StrokeThickness="0" Fill="Black" VerticalAlignment="Bottom" HorizontalAlignment="Left" Margin="20, 0, 0, 0"/>
    </Grid>
</UserControl>
{% endhighlight %}
        
{% highlight csharp %}
public partial class BitmapVisualizerControl : UserControl
{
    public BitmapVisualizerControl()
    {
        InitializeComponent();
    }
}
{% endhighlight %}

# Register the adornment UI

The bitmaps will be displayed when the mouse hovers on a variable, so we need the editor to notify us when that happens.

This is done in two parts. First a visualizer factory is registered with the editor using a set of attributes. The factory also implements [IWpfTextViewCreationListener](https://msdn.microsoft.com/en-us/library/microsoft.visualstudio.text.editor.iwpftextviewcreationlistener.aspx) to give out visualizer objects -- the second part -- to the editor whenever it needs them.

{% highlight csharp %}
[Export(typeof(IWpfTextViewCreationListener))]
[ContentType("code")]
[TextViewRole(PredefinedTextViewRoles.Document)]
internal sealed class BitmapVisualizerFactory : IWpfTextViewCreationListener
{
    [Export(typeof(AdornmentLayerDefinition))]
    [Name(BitmapVisualizer.AdornmentLayerName)]
    [Order(After = PredefinedAdornmentLayers.Text)]
    internal AdornmentLayerDefinition EditorAdornmentLayer;

    private DTE2 m_dte;

    BitmapVisualizerFactory()
    {
        m_dte = (DTE2)ServiceProvider.GlobalProvider.GetService(typeof(DTE));
    }

    public void TextViewCreated(IWpfTextView view)
    {
        view.Properties.GetOrCreateSingletonProperty<BitmapVisualizer>(() => new BitmapVisualizer(m_dte.Debugger, view));
    }
}
{% endhighlight %}

The visualizer objects register for the `MouseHover` event when they are created.

{% highlight csharp %}
public BitmapVisualizer(EnvDTE.Debugger debugger, IWpfTextView view)
{
    m_debugger = debugger;
    m_view = view;
    m_layer = m_view.GetAdornmentLayer(AdornmentLayerName);
    m_view.MouseHover += OnMouseHover;
}
{% endhighlight %}
 
# Handle mouse hover

The `MouseHover` event fires whether the app is being debugged or not, so the event handler performs an early check and exits if not in debug-break mode.

{% highlight csharp %}
if (m_debugger.CurrentMode != dbgDebugMode.dbgBreakMode)
{
    return;
}
{% endhighlight %}
            
The event handler then looks up the variable under the mouse pointer and starts retrieving basic information from the debuggee. This part requires too much string parsing to copy/paste in a blog, so just refer directly to [DebuggerVariable.FindUnderMousePointer()](https://github.com/mmaitre314/BitmapVisualizer/blob/master/BitmapVisualizer/DebuggerVariable.cs) in the GitHub project.

A quick type check tells whether we should try to retrieve more data from the debuggee. Here we only handle `Bitmap` objects from the [Lumia Imaging SDK](https://msdn.microsoft.com/en-us/library/dn859593.aspx).

{% highlight csharp %}
if (variable.Type != "Lumia.Imaging.Bitmap")
{
    return null;
}
{% endhighlight %}
            
The bulk of data transfer happens next, when the buffer of the bitmap gets copied from the debuggee to the debugger. This part is a bit complex for two reasons: retrieving data is only possible via [EnvDTE.Debugger.GetExpression()](https://msdn.microsoft.com/en-us/library/envdte.debugger.getexpression.aspx) which treats everything as strings, and Store apps represent buffers using [IBuffer](https://msdn.microsoft.com/en-us/library/windows/apps/windows.storage.streams.ibuffer.aspx) instead of `byte[]` which does not give direct access to the data. 

The `DebuggerVariable` class mentioned earlier encapsulates both issues inside its `GetMemberIBuffer()` method: `IBuffer` is converted to `byte[]`, encoded to a base64 `string`, copied to the debugger, and  decoded back to `byte[]`.

{% highlight csharp %}
internal byte[] GetMemberIBuffer(string name)
{
    // Serialize the IBuffer to string and copy from debuggee to debugger
    var expressionText = "Convert.ToBase64String(System.Runtime.InteropServices.WindowsRuntime.WindowsRuntimeBufferExtensions.ToArray(" + this.Name + "." + name + "));";
    var expression = m_debugger.GetExpression(expressionText);
    if (!expression.IsValidValue)
    {
        return null;
    }

    // Deserialize buffer
    var bufferInBase64 = expression.Value.Substring(1, expression.Value.Length - 2); // Remove '"' string quotes at the beginning and the end
    byte[] buffer;
    try
    {
        buffer = Convert.FromBase64String(bufferInBase64);
    }
    catch (FormatException)
    {
        return null;
    }
    return buffer;
}
{% endhighlight %}

Some more metadata about the bitmap (width, height, pitch, color mode) is retrieved from the debuggee and the [WriteableBitmap](https://msdn.microsoft.com/en-us/library/system.windows.media.imaging.writeablebitmap(v=vs.110).aspx) to be displayed in the adornment UI can be created.

{% highlight csharp %}
var bitmap = new WriteableBitmap((int)width, (int)height, 96.0, 96.0, PixelFormats.Bgr32, BitmapPalettes.WebPalette);
bitmap.WritePixels(new Int32Rect(0, 0, bitmap.PixelWidth, bitmap.PixelHeight), buffer, (int)pitch, 0);

var control = new BitmapVisualizerControl();
control.Preview.Source = bitmap;
{% endhighlight %}

# Credits

Special thanks to [Kfir Karmon](https://github.com/kfirkarmon) who actually wrote most of the code.
