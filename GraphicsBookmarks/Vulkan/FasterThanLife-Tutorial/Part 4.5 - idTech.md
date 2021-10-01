## I Am Graphics And So Can You :: Part 4.5 :: idTech

[Graphics](https://www.fasterthan.life/blog/category/Graphics), [Vulkan](https://www.fasterthan.life/blog/category/Vulkan)

![We can all relate](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500345354579-O71GU8D5145MUAWH3IFC/image-asset.jpeg?format=750w)

We can all relate

-   ### [AN ASIDE :: I AM GRAPHICS AND SO CAN YOU :: MOTIVATION & EFFORT](https://www.fasterthan.life/blog/2017/7/15/i-am-graphics-and-so-can-you-effort)
    
-   ### [PART 5 :: I AM GRAPHICS AND SO CAN YOU :: YOUR PIXELS ARE SERVED](https://www.fasterthan.life/blog/2017/7/22/i-am-graphics-and-so-can-you-part-5-your-pixels-are-served)
    

As mentioned in my "Motivation & Effort" post, idTech is a lot of code and the rendering APIs are only a component of that ( a very important component though ).  All this rendering code can make little sense without some context on how it's being used.  So let's see how exactly idTech goes about invoking the magic.  

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500352176066-QCM6J8T7W6PTZTW0ZU7A/image-asset.jpeg?format=750w)

idTech4.x maintains the idea of both a frontend and backend renderer.  Most of everything we've covered so far, and will cover going forward deals with the backend.  The two roles could be summed up as this.

-   Frontend - Determine what's visible to draw.
-   Backend - Draw what the frontend tells us is visible.

Here's a run down of the typical frontend callstack.

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500352693644-Y4EXPT5P43K6CQF7SO87/image-asset.jpeg?format=750w)

The frontend actually builds a list of render commands which it then passes as a linked list to the backend's ExecuteBackendCommands.  When I say render commands, it's nothing glorious.  These are high level directives.  Here all all the commands.

-   RC_NOP - Do nothing
-   RC_DRAW_VIEW_3D
-   RC_DRAW_VIEW
-   RC_COPY_RENDER
-   RC_POST_PROCESS

/*
=============
idRenderBackend::ExecuteBackEndCommands

This function will be called syncronously if running without
smp extensions, or asyncronously by another thread.
=============
*/
void idRenderBackend::ExecuteBackEndCommands( const renderCommand_t *cmds ) {
	CheckCVars();

	resolutionScale.SetCurrentGPUFrameTime( commonLocal.GetRendererGPUMicroseconds() );

	if ( cmds->commandId == RC_NOP && !cmds->next ) {
		return;
	}

	ResizeImages();

	renderLog.StartFrame();
	GL_StartFrame();

	uint64 backEndStartTime = Sys_Microseconds();

	GL_SetDefaultState();

	for ( ; cmds != NULL; cmds = (const renderCommand_t *)cmds->next ) {
		switch ( cmds->commandId ) {
			case RC_NOP:
				break;
			case RC_DRAW_VIEW_3D:
			case RC_DRAW_VIEW_GUI:
				DrawView( cmds );
				break;
			case RC_COPY_RENDER:
				CopyRender( cmds );
				break;
			case RC_POST_PROCESS:
				PostProcess( cmds );
				break;
			default:
				idLib::Error( "ExecuteBackEndCommands: bad commandId" );
				break;
		}
	}

	GL_EndFrame();

	m_pc.totalMicroSec = Sys_Microseconds() - backEndStartTime;

	renderLog.EndFrame();
}

Frankly, everything but DrawView is fairly trivial.  But now we're starting to see GL_* calls.  Anytime you see this in idTech, it signifies a wrapper for some graphics API functionality.  So we're close to Vulkan!  Before we get ahead of ourselves though, let's further cement the relationship between the frontend and backend.  

/*
===========================================================================

idRenderBackend

all state modified by the back end is separated from the front end state

===========================================================================
*/
class idRenderBackend {
public:
	idRenderBackend();
	~idRenderBackend();

	// Setup things ( See Part 3 of the series "The First 1,000" )
	void Init();
	// Tear everything down
	void Shutdown();

