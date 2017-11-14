---
layout: post
title: Problems with DesignMode in WinForms
date: 2011-03-22
---

Time for a quick post about a problem that I encountered today at work which should be worth sharing.

I have just started creating a level editor with Windows Forms that uses XNA for most of the rendering. As a starting point I decided to use the [Windows Forms Series 1: Graphics Device](http://create.msdn.com/en-US/education/catalog/sample/winforms_series_1) sample by Microsoft which I had heard was the most recommended way of doing it.

So, I downloaded the sample, inherited my scene rendering control from the *GraphicsDeviceControl* class, and wrote my own *Initialize* and *Draw* methods. Everything seemed to work okay! However, once I tried adding my control as a child of another control (in this case, a simple panel) the Visual Studio's designer could no longer display my form. Here's what I got:

![Error Message]({{ "/assets/2011-03-22-problems-with-designmode-in-winforms/error2.png" | relative_url }})

The error message hinted that even though I was in designer mode, Visual Studio was *still* calling my *Initialize* method along with all of the content loading that was happening there.

Okay... So I turned to *GraphicDeviceControl's* source code in order to see where Initialize was being called. Here's what I found:

~~~ c#
/// <summary>
/// Initializes the control.
/// </summary>
protected override void OnCreateControl()
{
    // Don't initialize the graphics device if we are running in the designer.
    if (!DesignMode)
    {
        graphicsDeviceService = GraphicsDeviceService.AddRef(Handle, ClientSize.Width, ClientSize.Height);

        // Register the service, so components like ContentManager can find it.
        services.AddService<IGraphicsDeviceService>(graphicsDeviceService);

        // Give derived classes a chance to initialize themselves.
        Initialize();
    }

    base.OnCreateControl();
}
~~~

Strange. *DesignMode* is a property that returns true (or should return true) when the control is being seen from Visual Studio's designer instead of at run-time. Therefore, the condition on line 7 should be enough to ensure that the graphics device is not created and that Initialize is never called while in design mode. Then why?

After googling around for a bit, I ran into this [thread](http://stackoverflow.com/questions/34664/designmode-with-controls/708594#708594) and it turns that the *DesignMode* property is seriously broken to begin with. More specifically, it doesn't work inside the control's constructor (which is not the case here) AND it doesn't work at all if the control is a child of some other control! And this seems to be a known bug of the framework which apparently won't be fixed due to some compatibility problems. *sigh*

Luckily there are some workarounds for it, and since the check was being done outside of the constructor, I went with the recommendation I found on that thread:

1) I added the following method to the *GraphicsDeviceControl* class.

~~~ c#
/// <summary>
/// The DesignMode property does not correctly tell you if
/// you are in design mode.  IsDesignerHosted is a corrected
/// version of that property.
/// (see https://connect.microsoft.com/VisualStudio/feedback/details/553305
/// and http://decav.com/blogs/andre/archive/2007/04/18/1078.aspx )
/// </summary>
public bool IsDesignerHosted
{
    get
    {
        Control ctrl = this;

        while (ctrl != null)
        {
            if ((ctrl.Site != null) && ctrl.Site.DesignMode)
                return true;
                ctrl = ctrl.Parent;
            }
            return false;
        }
}
~~~

2) And I replaced the call to *DesignMode* on *GraphicsDeviceControl.OnCreateControl* with this new property instead.

~~~ c#
// OLD if (!DesignMode)
if(!IsDesignerHosted)
~~~

Now it works correctly!

Although I'm still having some other minor problems, for instance, sometimes after I run my application once, I can no longer select my *GraphicsDeviceControl* on the form without closing it and re-opening it first. That and I've had my control disappear by itself from the form two or three times already... But I'll get there eventually.
