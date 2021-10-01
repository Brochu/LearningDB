## I Am Graphics And So Can You :: An Aside :: Motivation & Effort

[Graphics](https://www.fasterthan.life/blog/category/Graphics), [Vulkan](https://www.fasterthan.life/blog/category/Vulkan)

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500163431893-S79L60WLWQU0528XR9PS/image-asset.jpeg?format=750w)

-   [PART 4 :: I AM GRAPHICS AND SO CAN YOU :: RESOURCES RUSH THE STAGE](https://www.fasterthan.life/blog/2017/7/13/i-am-graphics-and-so-can-you-part-4-)
-   [PART 4.5 :: I AM GRAPHICS AND SO CAN YOU :: IDTECH](https://www.fasterthan.life/blog/2017/7/16/i-am-graphics-and-so-can-you-idtech)

Let's actually step away from Vulkan for a bit and talk shop.  If you've made it this far in the series, we've covered a lot of technical ground, but with only a smidge of context on the larger project.  Before we continue talking about pipelines and actual rendering, I want to stop and impart some wisdom to you.  You see implementing a graphics API does not a renderer make.  I know this might feel a bit like finding out the great and powerful Oz is just a man behind the curtain, and frankly that's because that's exactly what it is.  Vulkan, OpenGL, Metal, GNM, DirectX are serious technologies, but they are only a component of an entire technology stack.  Let's take a look at some more data on this.

## Renderer: Lines of Code

FrontEnd is the term used for API agnostic render code.  As you can see, even the amount of OpenGL code outpaces the Vulkan renderer's.  ( Granted not everything is implemented Vulkan side, but it would be comparable ).  

DOOM 3 BFG shipped on PC ( OpenGL ), PS3 ( GCM ), Xbox 360 ( DirectX ).  That sounds like a lot of work right?  Well frankly it is.  And I don't say that to discourage you.  Rather I say that to encourage you.  ( It's natural so don't personalize it ) You can spend a lot of time on these technologies and see little to no visible progress.  This can be very discouraging, especially to newcomers.  On top of that, the deluge of terms and options overwhelms people and they turn away.  BUT, if you've made it this far, you've seen that at the core of these concepts, things can be rather simple.  It's just a lot of little things glued together to make something of value.  

And frankly, value is what you should be focused on producing.  If you're getting into graphics ( or games in general ) , you're wanting to show people things.  You're giving people an avenue to express their work, enjoy someone else's, to show them something they haven't seen.  The catharsis of shipping something can be immeasurably rewarding.  That's why people subject themselves to understanding this complexity.  

So don't give up on yourself.  We often think one person ( yourself ) has to do everything.  And that's just simply not the case anymore.  Our work is built on top of those that came before us, or those fighting beside us.  You should be seeking out others with similar passions.  Grow better together.  Learn from each other.  Make something of value together.  

Let's look at some programmer credits real quick.

-   Original DOOM 3
    -   John Carmack
    -   Timothee Besset
    -   Jim Dose
    -   Robert Duffy
    -   Jan Paul van Waveren
    -   Jonathan Wright
-   BFG Edition
    -   Brian Harris
    -   Curtis Arink
    -   Jeff Farrand
    -   Ryan Gerleve
    -   Billy Khan
    -   Gloria Kennickell
    -   Mike Maynard
    -   John Roberts
    -   Steven Serafin

All these people put an extraordinary amount of effort into idTech4.x.  JP's special PDA thanks at the end of the game always sticks with me.

![pda-secret.JPG](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500186458780-7VMJV996RX54H4T7OGRY/pda-secret.JPG?format=1000w)

Stripped for spare parts indeed.  It can most certainly feel that way.  ( Rest in peace JP )

These people dedicated years of their lives to making idTech run.  And this is the reality of working with these technologies.  It takes time.  It takes sweat.  It takes long nights ( sometimes ).  The God mentality is false ( see [There Are No Gods](https://www.fasterthan.life/blog/2017/6/25/there-are-no-gods) ).  You have to be willing to put in the effort.  Results didn't come immediately to any of the names above, and they most certainly won't come immediately to you.  

If you undervalue yourself, you might think it's hopeless.  But in reality, you're probably smart enough to understand everything I'm going over in this series.  Even a functional understanding is great if it leads you to produce something of value.  You don't have to be an expert to make an impact.  Are you willing to put in the effort?  Will you be one of those that sticks it out?  

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500187196474-EU1TU9KAO9N3XPH7MZEE/image-asset.jpeg?format=750w)

Ok done with being serious.  *Claps and rubs hands together in anticipation*. ( Back to bad memes and dad jokes )  Next ( in Part 4.5 ) we'll actually talk a bit about how exactly idTech4.5 and DOOM 3 BFG leverage the new Vulkan backend.  This will introduce several rendering concepts that'll be crucial to your understanding when proceeding further with Vulkan as an API.  See you there!