	// What I just showed you.
	void ExecuteBackEndCommands( const renderCommand_t *cmds );
	// Swap out the current presented image from the swapchain for the next.
	void BlockingSwapBuffers();

	// Just prints a bunch of debug info
	void Print();

That's it!  That's the separation.  One of the big efforts in VkNeo was actually to corral all the OpenGL code into something with an API contract that could be suitable to implementing Vulkan ( Without losing existing functionality.  )  This resulted in the idRenderBackend where there's a clear handoff of the data and responsibility.   

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500522392357-QNTWRKA7BEFLVDLO9JC8/image-asset.gif?format=500w)

Now let's look at that DrawView call shall we?  Buckle up, the rabbit hole gets a bit deep.  Note that this is idTech4.5 code with some slight modifications for VkNeo.  You can already find the full OpenGL only source on [GitHub](https://github.com/id-Software/DOOM-3-BFG).  

# DrawView

This particular call accounts for ~99% of what you see in DOOM 3 BFG.  The source has been distilled down for brevity, while capturing the core functionality.

/*
==================
idRenderBackend::DrawView
==================
*/
void idRenderBackend::DrawView( const void * data ) {
	const drawSurfsCommand_t * cmd = (const drawSurfsCommand_t *)data;

	m_viewDef = cmd->viewDef;

	// render the scene
	{
		drawSurf_t **drawSurfs = (drawSurf_t **)&m_viewDef->drawSurfs[0];
		const int numDrawSurfs = m_viewDef->numDrawSurfs;

		//-------------------------------------------------
		// RB_BeginDrawingView
		//
		// Any mirrored or portaled views have already been drawn, so prepare
		// to actually render the visible surfaces for this view
		//
		// clear the z buffer, set the projection matrix, etc
		//-------------------------------------------------

		// set the window clipping
		GL_Viewport( m_viewDef->viewport.x1,
			m_viewDef->viewport.y1,
			m_viewDef->viewport.x2 + 1 - m_viewDef->viewport.x1,
			m_viewDef->viewport.y2 + 1 - m_viewDef->viewport.y1 );

		// the scissor may be smaller than the viewport for subviews
		GL_Scissor( m_viewDef->viewport.x1 + m_viewDef->scissor.x1,
					m_viewDef->viewport.y1 + m_viewDef->scissor.y1,
					m_viewDef->scissor.x2 + 1 - m_viewDef->scissor.x1,
					m_viewDef->scissor.y2 + 1 - m_viewDef->scissor.y1 );
		m_currentScissor = m_viewDef->scissor;

		// ensures that depth writes are enabled for the depth clear
		GL_State( GLS_DEFAULT | GLS_CULL_FRONTSIDED, true );

		// Clear the depth buffer and clear the stencil to 128 for stencil shadows as well as gui masking
		GL_Clear( false, true, true, STENCIL_SHADOW_TEST_VALUE, 0.0f, 0.0f, 0.0f, 0.0f );

		//------------------------------------
		// sets variables that can be used by all programs
		//------------------------------------
		{
			//
			// set eye position in global space
			//
			float parm[4];
			parm[0] = m_viewDef->renderView.vieworg[0];
			parm[1] = m_viewDef->renderView.vieworg[1];
			parm[2] = m_viewDef->renderView.vieworg[2];
			parm[3] = 1.0f;

			renderProgManager.SetRenderParm( RENDERPARM_GLOBALEYEPOS, parm ); // rpGlobalEyePos

			// sets overbright to make world brighter
			// This value is baked into the specularScale and diffuseScale values so
			// the interaction programs don't need to perform the extra multiply,
			// but any other renderprogs that want to obey the brightness value
			// can reference this.
			float overbright = r_lightScale.GetFloat() * 0.5f;
			parm[0] = overbright;
			parm[1] = overbright;
			parm[2] = overbright;
			parm[3] = overbright;
			renderProgManager.SetRenderParm( RENDERPARM_OVERBRIGHT, parm );

			// Set Projection Matrix
			float projMatrixTranspose[16];
			R_MatrixTranspose( m_viewDef->projectionMatrix, projMatrixTranspose );
			renderProgManager.SetRenderParms( RENDERPARM_PROJMATRIX_X, projMatrixTranspose, 4 );
		}

		//-------------------------------------------------
		// fill the depth buffer and clear color buffer to black except on subviews
		//-------------------------------------------------
		FillDepthBufferFast( drawSurfs, numDrawSurfs );

		//-------------------------------------------------
		// main light renderer
		//-------------------------------------------------
		DrawInteractions();

		//-------------------------------------------------
		// now draw any non-light dependent shading passes
		//-------------------------------------------------
		DrawShaderPasses( drawSurfs, numDrawSurfs );

		//-------------------------------------------------
		// Further processing involves full screen 
		// post processing and was removed for brevity
		//-------------------------------------------------
        }
}

