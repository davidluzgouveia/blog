---
layout: post
title: 2D Camera with Parallax Scrolling in XNA
date: 2011-04-10
---

Parallax scrolling is a technique that is frequently used in sidescrolling 2D games to give the impression of depth. It consists of making objects that are further away appear to move slower in relation to the viewer than objects that are near. Pay attention to your surroundings next time you're on a moving vehicle, and you'll notice that this also appears to happen in real life!

They say a picture is worth a thousand words, but with parallax scrolling you really have to see it in motion in order to appreciate it. So I'll start by showing the video sample for this article right away:

<iframe width="420" height="315" src="//www.youtube.com/embed/1u5cGZPdyEk?rel=0" frameborder="0" allowfullscreen=""></iframe>

There you have it. So, there are actually many different ways to achieve this effect, but I'd like to focus on the two that I find easier to use:

1. **Scrolling Textures** - Useful for scenes like a starfield or a 2D sky. Somewhat limited beyond that.
2. **Camera Modification** - General purpose and much more flexible.


## Method 1 - Scrolling Textures

If you want to implement parallax scrolling entirely with repeating textures, then the easiest way is to use the technique described in my earlier article called [Scrolling Textures in XNA]({{ "/scrolling-textures-in-xna" | relative_url }}). This is useful for instance to create starfields or 2D skies, but not as useful for games where the background needs to change constantly.

So let's say you have three textures (texture1, texture2 and texture3) that you want to draw on top of each other and scroll them at different speeds. All you have to do is use the [technique I described]({{ "/scrolling-textures-in-xna" | relative_url }}) passing your camera position as your texture's scroll position, and multiplying each of them by a different constant (e.g. 1.0f would mean full speed, 0.5f half speed, 2.0f double speed, etc.). Example below:

~~~ c#
spriteBatch.Begin(SpriteSortMode.Deferred, null, SamplerState.LinearWrap, null, null);
spriteBatch.Draw(texture1, position, new Rectangle(cameraX * 0.5f, cameraY * 0.5f, texture1.Width, texture1.Height), Color.White);
spriteBatch.Draw(texture2, position, new Rectangle(cameraX * 0.8f, cameraY * 0.8f, texture2.Width, texture2.Height), Color.White);
spriteBatch.Draw(texture3, position, new Rectangle(cameraX * 1.0f, cameraY * 1.0f, texture3.Width, texture3.Height), Color.White);
spriteBatch.End();
~~~

That's all there is to it! And in case you didn't notice, the video sample of my [Scrolling Textures in XNA]({{ "/scrolling-textures-in-xna" | relative_url }}) article already had parallax scrolling in it - there are in fact three layers of clouds moving at different speeds - although the effect is very subtle!

<iframe width="420" height="315" src="//www.youtube.com/embed/Ly7SJeiirhU?rel=0" frameborder="0" allowfullscreen></iframe>

> Update: I wrote a new article about this topic which lets you use zoom and rotation at the same time as scrolling the background infinitely. Check it out [here]({{ "/scrolling-textures-with-zoom-and-rotation" | relative_url }}).**
{: .info}

## Method 2 - Camera Modification

Although using scrolling textures is quite easy, it is not applicable to all cases. Most of the time, your game world simply won't be made up of repeating textures, but you'll still want to leverage parallax scrolling. Other times you'll also need the ability to move your camera around, rotate it and zoom it.

