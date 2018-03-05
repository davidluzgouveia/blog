---
layout: post
title: One-line Algorithmic Music in XNA
date: 2011-10-21
---

Last week I found an interesting video on YouTube, accompanied by an [article](http://countercomplex.blogspot.com/2011/10/algorithmic-symphonies-from-one-line-of.html):

<iframe width="420" height="315" src="//www.youtube.com/embed/qlrs2Vorw2Y?rel=0" frameborder="0" allowfullscreen=""></iframe>

I liked the idea so much that I decided to give it a try on XNA. I also saw a [JavaScript implementation online](http://wurstcaptures.untergrund.net/music/), where the user could write the code directly into an input field, and the application would generate the corresponding sound clip for you. This left me wondering about two points in particular:

1. Can I get the XNA Audio API to work on a Windows Form application as a standalone, i.e. without having a Game instance or even graphics at all?
2. Is there a way to let the user write the code into a text box and hear the result without having to recompile the application?

The answer to both questions is yes. Here's how I did it.

## Using the XNA Audio API as a standalone library

This one was particularly easy to solve. I started by creating a Windows Forms application and manually *adding a reference* to the Microsoft.Xna.Framework assembly. This was enough to grant me access to everything I needed from XNA, in particular the DynamicSoundEffectInstance class.

There's one catch though. If you're not using the Game class (such as in this case), XNA expects you to call the FrameworkDispatcher.Update() method periodically. I've read that around 20 times a second is enough so I went with that. You also need to call that method at least once before starting audio playback, or your application will crash.

Here's how I solved it. Note: If you're not familiar with XNA's dynamic audio API then check one of my earlier articles such as this: [Creating a Basic Synth in XNA 4.0 - Part II]({{ "/creating-a-basic-synth-in-xna-part-ii" | relative_url }}).

1. Created a new thread for the audio.
2. On that new thread, I started by calling FrameworkDispatcher.Update() once.
3. Then I enter a loop where I call FrameworkDispatcher.Update() again, and submit my audio buffers.
4. I added a Thread.Sleep(50) at the end of the loop to make it loop roughly 20 times per second.

Here's the relevant part of the code. When the form has finished loading, I create my audio buffer, a DynamicSoundEffectInstance object, and start the audio thread:

~~~ c#
protected override void OnLoad(EventArgs e)
{
    // Create byte buffer for storing and submitting audio samples
    _buffer = new byte[256*2];

    // Create a new DynamicSoundEffectInstance at 8Khz Mono
    _instance = new DynamicSoundEffectInstance(8000, AudioChannels.Mono);

    // Creates and starts the audio thread
    new Thread(AudioThread).Start();
}
~~~

And this is the thread method. The comments should make it self-explanatory:

~~~ c#
private void AudioThread()
{
    // Service the XNA Framework once before everything is started
    FrameworkDispatcher.Update();

    // Start the DynamicSoundEffectInstance
    _instance.Play();

    // Go into a loop that repeats around 20 times per second (50ms sleep)
    while (_running)
    {
        // Service the XNA Framework once per loop
        FrameworkDispatcher.Update();

        // Fill and submit buffer
        while (_instance.PendingBufferCount &lt; 3)
            SubmitBuffer();

        Thread.Sleep(50);
    }
}
~~~

To make the audio thread quit when the application exists, I did this:

~~~ c#
protected override void OnClosed(EventArgs e)
{
    // Triggers the audio thread to close
    _running = false;
}
~~~

Finally, here's what I did to submit the audio buffers. I'll talk about the GetSample() method later, since it's linked to the runtime compiler side of the application (the part which lets you write and listen to your own code in real-time). Also notice that in the for-loop I also increment an integer variable called "_time". This is the value that will correspond to the "t" on our formulas:

~~~ c#
private void SubmitBuffer()
{
    // Fill the buffer and advance time
    for (int i = 0; i != _buffer.Length; ++i, ++_time)
        _buffer[i] = (byte) GetSample();

    // Submit buffer
    _instance.SubmitBuffer(_buffer);
}
~~~

And just to make it complete, here's all of the member variables used. Unlike my earlier synth articles, these algorithms work directly with bytes, so I only needed to create a single byte buffer to do the processing:

~~~ c#
private DynamicSoundEffectInstance _instance;
private byte[] _buffer;
private int _time;
private Type _generatorType;
private bool _running = true;
~~~

That's all for the audio generation part of the application. Now for the part that takes your one-line algorithm (as a string) and turns it into code that you can execute!

## The C# Runtime Compiler

Dynamic languages usually have a method that lets you execute code in the form of a string (frequently named something like "eval"). However, C# does not have such a mechanism. What it does have however, is a way to compile and assemble C# code at runtime. This means that you can write a complete class as a string, feed it through the runtime compiler, and then have access to it like any other class in your project (by making use of some Reflection).

So, imagine we had a class like this:

~~~ c#
namespace MusicAlgorithm {
    public static class AudioGenerator {
        public static int Generate(int t)
        {
            return /* YOUR ALGORITHM HERE */;
        }
    }
}
~~~

Then all we'd need to do is call the static AudioGenerator.Generate(_time) method using our "_time" variable, and store the resulting value in our audio buffer. Here's how you can compile and assemble a class like this at runtime. Read through the comments for a better understanding of what I did:

~~~ c#
private void SetAlgorithm(string algorithm)
{
    // Reset values
    _time = 0;
    _generatorType = null;

    // Ignore if the user wrote nothing
    if (String.IsNullOrWhiteSpace(algorithm))
        return;

    // Create string containing the code for the class
    // It concatenates the algorithm written by the user with the return statement
    StringBuilder codeBuilder = new StringBuilder();
    codeBuilder.AppendLine("namespace MusicAlgorithm {");
    codeBuilder.AppendLine(" public static class AudioGenerator {");
    codeBuilder.AppendLine(" public static int Generate(int t) { return " + algorithm + "; }");
    codeBuilder.AppendLine(" }");
    codeBuilder.AppendLine("}");
    string code = codeBuilder.ToString();

    // Compile code string in memory. Notice that the language chosen is C#
    // And how the GenerateExecutable and GenerateInMemory properties are set
    CodeDomProvider codeDomProvider = CodeDomProvider.CreateProvider("C#");
    CompilerParameters compileParams = new CompilerParameters { GenerateExecutable = false, GenerateInMemory = true };
    CompilerResults compilerResults = codeDomProvider.CompileAssemblyFromSource(compileParams, new[] {code});

    // On error display message and exit
    if (compilerResults.Errors.HasErrors)
    {
        MessageBox.Show(GetErrorMessage(compilerResults));
        return;
    }

    // Otherwise store a Type reference to the assembly we compiled
    // Using reflection we can call the Generate method from this Type
    _generatorType = compilerResults.CompiledAssembly.GetType("MusicAlgorithm.AudioGenerator");
}
~~~

By the way, GetErrorMessage() is just an helper method I made to join all the error messages in a single string for output. It is defined like so:

~~~ c#
private static string GetErrorMessage(CompilerResults compilerResults)
{
    StringBuilder errorBuilder = new StringBuilder();
    foreach (CompilerError compilerError in compilerResults.Errors)
        errorBuilder.AppendLine(compilerError.ErrorText);
    return errorBuilder.ToString();
}
~~~

By the end of it, if calling SetAlgorithm(algorithm) did not find any errors on your code, then the Type object "_generatorType" will be pointing at the type of the class you compiled at runtime. Using this Type object, you can simply use Reflection to invoke the Generate method on it:

~~~ c#
private int GetSample()
{
    // Redirects the call to our compiled code if any
    return _generatorType != null ? (int) _generatorType.GetMethod("Generate").Invoke(null, new object[] {_time}) : 0;
}
~~~

And that's all. I was worried that this might turn out to be too slow for real-time usage, but it seems to work okay, at least on my machine.

## Results

Here's a video of the final result as usual, and the source code:

<iframe width="420" height="315" src="//www.youtube.com/embed/vvcALTJf-OQ?rel=0" frameborder="0" allowfullscreen=""></iframe>

[Source Code](https://github.com/davidluzgouveia/blog-music-algorithm)
{: .button}
