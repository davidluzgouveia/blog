---
layout: post
title: RenderMonkey Beginner's Tutorial
date: 2011-09-11
---

This time I'll be presenting a beginner's tutorial about RenderMonkey, which is a shader development environment created by AMD that facilitates the task of creating and testing shader programs. Although AMD has stopped developing it, you can sill [get it for free](http://developer.amd.com/archive/gpu/rendermonkey/).

I'll be using HLSL for the scope of this tutorial, but I intend to focus only on how to get RenderMonkey up and running, so I won't be teaching you how to program in HLSL. If you don't have any experience, [here's a nice resource](http://www.catalinzima.com/tutorials/crash-course-in-hlsl/) for getting started with HLSL - kudos to [Catalin Zima](http://www.catalinzima.com/) for the wonderful tutorial. After learning the syntax, you should also learn how to implement some basic shaders. The XNA education catalogue[ has a series just for that](http://create.msdn.com/en-US/education/catalog/article/shader_primer)!

## Shader Code

The shader used in this tutorial mixes ambient and diffuse lighting with a diffuse texture map. Here's the code for the shader:

~~~ c#
float4x4 WorldViewProjection;
float4x4 WorldInverseTranspose;
float3 LightDirection;
float4 Ambient;
sampler Texture;

struct VS_INPUT
{
    float4 position : POSITION0;
    float3 normal : NORMAL0;
    float2 texturecoord : TEXCOORD0;
};

struct VS_OUTPUT
{
    float4 position : POSITION0;
    float3 normal : TEXCOORD0;
    float2 texturecoord : TEXCOORD1;
};

VS_OUTPUT vs_main(VS_INPUT input)
{
    VS_OUTPUT output;
    output.position = mul(input.position, WorldViewProjection);
    output.normal = mul(input.normal, WorldInverseTranspose);
    output.texturecoord = input.texturecoord;
    return output;
}

float4 ps_main(VS_OUTPUT input) : COLOR0
{
    float3 normal = normalize(input.normal);
    float3 light = normalize(LightDirection);
    float4 diffuse = tex2D(Texture, input.texturecoord);
    float diffuseContribution = clamp(dot(normal, -light), 0, 1);
    return Ambient + diffuse * diffuseContribution;
}
~~~

And what it's supposed to look like:

[![0]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/0-300x213.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/0.jpg" || relative_url }})

So let's get started!

## Creating the Effect

When starting RenderMonkey you'll get an Empty Workspace that's ready to be used. Your application should look like this:

[![01]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/01-300x208.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/01.jpg" | relative_url }})

A lot of your work will be done on the Workspace Window on the left, which by default shows a single empty workspace. So let's start by adding a new Effect to it. Right-click on Effect Workspace and add a new Default Effect of your desired type (2). Since I'm programming in HLSL, I created a new DirectX shader. There are many templates ready that you can use when starting your own shaders, but for this tutorial I'll just choose "DirectX" from the list which creates the most basic shader possible (i.e. a vertex shader that simply applies a WorldViewProjection transformation and a pixel shader that always outputs the same color).

[![02]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/02-300x63.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/02.jpg" | relative_url }})

Once you have your Effect created, your window should look like this:

[![03]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/03-300x208.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/03.jpg" | relative_url }})

Now we're ready to start setting everything up!

## Changing the Model

You probably noticed that the default model used by RenderMonkey is a sphere. That's not the best choice for testing your shaders, so let's start by changing it. On your Workspace Window simply right-click on Model and select a new one:

[![04]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/04.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/04.jpg" | relative_url }})

I'll use a simple teapot:

[![05]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/05-300x208.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/05.jpg" | relative_url }})

## Setting Variables

The next step is to create a link between RenderMonkey and the global variables used in our shader. The global variables used by our shader are as follows:

~~~ c#
float4x4 WorldViewProjection;
float4x4 WorldInverseTranspose;
float3 LightDirection;
float4 Ambient;
sampler Texture;
~~~

We'll need to create a new variable inside our Effect on the Workspace Window for each global variable in our code, so that RenderMonkey can interact with our shader. Make sure to give them the same name and the appropriate data type too. Creating a new variable is as simple as right-clicking on your Effect, going under Add Variable and picking the right data type from the list:

[![06]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/06-300x135.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/06.jpg" | relative_url }})

You can start by creating a Float3 called LightDirection and a Color called Ambient.

As for defining our matrixes (such as WorldViewProjection and WorldInverseTranspose), these usually have special meanings, and we'd like RenderMonkey to establish the right connection between them and our scene (e.g. link the View matrix to our camera). To do that, simply use one of the Predefined matrixes that RenderMonkey provides, as each of them already come with the appropriate semantic assigned:

[![07]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/07-300x148.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/07.jpg" | relative_url }})

Finally, for creating the texture variable that will be sampled by the shader, we need to do it in two steps.

First, right-click on your Effect, select Add Texture and load a 2D Texture of your choice:

[![08]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/08-300x108.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/08.jpg" | relative_url }})

Next, scroll down to Pass 0, right-click on it and add a new Texture Object, linking it to the texture you just created. Be sure to name this Texture Object the same as the sampler used in your code (in our case, Texture):

[![09]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/09-300x250.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/09.jpg" | relative_url }})

I also got rid of the matrix that came with the default effect since I won't be needing it. When you're done, your Effect tree should look like:

[![10]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/10.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/10.jpg" | relative_url }})

## Binding Shader Inputs

One other step we need to take, that is frequently forgotten, is to bind RenderMonkey to our vertex shader inputs so that the correct data is streamed. Looking back on our code, we had the following vertex shader input structure:

~~~ c#
struct VS_INPUT
{
    float4 position : POSITION0;
    float3 normal : NORMAL0;
    float2 texturecoord : TEXCOORD0;
};
~~~

We'll need to mimick this structure on RenderMonkey. Inside your Effect tree, right-click on Stream Mapping and choose Edit:

[![11]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/11-273x300.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/11.jpg" | relative_url }})

On the window that appears, simply add two new channels and configure them such that they're compatible with the data types and semantics of your vertex shader input. In our case:

[![12]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/12-300x106.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/12.jpg" | relative_url }})

## Create the Code

Now for the most important part, entering the code. Scroll down to Pass 0 and right-click on Vertex Shader, choosing Edit from the list. That should open the Code Editor for your effect, with the default implementation still on it:

[![13]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/13.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/13.jpg" | relative_url }})

[![14]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/14-300x179.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/14.jpg" | relative_url }})

The most relevant thing to notice here is that unlike most shaders you write, in RenderMonkey you will write your vertex shader and your pixel shader in separate tabs, and the variables you define in one of the tabs are local to it (i.e. can't be accessed by the other tab).

This means that you'll have to split your shader code in two, the vertex half and the pixel half. That's no big deal though. Here's what it looks like after the split (refer to the full implementation on top for comparison).

Under the Vertex Shader tab:

~~~ c#
float4x4 WorldViewProjection;
float4x4 WorldInverseTranspose;

struct VS_INPUT
{
    float4 position : POSITION0;
    float3 normal : NORMAL0;
    float2 texturecoord : TEXCOORD0;
};

struct VS_OUTPUT
{
    float4 position : POSITION0;
    float3 normal : TEXCOORD0;
    float2 texturecoord : TEXCOORD1;
};

VS_OUTPUT vs_main(VS_INPUT input)
{
    VS_OUTPUT output;
    output.position = mul(input.position, WorldViewProjection);
    output.normal = mul(input.normal, WorldInverseTranspose);
    output.texturecoord = input.texturecoord;
    return output;
}
~~~

Under the Pixel Shader tab:

~~~ c#
float3 LightDirection;
float4 Ambient;
sampler Texture;

struct VS_OUTPUT
{
    float4 position : POSITION0;
    float3 normal : TEXCOORD0;
    float2 texturecoord : TEXCOORD1;
};

float4 ps_main(VS_OUTPUT input) : COLOR0
{
    float3 normal = normalize(input.normal);
    float3 light = normalize(LightDirection);
    float4 diffuse = tex2D(Texture, input.texturecoord);
    float diffuseContribution = clamp(dot(normal, -light), 0, 1);
    return Ambient + diffuse * diffuseContribution;
}
~~~

The only points worth noting is that I repeated my VS_OUTPUT structure since it was needed on both sides, and I cleaned my global variables so that only the ones that were used in each file were present.

Also, ensure that your vertex shader and pixel shader functions names match the ones defined under Entry Point. By default, these should be "vs_main" and "ps_main" respectively.

Now let's see the code in action! Simply press the Compile button on the toolbar:

[![15]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/15-300x83.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/15.jpg" | relative_url }})

And this is what you should get:

[![16]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/16-300x208.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/16.jpg" | relative_url }})

It looks all white! But don't worry, that's only because your LightDirection and Ambient variables are still with their default values. RenderMonkey provides an interesting way for you to edit your variables, so that you may see your shader updating in real-time as you change them!

## Tweaking Variables

There are two variables we'd like to tweak here: LightDirection and Ambient. Start by right-clicking on your LightDirection variable and setting it as an Artist Variable. Colors are set as Artist Variables by default so you don't need to repeat the process for your Ambient color:

[![17]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/17.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/17.jpg" | relative_url }})

Now fire up the Artist Editor window:

[![18]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/18.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/18.jpg" | relative_url }})

And this is what you'll see on the right side of your application:

[![19]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/19.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/19.jpg" | relative_url }})

Now all you have to do is tweak the variables to your liking. Expand your options in the Artist Editor window so that you have better control over them. In my case I specified that my LightDirection vector should be normalized, and I set my Ambient color to a very dark color:

[![20]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/20-300x208.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/20.jpg" | relative_url }})

## Texture Sampler Options

Just to throw in one last tip, the texture on my teapot looks a bit rough. I'm guessing it's probably being sampled using point filtering. Let's change that!

Right-click on your Texture Object and choose Edit. That's where you can configure everything about your sampler! I'll simply change my magnification/minification/mipmapping filters to linear, which should make the texture appear smoother:

[![21]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/21-300x280.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/21.jpg" | relative_url }})

[![22]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/22-300x296.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/22.jpg" | relative_url }})

## Conclusion

And that's it! You can now maximize your preview window and enjoy. Oh, and you can use the camera controls located on the toolbar to navigate the scene:

[![23]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/23.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/23.jpg" | relative_url }})

Here's the final result:

[![24]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/24-300x208.jpg" | relative_url }})]({{ "/assets/2011-09-11-rendermonkey-beginners-tutorial/24.jpg" | relative_url }})

I hope you enjoyed this tutorial, and as usual feel free to post any questions on the comment section below.
