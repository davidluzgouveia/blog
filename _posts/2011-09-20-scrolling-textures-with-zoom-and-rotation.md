---
layout: post
title: Scrolling Textures with Zoom and Rotation
date: 2011-09-20
---

I've already written twice before ([here]({{ "/scrolling-textures-in-xna" | relative_url }}) and [here]({{ "/2d-camera-with-parallax-scrolling-in-xna" | relative_url }})) about a way to make a texture seem to repeat itself infinitely in every direction as you moved your camera around, all with a single draw instruction. That can be very useful in a number of game scenarios, such as creating a spacefield background for a spaceship shooter type of game.

That technique relied on the graphic's card wrap texture addressing mode to do the wrapping automatically for you, while the spritebatch's source and destination rectangles were used to to move (changing the source rectangle position) or zoom (change the source rectangle size in relation to destination rectangle) the camera. But it still had the problems that it didn't support rotation (setting the rotation on SpriteBatch.value would cause the entire quad to rotate, not just the content) or respect the camera's origin.

In order to do that, we need a bit more control. So this time I'll show you how to correct this behavior by taking control over the texture coordinate generation process using a simple vertex shader. Using this technique you can take any texture (of any size) and make it repeat infinitely no matter what camera orientation you're currently in. Consider it an upgrade to the older article!

I'll start with the video sample for this article which should show you what to expect from it:

<iframe width="420" height="315" src="http://www.youtube.com/embed/I-GeeSntNrQ?rel=0" allowfullscreen="" frameborder="0"></iframe>

## Part 1 - Setting up XNA

Okay, so we'd like to load a texture and draw it so that it fills the entire screen and seems to repeat infinitely. Furthermore we want to do this in a single draw call, and also want to be able to transform our camera at will without the effect breaking. You should already know how to do the first two steps of the process from the other article, which are, loading the texture and starting the spritebatch in wrap mode:

1) Load the texture:

~~~ c#
// Class scope
Texture2D _bg;

// On LoadContent
_bg = Content.Load<Texture2D>("bg");
~~~

2) Activate texture wrapping - Pass SamplerState.LinearWrap to SpriteBatch.Begin:

~~~ c#
// Down inside your Draw method
_spriteBatch.Begin(SpriteSortMode.Deferred, null, SamplerState.LinearWrap, [omitted other parameters]);
~~~

Next what we will do is draw this texture using a quad (i.e. two triangles connected to each other that form a rectangle - SpriteBatch already does this for us) that ALWAYS covers the entire screen, and leave it to a vertex shader to displace our texture coordinates so that it looks like we're seeing it from another angle and distance. This means that even though the quad is always standing still, because of our vertex shader, its content will appear to change position, rotation, zoom, etc.

So let's render that quad...

3) Render the texture as a fullscreen quad - Pass GraphicsDevice.Viewport.Bound as both source and destination rectangles to SpriteBatch.Draw:

~~~ c#
// Draw texture to viewport-sized quad setting both source and destination rectangles
_spriteBatch.Draw(_bg, GraphicsDevice.Viewport.Bounds, GraphicsDevice.Viewport.Bounds, Color.White);
~~~

Now on to the vertex shader...

## Part 2 - Vertex Shader

I'll be using a vertex shader to take the original texture coordinates for each of the quad's vertices, and displace them so that we see any entirely different portion of the texture depending on our camera parameters. By doing so, the texture content will appear to move, rotate or zoom, even though the quad's geometry (the four vertices that make up the quad) will still be fixed to our screen's corners. Also, it doesn't matter if the resulting coordinate falls outside the 0.0-1.0 range because we are using the wrap texture addressing mode which automatically brings them back into the 0.0-1.0 range at the end!

Here's the vetex shader we need to use:

~~~ c#
sampler TextureSampler : register(s0);
float2 ViewportSize;
float4x4 ScrollMatrix;

void SpriteVertexShader(inout float4 color : COLOR0, inout float2 texCoord : TEXCOORD0, inout float4 position : POSITION0)
{
    // Half pixel offset for correct texel centering.
    position.xy -= 0.5;

    // Viewport adjustment.
    position.xy = position.xy / ViewportSize;
    position.xy *= float2(2, -2);
    position.xy -= float2(1, -1);

    // Transform our texture coordinates to account for camera
    texCoord = mul(float4(texCoord.xy, 0, 1), ScrollMatrix).xy;
}

technique SpriteBatch
{
    pass
    {
        VertexShader = compile vs_2_0 SpriteVertexShader();
    }
}
~~~