So let's break down what this function is doing at a high level.

### ESTABLISH VIEWPORT

Basically the rectangle the view is being rendered into.  This is your game window or full screen display.  Let's look at our screen ( shot ).  We'll be looking at the man in the mirror today.

View fullsize

![Click to Enlarge](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500607772549-HXPQCVG3FW2O8ULBAWD3/image-asset.jpeg?format=750w)

Click to Enlarge

### SETUP SCISSOR

Just like cutting paper for a craft, the renderer can cut out a specific part of the viewport to render to.  All operations outside this region are disregarded.  In our case, mirrors are implemented via sub views.  ( DrawView gets called for what we see in the mirror, and DrawView gets called for player's view of the mirror. )  The GL_Scissor for the subview looks like this.

View fullsize

![Click to Enlarge](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500608082929-PYWXHVTWVL3VY8Q7ZI3J/image-asset.jpeg?format=750w)

Click to Enlarge

### GL_STATE

This toggles all the switches and turns all the knobs fed to it.  This is probably the most contentious part between Vulkan and OpenGL for VkNeo.  There are 64 bits of state ( uint64 ) in total telling the renderer to do different things.  These bits are renderer agnostic.  ( Item [ bits ] )

-   Src & Dst Blend Functions [ 0 - 5 ]
-   Depth Mask [ 6 ]
-   Color Masks [ 7 - 10 ]
-   Polygon Mode [ 11 ]
-   Polygon Offset [ 12 ]
-   Depth Functions [ 13 - 14 ]
-   Cull Mode [ 15 - 16 ]
-   Blend Op [ 17 - 19 ]
-   Stencil State [ 20 - 47 ]
-   Alpha Test State [ 48 - 57 ] - No longer used
-   Depth Test Mask [ 58 ]
-   Front Face [ 59 ]
-   Separate Stencil [ 60 ]
-   Mirror View [ 61 ]

Ok, I lied, Neo doesn't used all 64 bits.  But it is getting crowded in there.  Each of these items can be a separate post, but for now it's safe to just know they're all wrapped up in a nice little GL_State function that idTech4.5 uses.  

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500611172304-92WKCX3MZDRWZ2ISIHAC/image-asset.gif?format=500w)

### GL_CLEAR

!["My first" rendering operation.](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500612063170-X7JMHS922HKEE5S96X13/image-asset.jpeg?format=500w)

"My first" rendering operation.

### RENDERPARMS

Next we run into setting some render parms.  Anytime you see RENDERPARM_ you can bank on this showing up in a shader's UBO ( uniform buffer ).   Essentially whenever a draw is called in idTech4.5 renderparms relevant to the shader(s) being used are scooped up into a uniform buffer the shader(s) can use.  For more information on UBOs, see [Part 4](https://www.fasterthan.life/blog/2017/7/13/i-am-graphics-and-so-can-you-part-4-).

### FILLDEPTHBUFFERFAST

I mean everything is better fast right?  Well some things.  Anyways, yes depth buffers.  To people with prior graphics experience this is old news.  To someone new to graphics, depth buffers can be something hard to grasp.  But I think often that's because they're not introduced to it in an intuitive way.  Most people think of images as 2D flat surfaces.  But in rendering, you need to cast aside those assumptions.  Let's start with the basics, 0 to 1.  

If you remember in Part 3 CreateRenderTargets we setup a depth image with VK_FORMAT_D32_SFLOAT_S8_UINT.  What this means is that depth and stencil share the same image, but we can ignore stencil for now.  The depth part is composed of 32 bits encoded as a signed float.  These 32 bits are stretched over the values of zero-to-one.  Now, we're going to switch contexts on how we think of this image a couple times.  

