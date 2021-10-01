## I Am Graphics And So Can You :: Part 7 :: The Renderer Of Worlds

[Graphics](https://www.fasterthan.life/blog/category/Graphics), [Vulkan](https://www.fasterthan.life/blog/category/Vulkan)

-   ### [PART 6 :: I AM GRAPHICS AND SO CAN YOU :: PIPELINES](https://www.fasterthan.life/blog/2017/7/24/i-am-graphics-and-so-can-you-part-6-pipelines)
    

![krishna.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1504064045620-9OLYR8BJ7ENK6RJ3OUMJ/krishna.jpg?format=750w)

I remembered the line from the pixel scriptures, Vulkan is trying to persuade the programmer that he should do his duty and, to impress him, takes on his multi-threaded form and says, 'Now I am become Graphics, the renderer of worlds.'

We've now come to the end of our journey, yet this is only the beginning.  By now you should feel downright comfortable picking up Vulkan for yourself and wielding it like the supreme being that you are.  To help you along your journey, I've open sourced VkNeo ( now renamed to vkDOOM3. )  You can find it over on [github](https://github.com/DustinHLand/vkDOOM3).  

![Results come from a laptop with an Intel quad core i7 and Quadro M3000M.](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1504060468181-NGQ2PS90XZS2EVAJ1OXH/ogl-vs-vk?format=750w)

Results come from a laptop with an Intel quad core i7 and Quadro M3000M.

Right off the bat we can see the benefits of switching to Vulkan.  Remember we talked about how idTech4 has a renderer frontend RF as well as a backend RB?  Well the RB value above represents all the API specific rendering the engine is performing.  A ~4x improvement isn't a bad start at all.  Let's look at something with a bit more beef.

![GTX 1080 with 4x MSAA.](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1504060890690-7NXRV35MH85QYWX12OCM/1000fps?format=500w)

GTX 1080 with 4x MSAA.

1,000 FPS isn't too bad either.  ( I've heard reports of 2,000 FPS on an RX580 with MSAA off ).  There are of course other systems which can't take this blinding speed, so expect side effects adjusting the com_engineHz cvar.  

![train-speed.gif](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1504064901394-R9NK063KL13KKH82YMTS/train-speed.gif?format=500w)

Now that we're up and running smoothly and on schedule, there's little else to do for this series.  So what are you waiting for?  Get out there and fork/download a copy of vkDOOM3 and godspeed on your journeys into Vulkan and rendering in general.  Thanks for joining me.  Cheers o/ !!!