The vertex shader is pretty minimalistic. Most of the vertex shader is boilerplate code that is needed for any vertex shader working with SpriteBatch, which I got straight of the MSDN samples. The only relevant line is where I transform the texture coordinates (last line of the vertex shader function) using a matrix that I called the "scroll matrix" (by lack of a better name - I just made that up). That's the matrix which does all of the work, and I'll describe it next. You'll need to supply that matrix along with the viewport's size from your game.

If you've never used a shader before, you should read some other tutorials first, but basically you need to put this code on a text file, give it an ".fx" file extension and add it to your content pipeline. You load it just like a Texture or another content pipeline object, but using the class Effect instead.

## Part 3 - Back on XNA: Generating the Scroll Matrix

Creating the "scroll matrix" is not too hard. If you already have a camera class, you only need to add this method to it:

~~~ c#
public Matrix GetScrollMatrix(Vector2 textureSize)
{
    return Matrix.CreateTranslation(new Vector3(-Origin / textureSize, 0.0f)) *
        Matrix.CreateScale(1f / Zoom) *
        Matrix.CreateRotationZ(Rotation) *
        Matrix.CreateTranslation(new Vector3(Origin / textureSize, 0.0f)) *
        Matrix.CreateTranslation(new Vector3(Position / textureSize, 0.0f));
}
~~~

If you compare it with your camera's usual View Matrix you'll notice a few differences. This one is basically a World Matrix (notice the S-R-T order of multiplication instead of T-R-S) with the catch that all translations (i.e. Position and Origin) are mapped to the texture's space (which is sometimes called uv space too - a two dimensional space where where (0,0) corresponds to the top-left corner of the texture, and and (1,1) corresponds to the bottom-right corner of the texture) before being applied, which can be done by dividing their magnitudes by the texture's size.

This matrix basically figures out where each of the corners of our screen/quad lie in texture space based on the camera's current transform. After figuring that out, the default pixel shader takes care of the rest!

## Part 4 - Loading and using the Effect

Now the only thing we need to do is load the vertex shader effect, pass it the camera's "scroll matrix" and viewport size, and use it on our SpriteBatch when drawing the texture:

1) Load effect:

~~~ c#
// Global scope
Effect _effect;

// On LoadContent
_effect = Content.Load<Effect>("infinite");
_effect.Parameters["ViewportSize"].SetValue(new Vector2(GraphicsDevice.Viewport.Width, GraphicsDevice.Viewport.Height));
~~~

2) Pass scroll matrix to the effect:

~~~ c#
// On Draw
// Prepare the effect with a view matrix
_effect.Parameters["ScrollMatrix"].SetValue(_scrollCamera.GetScrollMatrix(new Vector2(_bg.Width, _bg.Height)));
// Start the SpriteBatch using LinearWrap and the scroll effect
_spriteBatch.Begin(SpriteSortMode.Deferred, null, SamplerState.LinearWrap, null, null, _effect);
~~~

3) Prepare SpriteBatch to use our effect:

~~~ c#
// On Draw
// Start the SpriteBatch using LinearWrap and the scroll effect
_spriteBatch.Begin(SpriteSortMode.Deferred, null, SamplerState.LinearWrap, null, null, _effect);
~~~

And that's all there is to it. Check the source code below for the complete source code for this article:

## Final Thoughts

My first version of this article used a pixel shader instead of a vertex shader to do all the work. I've now changed it into a vertex shader instead which gives the same result, but is much, much more efficient (e.g. on my sample, with a vertex shader I transform only four texture coordinates for the entire effect, while with a pixel shader I was transforming hundreds of thousands of texture coordinates). For reference, here is the pixel shader I was using.

> Warning: Use the vertex shader version. It's a lot more efficient.
{: .error}

~~~ c#
sampler TextureSampler : register(s0);
float4x4 ScrollMatrix;

float4 main(float4 color : COLOR0, float2 texCoord : TEXCOORD0) : COLOR0
{
    // Convert our uv coordinates into a 4d vector for matrix multiplication
    // Transform it by the Scroll Matrix
    float2 coords = mul(float4(texCoord.xy, 0, 1), ScrollMatrix).xy;

    // Sample the texture at the calculated spot
    return tex2D(TextureSampler, coords) * color;
}

technique Infinite
{
    // Make sure you're using Wrap mode and a full screen quad
    pass Pass1
    {
        PixelShader = compile ps_2_0 main();
    }
}
~~~

[Source Code](https://github.com/davidluzgouveia/blog-infinite-layer)
{: .button}