First let's think of it as color.  What is 0?  In graphics it's black.  It's the absence of any color.  What is 1?  It's white as the color channel(s) are fully saturated.  This is actually how physics works as well.  Things appear black because little to no light is reaching our eyes from that object, whereas white is a large mixture of different frequencies.  

But we're not trying to represent color data, we're trying to represent depth data.  ie how far away is something?  We use the 0-1 range to encode this distance.  ( See images can be more than just things to look at.  )  Let's walk through a few draw calls and look at what FillDepthBufferFast produces for us. ( We'll start with final image then first and work back to final.  Keep Clicking!!! )

![d12.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500616677272-WP4LDRREROLHJXC77C5V/d12.jpg?format=1000w)

![d4.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500616675704-RRC0PF9W6U41GV1QA7HC/d4.jpg?format=1000w)

![d5.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500616675751-NJKLMPHORJZ62W3VG1UE/d5.jpg?format=1000w)

![d6.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500616676256-IBQHJW508EWGPIP5J9RF/d6.jpg?format=1000w)

![d7.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500616676376-M8676YYIYP69A8BT5O3A/d7.jpg?format=1000w)

![d8.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500616676548-9SM8NRC7F6P9FPSDZ87M/d8.jpg?format=1000w)

![d9.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500616676718-QJGE7TQMM3EEUX7VLHOS/d9.jpg?format=1000w)

![d10.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500616676891-9CPBR0NQJ2TLDRPA7LY5/d10.jpg?format=1000w)

![d11.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500616677068-5VIRKLTSPOZMLK0R9UND/d11.jpg?format=1000w)

Cool huh?  But what value is it?  Well think of it in natural terms?  If you can't see the zombie behind the bathroom wall, does it exist?  ( Not talking Copenhagen Interpretation or anything ).  What I mean is, does it or should it exist to the renderer?  Why waste time on drawing it, if the player can't see it.  This is what depth helps you determine.  Let's look at one final example to further cement things.  I'll use a simpler depth image to demonstrate ( this time of the actual player looking at the mirror. ).  

![d3-modified.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500619142312-IBF2EXRPYRBZDNKKM09Z/d3-modified.jpg?format=750w)

If we took just this slice of what was visible and shot rays at it, how far would they reach?  It would look something like this graph.

![chartz.JPG](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500621290934-YH2UA5CJ2GSMFS7SHXWW/chartz.JPG?format=750w)

Anything in the white space of the graph essentially wouldn't be rendered.  See, not so magical after all right?  Any concept, once you dig deep down, has a simple truth.  The hardest thing to learn often is "how" to think about things; not the "what".

### DRAWINTERACTIONS

So right now our color buffer looks like this.

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500697215115-L194EUA5NYYPEQHJER9F/image-asset.jpeg?format=750w)

Not very pretty right?  This is where we start introducing light into the scene.  DrawInteractions goes over every light that touches ANYTHING visible in the scene and checks them for performing a series of actions.

-   Global Light Shadows -> StencilShadowPass
-   Local Light Interactions -> RenderInteractions
-   Local Light Shadows -> StencilShadowPass
-   Global Light Interactions -> RenderInteractions 
-   Translucent Interactions -> RenderInteractions

I'll touch on stencil shadows at some other point or we'll just be here all night.  People have a natural intuition of how light reveals things.  So let's see how the scene gets rendered in steps.  Click through the following gallery.  Note remember that this is all driven by lights in the scene.  Nothing is drawn unless it is being lit.

![c1.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500699633274-2BBIH1QJTN4SKBX98S79/c1.jpg?format=1000w)

![c2.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500699633305-2D9WY24MM1FO9YKVPR3Q/c2.jpg?format=1000w)

![c3.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500699633740-LTCILQ97OKA83L7RX4TV/c3.jpg?format=1000w)

![c4.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500699634054-292O7AW0IAW9QVB14K0S/c4.jpg?format=1000w)

![c5.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500699634441-VIKMCB1DSUJ7V5AXWYQC/c5.jpg?format=1000w)

![c6.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500699634974-UID4E9UYLX7D7GQ3PQQ7/c6.jpg?format=1000w)

