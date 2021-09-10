## Applying for Entry Level Graphics Jobs in Games

![](http://alextardif.com/images/ESOScreenshot.jpg)  
  

### Overview

When I was applying for graphics jobs out of college, I didn't have a clue what I was doing, or any reference for what might be important in my applications. This guide is for those who want to get their foot in the door in graphics in the game industry, but don't know how, or aren't sure what to focus on. This is not a comprehensive guide to everything you could be doing to get a job, but rather a number of important things that will boost your chances, as well as some general advice. There are plenty of different kinds of graphics jobs out there with completely different applicant considerations, which is why I've limited this to game industry jobs specifically.  
  
  

### Job Listing Requirements in Context

Unless it's a good one, by the end of reading the requirements for a typical graphics job listing, you probably feel like it's looking for you to be:  
-A master of C++  
-A professional mathematician  
-A PhD in light physics  
-A driver engineer for Nvidia  
-An active developer in the Khronos Group  
  
Understand that this is a normal reaction, because on average these listings are asking for an impossible level of knowledge out of an entry candidate (and often beyond that). As an industry we're generally pretty bad at writing position requirements, and things like this can especially hinder a diversity in applicants companies will receive. None of this is the fault of you, the entry candidate, but nonetheless applying for these listings despite the unrealistic requirements is the only way forward until the industry starts adopting better practices for their listings. Apply anyway, and you'll find that much more often than not, studios will not expect you to fit all of their "requirements".  
  
  

### Understand the Graphics Job Listing

Beyond the general requirements above, use the information listed in the requirements as hints for what the position might actually entail. By nature of our industry I suppose, a "graphics programmer" can mean a lot of different things even within just the subset of game development. It can range from an engine programmer all the way to a technical artist. Some might be looking for people to write shaders for FX artists, others include animation in the job responsibilities. It varies wildly. Read over the description a few times to try to get a better feel for what kind of graphics programmer they are looking for, so that you can better target your application for those responsibilities.  
  
  

### Programming Fundamentals

The positions you're applying to tend to have expectations about your general programming ability. For an entry-level graphics candidate, it may surprise you that interviewers might not care as much about how much graphics knowledge you can recite, but foremost that your C++ skills (or whatever their primary language will be) are satisfactory, depending on the position. Graphics teams might expect to need to teach you plenty about graphics on the job, but they may not want to have to teach you C++ on top of it. Make sure you've got the fundamentals of programming with your target language reasonably understood, as it's almost guaranteed to be in your programming tests.  
  
  

### You Don't Need to be a Math Wizard

Something that deters many applicants from applying to entry graphics jobs is the idea that you need to be incredible with math to work in graphics. You don't. What you need in order to get started is a basic understanding of vectors and matrices as they pertain to graphics development. Frankly, if you understand dot and cross products, as well as how matrices are constructed and how to convert coordinates between spaces with matrices, you've got most of what you'll end up using already, unless you're jumping into some kind of research position. If you want to have a strong understanding of math for games and further your preparation, I highly recommend Eric Lengyel's [Foundations of Game Engine Development, Volume 1: Mathematics](https://www.amazon.com/Foundations-Game-Engine-Development-Mathematics/dp/0985811749). It's a great read, and if you take the time to understand it, you'll be plenty prepared as far as math goes.  
  
  

### Solidify Your Familiarity with the Graphics Pipeline

Knowing the different stages of the pipeline, what they do, and how they function can be a great boost to your interviews. You don't have to go crazy deep to get started, but having decent running knowledge of the pieces is great. For this, I recommend Fabian Giesen's fantastic series [A trip through the Graphics Pipeline](https://fgiesen.wordpress.com/2011/07/09/a-trip-through-the-graphics-pipeline-2011-index/). You'll be in great shape after reading and understanding that.  
  
  

### Learn the Basics of a Modern Graphics API

Lately this has been a bit of a controversial topic because some find the modern APIs (D3D12, Vulkan, Metal) to be needlessly complex, to varying degrees depending on the API. Regardless, the industry overall is moving in this direction, and it will strengthen your application if you can demonstrate your work with (or at least be able to talk about) one of them. As a bonus, working with a lower level API will give you an understanding and appreciation for what higher level APIs are doing under the hood for you, which will improve your skills with them as well. I'm not talking about needing to write a full featured renderer yourself, but getting to an understanding of what it takes to render a basic shaded model should cover all the important pieces. If you don't have a preference, I recommend D3D12. Beyond the fact that Xbox and many PC games will use D3D12, Vulkan requires a non trivial amount of additional effort to get up and running, and unless you're developing for Apple products you're never going to touch Metal. I also think the [Microsoft DX12 "Hello World" samples](https://github.com/microsoft/DirectX-Graphics-Samples/tree/master/Samples/Desktop/D3D12HelloWorld/src) and [programming guide](https://docs.microsoft.com/en-us/windows/win32/direct3d12/directx-12-programming-guide) are pretty solid for learning.  
  
  

### Implement a Graphics Paper and Share Results

A lot of what you end up doing in graphics when writing features is reading papers/presentations by researchers, specialists, other graphics programmers, etc, and implementing them in the context of your games. What can go a long way with your application is showing an example (either on your own website, or github, etc) of you taking one of those and implementing it, with a focus on displaying your understanding of how it works. Doing this shows you can take research and execute with it, and also gives you examples for discussion during your interviews. It doesn't have to be some super complex topic.  
  
  

### Try Out a Graphics Debugger

Pick any graphics debugger: PIX, RenderDoc, Intel GPA, Nsight, VS Graphics Debugger, etc, and learn the basics of how to debug a graphics sample with it. This can be your own work, or a public sample like I referenced above. They are a graphics programmer's best friend, and you'll end up using them all the time in your professional work. Understanding a graphics debugger is a great bullet point for a resume, and a good point of discussion in interviews.  
  
  

### Highlight Collaborative Skills

If you have examples of where you were able to work successfully with other people (artists, designers, other programmers) as a programmer, make sure this is visible somewhere on your application and portfolio. Like many areas in games, graphics is a highly collaborative role, and if you can show that you'll be able to work well with artists, it will strengthen your application.  
  
  

### Apply Everywhere You Can

This applies to any game job too but I'll elaborate. This might just be my experience, but it seems to me that the game industry is starved for graphics programmers. There aren't all that many of us to begin with, and I've noticed that we're increasingly competing with film and other software industries for talent. On top of that, companies like Epic and Unity have soaked up so many graphics engineers from the rest of the industry that game studios can be hard-pressed to find experienced graphics programmers. This works to your advantage, because the higher the demand with such a low supply, the higher the chances are that you can find a listing that doesn't include the words "seasoned industry veteran." Look outside the typical game studio hub cities too, there are plenty of companies outside of LA and San Francisco looking for your skills!  
  
  
That's all I've got for now. Hopefully this proves helpful!