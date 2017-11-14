---
layout: post
title: Limiting 2D Camera Movement with Zoom
date: 2011-09-10
---

On my [2D Camera with Parallax Scrolling in XNA]({{ "/2d-camera-with-parallax-scrolling-in-xna" | relative_url }}) article, one of the topics I described down in the FAQ section was how to limit the range of movement of a 2D camera so that only part of the game world may be seen.

I implemented it as a simple optional rectangle property called Limits which restricted camera movement so that the player could not see anything beyond that region.

However, I considered the simplest of the scenarios where you don't need this functionality AND camera zooming/rotation at the same time. To answer a question posted under that article, this time I'll be presenting a more elaborate implementation that also takes zoom into consideration.

Camera rotation will still be kept out of the equation, since I can't imagine how it could be made to work with sensible results considering that the viewport is rectangular.

## Introduction

I'll start by presenting a little video of what the purpose of this article is. You can download the source code at the end as usual:

<iframe width="420" height="345" src="http://www.youtube.com/embed/tZGl8Utw_cs" allowfullscreen="" frameborder="0"></iframe>

Now onto how it's done. Let us consider the following simple Camera interface for our example:

~~~ c#
public interface Camera
{
    public Viewport Viewport { get; }
    public Vector2 Position { get; set; }
    public float Zoom { get; set; }
    public Matrix ViewMatrix { get; }
    public Rectangle? Limits { set; }
}
~~~

What we'd like to achieve is that whenever the Limits property has a value assigned to it, any changes to Position and Zoom should be validated to ensure the player never sees anything beyond the limited region. Changing the camera's limit should also revalidate everything.

To make this task easier, we'll create two helper methods, ValidateZoom() and ValidatePosition() which will be called from within our setters above and correct any invalid values for us. Once we have these two methods implemented, the rest is trivial.

## Validate Zoom

Making sure the camera's zoom is valid is not too hard once you think about it. In general terms what you'd like to achieve is to ensure that the area that the camera can see is never larger than the limiting rectangle.

So, exactly how big is this area that the camera can see? Well, we know that when the camera isn't zoomed at all (i.e. the Zoom property is 1), then the area the camera can see corresponds exactly to the game's Viewport. It is also intuitive to think that if you're zoomed in, you can see less of the world. Conversely, if you're zoomed out you can see more than you could before.

This implies that the camera "size" (i.e. the size of the area visible by the camera) is inversely proportional to the camera's zoom, or in mathematical terms:

~~~ c#
CameraSize = ViewportSize / Zoom
~~~

Now that we know how large the camera is, let us revisit our initial statement - "Ensure that the area that the camera can see is never larger than the limiting rectangle". We can just as easily model that with a mathematical expression:

~~~ c#
CameraSize <= LimitSize
~~~

Combining both equations into one and rearranging it we arrive at the following conclusion:

~~~ c#
Zoom >= ViewportSize / LimitSize
~~~

All that remains is to implement it. The only problem here is that while Zoom is a scalar (float), ViewportSize and LimitSize are both vectors so we can't compare them directly. The solution is to compare separatedly for the X and Y axes and ignore the smallest value of the two. In code:

~~~ c#
private void ValidateZoom()
{
    if (_limits.HasValue)
    {
        float minZoomX = (float)_viewport.Width / _limits.Value.Width;
        float minZoomY = (float)_viewport.Height / _limits.Value.Height;
        _zoom = MathHelper.Max(_zoom, MathHelper.Max(minZoomX, minZoomY));
    }
}
~~~


## Validate Position

Validating the camera's position after moving it or changing zoom is a bit harder. First you need to know where the top left corner of the camera stands in World space. For that you need to take your ViewMatrix and invert it so that you can transform points from View space back into World space, and apply it to the camera's local top left corner (which in View space is simply Vector2.Zero), which gives you its position in world space. That position is then clamped inside the limiting rectangle's min and max corners. However there are two catches that you need to take care of.

The first one is that, we're clamping our top LEFT corner against the limiting rectangle's bottom RIGHT corner, so in order to compensate we need to subtract it the camera's size too (remember the formula above?). The second one is that the camera's top left corner in world space doesn't necessarily correspond to the camera's position (because of zoom). To solve that I start by taking note of how much both of these values differ in order to correct that offset at the end. Here's the code:

~~~ c#
private void ValidatePosition()
{
    if(_limits.HasValue)
    {
        Vector2 cameraWorldMin = Vector2.Transform(Vector2.Zero, Matrix.Invert(ViewMatrix));
        Vector2 cameraSize = new Vector2(_viewport.Width, _viewport.Height) / _zoom;
        Vector2 limitWorldMin = new Vector2(_limits.Value.Left, _limits.Value.Top);
        Vector2 limitWorldMax = new Vector2(_limits.Value.Right, _limits.Value.Bottom);
        Vector2 positionOffset = _position - cameraWorldMin;
        _position = Vector2.Clamp(cameraWorldMin, limitWorldMin, limitWorldMax - cameraSize) + positionOffset;
    }
}
~~~


## Updating the Properties

All that remains is to make use of these two methods in order to validate our data. Just add them inside your property setters. When changing position you don't need to do any zoom validation. When changing zoom however, you have to follow it with a position validation because if you were standing close to the border, the position might end up being invalid after zooming out. Same thing when changing the camera's limit, don't forget to validate everything.

~~~ c#
public Vector2 Position
{
    get { return _position; }
    set
    {
        _position = value;
        ValidatePosition();
    }
}

public float Zoom
{
    get { return _zoom;	}
    set
    {
        _zoom = value;
        ValidateZoom();
        ValidatePosition();
    }
}

public Rectangle? Limits
{
    set
    {
        _limits = value;
        ValidateZoom();
        ValidatePosition();
    }
}
~~~


## Usage Example

Using this technique is as simple as setting a new Rectangle to the Limits property, such as:

~~~ c#
_camera = new Camera(GraphicsDevice.Viewport);
_camera.Limits = new Rectangle(0, 0, 512, 512);
~~~

No matter how big or small your Viewport and Limits rectangle currently are, the camera's position and zoom are initialized so that the player is guaranteed to only see the inside of that region. As you saw on the video earlier, I cycle through rectangles of many different sizes and locations and the camera is automatically updated to focus on that content.

[Source Code](https://github.com/davidluzgouveia/CameraLimiting)
{: .button}