![c7.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500699635467-11GWCDBZZBT37C4KNC97/c7.jpg?format=1000w)

![c8.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500699636189-3FBYJIAJ4CYFM6R763KL/c8.jpg?format=1000w)

![c9.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500699636428-FNFOWK0PFX7J1GA13K2U/c9.jpg?format=1000w)

![c10.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500699637275-OKJUV26VLZ1ANR099J45/c10.jpg?format=1000w)

![c11.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500699637688-TKOAA6C7NJZQXIQJ9ZAY/c11.jpg?format=1000w)

![c13.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500699638907-31YNPTVJLW4268U6TM24/c13.jpg?format=1000w)

### DRAWSHADERPASSES

Last but not least, we put on the final polish.  These draws are not related to the lighting environment so the shader passes happen outside of DrawInteractions.  Click through this gallery to see that coat of paint go on.

![c1.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500700084132-S028976QAEA10BZ72MSN/c1.jpg?format=1000w)

![c2.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500700084110-BEFTHM6PUU2XDRP55HP0/c2.jpg?format=1000w)

![c3.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500700085545-STWYW18I17P2CEI55MKM/c3.jpg?format=1000w)

![c4.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500700085787-S8HUDTMH5CFQYCVUD8RJ/c4.jpg?format=1000w)

![c5.jpg](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500700086568-TQ0NJTBQQOFS356DP92B/c5.jpg?format=1000w)

And that's it!  With these 3x procedures we've produced > 90% of what people would consider the visible game.  But, But, But.... you might be wondering why I spent so much time the depth buffer section and not the color ones.  Very astute observation Watson!

# "You know my method. It is founded upon the observation of trifles."

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500701170090-48PZEVRXBT4FKZHLBHPO/image-asset.jpeg?format=750w)

Regardless of how splendid the end result, the core functionality producing it can be and is rather quite simple ( elementary one might say ).  Often the complexity is in the data, not necessarily the code.  This is another major deterrent to rendering for some people.  It's not the API, or the systems, or architecture, or hardware.  It's the components of data and how they relate to one another.  That sea of entropy has swallowed many a soul.  But you can weather the storm with a prepared mind.  Remember the difficulty isn't the facts, but the way in which you think about them.

So let's think about what's going on.  How was this scene produced?  There are a lot of directions we could take this; going down all sorts of rabbit holes.  But we're here to learn Vulkan remember?  ( oh yeah that.  And I'd have to start charging tuition to cover everything else )  Essentially it comes down to collecting enough details to submit for a draw call.  This includes things such as...

-   GLState bits mentioned before.
-   Surfaces -> vertices to draw
-   Lights affecting the surface
-   Textures associated with the draw
-   Render parms associated with the draw.

That can seem like an impossible symphony to orchestrate.  But wait, that's the whole reason I started off this series about [Inuition in Part 2](https://www.fasterthan.life/blog/2017/7/11/i-am-graphics-and-so-can-you-part-2-intuition).  Each draw call just goes down the graphical pipeline and emerges out the other side as a series of pixels.  So the details we collect inform the pipeline how to behave.  ( I go through each stage of the pipeline in Part 2 from a high level so I won't reiterate that here ).  So before we go, let's look at one more thing that demonstrates all of this.  Then in the next article we'll take the plunge into drawing with Vulkan.

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500703287415-N25A2RVSN5N2XL7OE39T/image-asset.jpeg?format=1000w)

We'll look at DrawSingleInteraction.  This is just a few steps down the trail from DrawInteractions we looked at above.  In it you'll notice what I'm talking about regarding gathering data.  The frontend's job is to take the representation of the world, and break it into a collection of surfaces.  The backend then looks at these surfaces, the associated entities, and the lights in view and just steps through each surface 1-by-1 calling draw on it.  