A couple years ago, David Amador wrote a [popular article](http://www.david-amador.com/2009/10/xna-camera-2d-with-zoom-and-rotation/) on how to create a 2D camera with zoom and rotation using a simple transformation matrix. I'm not going to describe the entire technique again here, so if you don't already have a matrix based 2D camera implemented, you might want to start there in order to understand the basics.

It turns out that adding parallax scrolling support to a matrix based camera class is extremely simple (the video at the top of the article uses this technique). Here's the entire camera implementation:

~~~ c#
public class Camera
{
    public Camera(Viewport viewport)
    {
        Origin = new Vector2(viewport.Width / 2.0f, viewport.Height / 2.0f);
        Zoom = 1.0f;
    }

    public Vector2 Position { get; set; }
    public Vector2 Origin { get; set; }
    public float Zoom { get; set; }
    public float Rotation { get; set; }

    public Matrix GetViewMatrix(Vector2 parallax)
    {
        // To add parallax, simply multiply it by the position
        return Matrix.CreateTranslation(new Vector3(-Position * parallax, 0.0f)) *
            // The next line has a catch. See note below.
            Matrix.CreateTranslation(new Vector3(-Origin, 0.0f)) *
            Matrix.CreateRotationZ(Rotation) *
            Matrix.CreateScale(Zoom, Zoom, 1) *
            Matrix.CreateTranslation(new Vector3(Origin, 0.0f));
    }
}
~~~

I simplified it as much as possible, but the only change made in order to support parallax was to make the *GetViewMatrix* method require you to specify the desired parallax factor (if you do not wish to have any parallax, simply pass it *Vector.One*), which is then multiplied by the camera's position when calculating the view matrix.

> The Origin Trick: I made another change in comparison with David Amador's method (which is more a matter of personal preference) that was include an additional (negative) Origin translation matrix before the Rotation and Zoom matrices (I marked the code with a comment). I prefer doing this because it makes your origin calculations only affect zooming and rotating, but leave translation untouched. That's because I prefer to set my camera's position using the top-left corner, but still want to have it rotate and zoom around the center of the screen. This way you can have both at the same time! Also I recommend doing this since it makes some optional calculations (such as limiting the camera's position or looking at an object) easier to handle, because you won't have to factor in your origin.
{: .info}

What this means is that, for instance, if you pass it a Vector2(0.5f, 0.5f), objects will appear to move at half of their original speed, and if you pass it a Vector2(2.0f, 2.0f) they'll appear to move twice as fast. As I mentioned above, passing it Vector.One means that there will be no parallax scrolling applied to that object, and passing it a Vector2.Zero will make the object stay in the same place even as you move the camera around. You can even pass different amounts of parallax for the X and Y axis if you wish!

## Usage Scenario

Now, in order to use this class, simply instantiate it passing it your current viewport:

~~~ c#
Camera camera = new Camera(GraphicsDevice.Viewport);
~~~

And use the *GetViewMatrix* method to get a transformation matrix for your SpriteBatch:

~~~ c#
Vector2 parallax = new Vector2(0.5f);
spriteBatch.Begin(SpriteSortMode.Deferred, null, null, null, null, null, camera.GetViewMatrix(parallax));
spriteBatch.Draw(texture, position, Color.White); // This sprite will appear to move at 50% of the normal speed
spriteBatch.End();
~~~

By the way, I'm not worrying about performance optimizations in this article for the sake of simplicity. In practice, you might not want to recalculate the view matrix every frame, and instead cache it and only update it when the camera changes. I'll leave that exercise up to the reader, but if you need a tip just leave a comment below.

If all you wanted to know was how to implement parallax scrolling, then by now you should already have everything that you need. Below I'll be describing some more advanced topics, which I believe should be worthwile to know, so I recommend reading to the end!

## Optional: Layer Grouping

I find that the easiest way to handle different parallax values it to introduce the concept of layers. A layer is simply a group of game objects that share the same parallax value. Ideally you should be able to create as many layers as you want, set each of their parallax speeds, add your game objects to them, and just tell them to draw themselves. Here's how I usually do it:

I'll start by creatig a *Layer* class which has its own parallax speed, and holds a bunch of *Sprite* objects. Calling *Draw* will make all sprites be displayed with the correct parallax.

Here's a minimal *Sprite* class implementation (for simplicity I stripped out rotation and scaling information, since it doesn't serve any purpose in this example):

~~~ c#
public struct Sprite
{
    public Texture2D Texture;
    public Vector2 Position;

    public void Draw(SpriteBatch spriteBatch)
    {
        if(Texture != null)
            spriteBatch.Draw(Texture, Position, Color.White);
    }
}
~~~

And here's the *Layer* class. You need to already have a *Camera* object created when you create a new *Layer*, but after that it's pretty straighforward. The only significant part is the call to *SpriteBatch.Begin* which relies on the camera to build a view matrix for the appropriate parallax value.

~~~ c#
public class Layer
{
    public Layer(Camera camera)
    {
        _camera = camera;
        Parallax = Vector2.One;
        Sprites = new List<Sprite>();
    }

    public Vector2 Parallax { get; set; }
    public List<Sprite> Sprites { get; private set; }

    public void Draw(SpriteBatch spriteBatch)
    {
        spriteBatch.Begin(SpriteSortMode.Deferred, null, null, null, null, null, _camera.GetViewMatrix(Parallax));
        foreach(Sprite sprite in Sprites)
            sprite.Draw(spriteBatch);
        spriteBatch.End();
    }

