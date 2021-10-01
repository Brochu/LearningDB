## I Am Graphics And So Can You :: Part 1 :: Intro

[Graphics](https://www.fasterthan.life/blog/category/Graphics), [Vulkan](https://www.fasterthan.life/blog/category/Vulkan)

( Note this is the beginning of an on going series diving into Vulkan, graphics programming, and how you can gain mastery of the subject.  So stay tuned! My target audience is mainly non experts, people that want to get into graphics or want to learn Vulkan. )

-   ### [PART 2:: I AM GRAPHICS AND SO CAN YOU :: INTUITION](https://www.fasterthan.life/blog/2017/7/11/i-am-graphics-and-so-can-you-part-2-intuition)
    

Hi, I'm a senior programmer, whatever that means. Twelve years coding, and I never once touched graphics. Sure I may have fixed a crash in some rendering code. Sure I played around with open cv for picking out image features. Sure I picked up an OpenGL bible and went through all the examples. But did I intuitively understand how graphics worked? Not really. 

I'm sure my story resonates with many of you reading this. No matter how many times you bang your head against the wall it just doesn't cave. So you go back to the safety of what you know and leave graphics to more "experienced" hands.  After all, the race has been run. The winners decided. How will you ever acquire all the arcane knowledge you need to be a "graphics programmer"?

First off, let's address the elephant in the room. Yes graphics programmers have an elevated status in the games industry. If you want to know my thoughts on this matter see my post ["There Are No Gods"](https://www.fasterthan.life/blog/2017/6/25/there-are-no-gods). You can't focus on them, only yourself. You have to ask why you're wanting to learn graphics. Be honest with yourself, and take yourself where you are. Approach learning it like you would anything else. Celebrate your victories, even if they can't compare to "those peoples'" work. That's not the point. In the end if you don't enjoy it, then don't force yourself. There are plenty of other problem spaces for you to get involved with in the gaming world. 

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1499827741071-PYJAMQTUMOYLR0SZB3UF/image-asset.jpeg?format=500w)

Still here?  Good. So how did I finally break through that wall of understanding?  I'll give you a hint. It begins with a V and ends with ulkan. "But Vulkan is the hardz" I hear you saying. "Shouldn't I start with something easy like OpenGL?"  Emphatically I say NO. 

OpenGL is a bloated mountain to climb. It's a mystical land where you offer prayers to priestesses on high in hopes of the sky changing color. Vulkan is the government office telling you, you still don't have all the necessary forms to proceed. ( Maybe my analogies made OpenGL too attractive ).

The point is that Vulkan removes the mystery.  It spells things out, plain as day. And for a programmer, this is awesome!  It's intellectually honest about what it needs, and what it's doing. It will teach you what it really means to program a GPU. Don't think of all the extra code as a mountain to climb, but instead as a clearly articulated set of instructions to help you get where you need to go. 

Added bonus? It's still new, and people are wrapping their heads around it. There's opportunity. That's what spurred me to try again. And this time I broke through. How did I do that?  I challenged myself with writing a Vulkan renderer for the open source version of DOOM 3 BFG.  And four full time months of work later I have a working renderer. ( I call it **VkNeo** after the project code name )

So to help more people like me, I'd like to walk you through what I did. I'll teach you Vulkan, **because I am graphics and so can you**.  

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1499830272560-I1INIY9SUQ5ST07AVESL/image-asset.jpeg?format=750w)