struct drawSurf_t {
    const srfTriangles_t *   frontEndGeo;        // don't use on the back end, it may be updated by the front end!
    int                      numIndexes;
    vertCacheHandle_t        indexCache;         // triIndex_t
    vertCacheHandle_t        ambientCache;       // idDrawVert
    vertCacheHandle_t        shadowCache;        // idShadowVert / idShadowVertSkinned
    vertCacheHandle_t        jointCache;         // idJointMat
    const viewEntity_t *     space;
    const idMaterial *       material;           // may be NULL for shadow volumes
    uint64                   extraGLState;       // Extra GL state |'d with material->stage[].drawStateBits
    float                    sort;               // material->sort, modified by gui / entity sort offsets
    const float     *        shaderRegisters;    // evaluated and adjusted for referenceShaders
    drawSurf_t *             nextOnLight;        // viewLight chains
    drawSurf_t **            linkChain;          // defer linking to lights to a serial section to avoid a mutex
    idScreenRect             scissorRect;        // for scissor clipping, local inside renderView viewport
    int                      renderZFail;
    volatile shadowVolumeState_t shadowVolumeState;
};

struct drawInteraction_t {
    const drawSurf_t *   surf;

    idImage *            bumpImage;
    idImage *            diffuseImage;
    idImage *            specularImage;

    idVec4               diffuseColor;    // may have a light color baked into it
    idVec4               specularColor;   // may have a light color baked into it
    stageVertexColor_t   vertexColor;     // applies to both diffuse and specular

    int                  ambientLight;    // use tr.ambientNormalMap instead of normalization cube map 

    // these are loaded into the vertex program
    idVec4               bumpMatrix[2];
    idVec4               diffuseMatrix[2];
    idVec4               specularMatrix[2];
};

/*
=================
idRenderBackend::DrawSingleInteraction
=================
*/
void idRenderBackend::DrawSingleInteraction( drawInteraction_t * din ) {
	// bump matrix
	renderProgManager.SetRenderParm( RENDERPARM_BUMPMATRIX_S, din->bumpMatrix[0].ToFloatPtr() );
	renderProgManager.SetRenderParm( RENDERPARM_BUMPMATRIX_T, din->bumpMatrix[1].ToFloatPtr() );

	// diffuse matrix
	renderProgManager.SetRenderParm( RENDERPARM_DIFFUSEMATRIX_S, din->diffuseMatrix[0].ToFloatPtr() );
	renderProgManager.SetRenderParm( RENDERPARM_DIFFUSEMATRIX_T, din->diffuseMatrix[1].ToFloatPtr() );

	// specular matrix
	renderProgManager.SetRenderParm( RENDERPARM_SPECULARMATRIX_S, din->specularMatrix[0].ToFloatPtr() );
	renderProgManager.SetRenderParm( RENDERPARM_SPECULARMATRIX_T, din->specularMatrix[1].ToFloatPtr() );

	RB_SetVertexColorParms( din->vertexColor );

	renderProgManager.SetRenderParm( RENDERPARM_DIFFUSEMODIFIER, din->diffuseColor.ToFloatPtr() );
	renderProgManager.SetRenderParm( RENDERPARM_SPECULARMODIFIER, din->specularColor.ToFloatPtr() );

	// texture 0 will be the per-surface bump map
 	GL_SelectTexture( INTERACTION_TEXUNIT_BUMP );
	GL_BindTexture( din->bumpImage );

	// texture 3 is the per-surface diffuse map
	GL_SelectTexture( INTERACTION_TEXUNIT_DIFFUSE );
	GL_BindTexture( din->diffuseImage );

	// texture 4 is the per-surface specular map
	GL_SelectTexture( INTERACTION_TEXUNIT_SPECULAR );
	GL_BindTexture( din->specularImage );

	DrawElementsWithCounters( din->surf );
}

DrawElementsWithCounters( surf ) is where this train stops.  But what is a surface you might ask?  Here are a few examples. ( click, click, click, click, )

![surf-1.JPG](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500706203271-SKVHXXDOCRQM2MDEQUQD/surf-1.JPG?format=750w)

![surf-2.JPG](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500706203311-KY537UL28UD31HNE4II3/surf-2.JPG?format=750w)

![surf-3.JPG](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500706203508-L8948HZ5KGYUN8UM06PB/surf-3.JPG?format=750w)

![surf-4.JPG](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1500706203638-3PD21TJ3LPGXQT6B8OUL/surf-4.JPG?format=750w)

See not so unfamiliar after all?  It can all seem alien, but it's just a familiar face with a different guise.