    private readonly Camera _camera;
}
~~~


## Usage Scenario

Putting it all together, you could for example setup a scene with 9 different parallax layers. Each of these layers will get a different parallax value, hold one sprite:

~~~ c#
_layers = new List<Layer>();

// Create a camera instance and limit its moving range
_camera = new Camera(GraphicsDevice.Viewport) { Limits = new Rectangle(0, 0, 3200, 600) };

// Create 9 layers with parallax ranging from 0% to 100% (only horizontal)
_layers = new List<Layer>
{
    new Layer(_camera) { Parallax = new Vector2(0.0f, 1.0f) },
    new Layer(_camera) { Parallax = new Vector2(0.1f, 1.0f) },
    new Layer(_camera) { Parallax = new Vector2(0.2f, 1.0f) },
    new Layer(_camera) { Parallax = new Vector2(0.3f, 1.0f) },
    new Layer(_camera) { Parallax = new Vector2(0.4f, 1.0f) },
    new Layer(_camera) { Parallax = new Vector2(0.5f, 1.0f) },
    new Layer(_camera) { Parallax = new Vector2(0.6f, 1.0f) },
    new Layer(_camera) { Parallax = new Vector2(0.8f, 1.0f) },
    new Layer(_camera) { Parallax = new Vector2(1.0f, 1.0f) }
};

// Add one sprite to each layer
_layers[0].Sprites.Add(new Sprite { Texture = Content.Load<Texture2D>("Layer1") });
_layers[1].Sprites.Add(new Sprite { Texture = Content.Load<Texture2D>("Layer2") });
_layers[2].Sprites.Add(new Sprite { Texture = Content.Load<Texture2D>("Layer3") });
_layers[3].Sprites.Add(new Sprite { Texture = Content.Load<Texture2D>("Layer4") });
_layers[4].Sprites.Add(new Sprite { Texture = Content.Load<Texture2D>("Layer5") });
_layers[5].Sprites.Add(new Sprite { Texture = Content.Load<Texture2D>("Layer6") });
_layers[6].Sprites.Add(new Sprite { Texture = Content.Load<Texture2D>("Layer7") });
_layers[7].Sprites.Add(new Sprite { Texture = Content.Load<Texture2D>("Layer8") });
_layers[8].Sprites.Add(new Sprite { Texture = Content.Load<Texture2D>("Layer9") });
~~~

And draw everything with a simple loop:

~~~ c#
foreach (Layer layer in _layers)
    layer.Draw(_spriteBatch);
~~~

And if you want to move the camera around, just use the properties exposed by the *Camera* class. All parallax magic will be done for you behind the scenes!

## Frequently Asked Questions

Since I probably won't be talking about the topic of 2D cameras again, I'll take the chance to answer a few questions that I often see on the forums:

**_Q: When moving the camera around while it's rotated, how do I make it follow the camera's rotation?_**

A: You'll need to do a little math for this. Add this method to your camera class:

~~~ c#
public void Move(Vector2 displacement, bool respectRotation = false)
{
    if (respectRotation)
    {
        displacement = Vector2.Transform(displacement, Matrix.CreateRotationZ(-Rotation));
    }

    Position += displacement;
}
~~~

Now whenever you want to move in accordance with the camera's rotation, just set the "respectRotation" parameter to true!

**_Q: How do I make my camera look at some object, or follow my character?_**

A: Centering your camera around an object is simply a matter of settings its position to the same as the object, and subtracting half of the screen's size from it (so you need to store your screen size inside your camera class. For this example I'll assume that you stored a *Viewport* object in a member variable called _viewport). Here's an helper method that will center the camera around some position:

~~~ c#
public void LookAt(Vector2 position)
{
    Position = position - new Vector2(_viewport.Width / 2.0f, _viewport.Height / 2.0f);
}
~~~

Now if you want to make your camera always follow your character, all you need to do is add something like this to the end of your update loop:

~~~ c#
// Updates your character's position
character.Update(gameTime);

// Updates your camera to lock on the character
camera.LookAt(character.Position);
~~~

> Note: If you didn't use my origin trick (explained earlier), then you'll also need to add the camera's Origin at the end in order to correct the offset.
{: .note}

**_Q: How do I make the camera stop moving beyond the edges of my level?_**

A: Okay, this is a bit more complicated, so I'll go over it in a few separate steps.

First of all, just like the previous question, one thing your limiting code will need to know is how big your viewport is. So start by storing the viewport when you get it in the constructor for later use.

~~~ c#
public Camera(Viewport viewport)
{
    _viewport = viewport;
}

private readonly Viewport _viewport;
~~~

Next I'll add a Rectangle property called Limits that lets you choose the extent of what your camera should be able to see. This extent should always be at least as big as the viewport. The question mark (?) after Rectangle makes it possible to assign it a null value. This isn't normally possible because Rectangle is a value type. I made it this way so that setting Limits to null will disable camera bound limiting altogether. Here's the code (notice how I validate the rectangle to ensure it's always bigger than the viewport):

~~~ c#
public Rectangle? Limits
{
    get { return _limits; }
    set
    {
        if(value != null)
        {
            // Assign limit but make sure it's always bigger than the viewport
            _limits = new Rectangle
                {
                    X = value.Value.X,
                    Y = value.Value.Y,
                    Width = System.Math.Max(_viewport.Width, value.Value.Width),
                    Height = System.Math.Max(_viewport.Height, value.Value.Height)
                };

            // Validate camera position with new limit
            Position = Position;
        }
        else
        {
            _limits = null;
        }
    }
}

private Rectangle? _limits;
~~~

Finally, change your Position property so that its setter checks for the existance of a limit, and clamps the position accordingly. This technique as it stands doesn't work so well when the camera is scaled or rotated, so I disabled the limiting behavior when the camera is transformed for the sake of correctness. But feel free to try to work around that limitation!

~~~ c#
public Vector2 Position
{
    get { return _position; }
    set
    {
        _position = value;

        // If there's a limit set and the camera is not transformed clamp position to limits
        if(Limits != null && Zoom == 1.0f && Rotation == 0.0f)
        {
            _position.X = MathHelper.Clamp(_position.X, Limits.Value.X, Limits.Value.X + Limits.Value.Width - _viewport.Width);
            _position.Y = MathHelper.Clamp(_position.Y, Limits.Value.Y, Limits.Value.Y + Limits.Value.Height - _viewport.Height);
        }
    }
}
~~~

That's it! Now if your level starts at (0,0) and has a size of (8000, 600) you can make sure your camera won't leave that region by simply doing:

~~~ c#
yourCamera.Limits = new Rectangle(0, 0, 8000, 600);
~~~

Even better is that if you combine this with the *LookAt* method described above, you will automatically get the common behavior where the camera seems to stop when your character reaches the borders of the map, and resumes again when you move away from the borders! No additional code needed.

> Note: Once again, if you didn't use the "origin trick" before you'll also need to add the origin's X and Y values to the Clamp calculation (inside the Clamp operations, on the min and max parameters).
{: .note}

> Update: I wrote another article with more information about this matter. Check it out here: [Limiting 2D Camera Movement with Zoom]({{ "/limiting-2d-camera-movement-with-zoom" | relative_url }}).
{: .info}

**_Q: How do I find what position in the screen (or world) a certain point in the world (or screen) corresponds to?_**

This is common for instance when you want to pick objects on the screen with your mouse (screen to world), or to find out which objects are inside of your view (world to screen). If you have parallax scrolling, then this is something you'll want to do at layer level (since each layer can have a different view matrix), but the general idea is still the same. If you don't have parallax scrolling, just do this in your camera class instead with the appropriate changes!

~~~ c#
public Vector2 WorldToScreen(Vector2 worldPosition)
{
    return Vector2.Transform(worldPosition, camera.GetViewMatrix(parallax));
}

public Vector2 ScreenToWorld(Vector2 screenPosition)
{
    return Vector2.Transform(screenPosition, Matrix.Invert(camera.GetViewMatrix(parallax)));
}
~~~

This will get you the correct results, even if your camera is all rotated and scaled.

## Conclusions

That's all for this article. If you have any questions that I didn't cover here, leave a comment below and I'll do my best to answer them and update the FAQ accordingly. And as usual, you can find the source code for this article's sample below. The camera class provided has all of the tips I described implemented, and I also included a SimpleCamera class (although I'm not using it) which is the bare minimum you need to get things working.

[Source Code](https://github.com/davidluzgouveia/ParallaxScrolling)
{: .button}
