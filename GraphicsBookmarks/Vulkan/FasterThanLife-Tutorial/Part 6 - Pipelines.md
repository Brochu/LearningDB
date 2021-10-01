## I Am Graphics And So Can You :: Part 6 :: Pipelines

[Graphics](https://www.fasterthan.life/blog/category/Graphics), [Vulkan](https://www.fasterthan.life/blog/category/Vulkan)

![I'm here to lay some pipelines. &nbsp;*Derp*](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1501035557565-BFSS1H177MV08Y56LPGT/Anchorman_flute.jpg?format=750w)

I'm here to lay some pipelines.  *Derp*

It was Vulkan pipelines that really made things click for me.  Everything seemed so disassociated before.  I knew what shaders did, I knew what data the GPU needed, and I knew what the individual pipeline functions were supposed to do.  But I had so many questions.  When was it ok to do X?  When should I call Y function?  Was this the GPU's responsibility or mine?  Then I saw VkGraphicsPipelineCreateInfo and did a triple take.  Who in their right mind would ever want to touch that thing?  And then I heard how people were front loading pipeline compiles so they weren't happening intra frame.  And then it dawned on me.  It all fell into place.  I started to see graphics as a cohesive process instead of a functional free-for-all.  And now I wanted to touch VkGraphicsPipelineCreateInfo.  

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1501035614113-BM6Z35ZCQ36TXXT10OC3/image-asset.png?format=750w)

Over the series I've taken you on a grand tour of Vulkan, but I've always shied away from going into idRenderProgManager.  This was for a very good reason.  Pipelines are at the core of what Vulkan and graphics are all about.  They tell the GPU what to do with draw commands from the beginning to the end.  In [Part 5](https://www.fasterthan.life/blog/2017/7/22/i-am-graphics-and-so-can-you-part-5-your-pixels-are-served) if you remember, there was a renderProgManager.CommitCurrent call right before the draw.  This is because we're committing the final pipeline state that will inform the GPU how to draw what is being submitted in the subsequent draw call.  

Let's start back at the beginning and catch up to present.  Initialize()!

/*
========================
idRenderProgManager::Init
========================
*/
void idRenderProgManager::Init() {
	idLib::Printf( "----- Initializing Render Shaders -----\n" );

	// Builtin shaders descriptions
	// 1.) Explicit enum used for binding in draw routines.
	// 2.) Name of shader
	// 3.) Shader stage [ VERTEX, FRAGMENT, ALL ]
	// 4.) Vertex layout [ DRAW_VERT, DRAW_SHADOW_VERT, DRAW_SHADOW_VERT_SKINNED ]
	struct builtinShaders_t {
		int index;
		const char * name;
		rpStage_t stages;
		vertexLayoutType_t layout;
	} builtins[ MAX_BUILTINS ] = {
		{ BUILTIN_GUI, "gui", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_COLOR, "color", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_SIMPLESHADE, "simpleshade", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_TEXTURED, "texture", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_TEXTURE_VERTEXCOLOR, "texture_color", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_TEXTURE_VERTEXCOLOR_SKINNED, "texture_color_skinned", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_TEXTURE_TEXGEN_VERTEXCOLOR, "texture_color_texgen", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_INTERACTION, "interaction", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_INTERACTION_SKINNED, "interaction_skinned", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_INTERACTION_AMBIENT, "interactionAmbient", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_INTERACTION_AMBIENT_SKINNED, "interactionAmbient_skinned", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_ENVIRONMENT, "environment", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_ENVIRONMENT_SKINNED, "environment_skinned", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_BUMPY_ENVIRONMENT, "bumpyEnvironment", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_BUMPY_ENVIRONMENT_SKINNED, "bumpyEnvironment_skinned", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },

		{ BUILTIN_DEPTH, "depth", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_DEPTH_SKINNED, "depth_skinned", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_SHADOW, "shadow", SHADER_STAGE_VERTEX, LAYOUT_DRAW_SHADOW_VERT },
		{ BUILTIN_SHADOW_SKINNED, "shadow_skinned", SHADER_STAGE_VERTEX, LAYOUT_DRAW_SHADOW_VERT_SKINNED },
		{ BUILTIN_SHADOW_DEBUG, "shadowDebug", SHADER_STAGE_ALL, LAYOUT_DRAW_SHADOW_VERT },
		{ BUILTIN_SHADOW_DEBUG_SKINNED, "shadowDebug_skinned", SHADER_STAGE_ALL, LAYOUT_DRAW_SHADOW_VERT_SKINNED },

		{ BUILTIN_BLENDLIGHT, "blendlight", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_FOG, "fog", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_FOG_SKINNED, "fog_skinned", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_SKYBOX, "skybox", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_WOBBLESKY, "wobblesky", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_POSTPROCESS, "postprocess", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_BINK, "bink", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_BINK_GUI, "bink_gui", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
		{ BUILTIN_MOTION_BLUR, "motionBlur", SHADER_STAGE_ALL, LAYOUT_DRAW_VERT },
	};
	m_renderProgs.SetNum( MAX_BUILTINS );
	
	// Go over all the builtin descriptions and build renderProg_t's for each.
	for ( int i = 0; i < MAX_BUILTINS; i++ ) {
		
		// FindShader will load and compile the shader module.
		// It also loads the shader's layout file for building descriptor sets for the pipeline.

		// Vertex shader
		int vIndex = -1;
		if ( builtins[ i ].stages & SHADER_STAGE_VERTEX ) {
			vIndex = FindShader( builtins[ i ].name, SHADER_STAGE_VERTEX );
		}

		// Optional fragment shader
		int fIndex = -1;
		if ( builtins[ i ].stages & SHADER_STAGE_FRAGMENT ) {
			fIndex = FindShader( builtins[ i ].name, SHADER_STAGE_FRAGMENT );
		}
		
		// Fill out the renderProg_t
		renderProg_t & prog = m_renderProgs[ i ];
		prog.name = builtins[ i ].name;
		prog.vertexShaderIndex = vIndex;
		prog.fragmentShaderIndex = fIndex;
		prog.vertexLayoutType = builtins[ i ].layout;

		// Now that all the shaders are loaded, go ahead and create the descriptor set layout.
		CreateDescriptorSetLayout( 
			m_shaders[ vIndex ],
			( fIndex > -1 ) ? m_shaders[ fIndex ] : defaultShader,
			prog );
	}

	// Initialize all render parm values to vec4_zero
	m_uniforms.SetNum( RENDERPARM_TOTAL, vec4_zero );

	// Mark all the skinned shaders as using joints
	m_renderProgs[ BUILTIN_TEXTURE_VERTEXCOLOR_SKINNED ].usesJoints = true;
	m_renderProgs[ BUILTIN_INTERACTION_SKINNED ].usesJoints = true;
	m_renderProgs[ BUILTIN_INTERACTION_AMBIENT_SKINNED ].usesJoints = true;
	m_renderProgs[ BUILTIN_ENVIRONMENT_SKINNED ].usesJoints = true;
	m_renderProgs[ BUILTIN_BUMPY_ENVIRONMENT_SKINNED ].usesJoints = true;
	m_renderProgs[ BUILTIN_DEPTH_SKINNED ].usesJoints = true;
	m_renderProgs[ BUILTIN_SHADOW_SKINNED ].usesJoints = true;
	m_renderProgs[ BUILTIN_SHADOW_DEBUG_SKINNED ].usesJoints = true;
	m_renderProgs[ BUILTIN_FOG_SKINNED ].usesJoints = true;

	// Create Vertex Descriptions
	CreateVertexDescriptions();

	// Create Descriptor Pools
	CreateDescriptorPools( m_descriptorPools );

	// Allocate a large uniform buffer for each frame.
	for ( int i = 0; i < NUM_FRAME_DATA; ++i ) {
		m_parmBuffers[ i ] = new idUniformBuffer();
		m_parmBuffers[ i ]->AllocBufferObject( NULL, MAX_DESC_SETS * MAX_DESC_SET_UNIFORMS * sizeof( idVec4 ) );
	}

	// Placeholder: mainly for optionalSkinning
	emptyUBO.AllocBufferObject( NULL, sizeof( idVec4 ) );
}

Fortunately DOOM 3 BFG only has a handful of shaders, so it's easy to just manually build them instead of relying on a complex system.  FindShader will attempt to load a shader if not found.  It's an API agnostic call, but let's look at the Vulkan implementation.  

/*
========================
idRenderProgManager::LoadShader
========================
*/
void idRenderProgManager::LoadShader( shader_t & shader ) {
	// Read in the spirv shader code and layout description.
	idStr spirvPath;
	idStr layoutPath;
	spirvPath.Format( "renderprogs\\spirv\\%s", shader.name.c_str() );
	layoutPath.Format( "renderprogs\\vkglsl\\%s", shader.name.c_str() );
	if ( shader.stage == SHADER_STAGE_FRAGMENT ) {
		spirvPath += ".fspv";
		layoutPath += ".frag.layout";
	} else {
		spirvPath += ".vspv";
		layoutPath += ".vert.layout";
	}

	void * spirvBuffer = NULL;
	int sprivLen = fileSystem->ReadFile( spirvPath.c_str(), &spirvBuffer );
	if ( sprivLen <= 0 ) {
		idLib::Error( "idRenderProgManager::LoadShader: Unable to load SPIRV shader file %s.", spirvPath.c_str() );
	}

	void * layoutBuffer = NULL;
	int layoutLen = fileSystem->ReadFile( layoutPath.c_str(), &layoutBuffer );
	if ( layoutLen <= 0 ) {
		idLib::Error( "idRenderProgManager::LoadShader: Unable to load layout file %s.", layoutPath.c_str() );
	}

	// Parse the layout description.
	idStr layout = ( const char * )layoutBuffer;

	idLexer src( layout.c_str(), layout.Length(), "layout" );
	idToken token;

	// Find the indices of all the renderpamrs this shader needs
	if ( src.ExpectTokenString( "uniforms" ) ) {
		src.ExpectTokenString( "[" );

		while ( !src.CheckTokenString( "]" ) ) {
			src.ReadToken( &token );

			int index = -1;
			for ( int i = 0; i < RENDERPARM_TOTAL && index == -1; ++i ) {
				if ( token == GLSLParmNames[ i ] ) {
					index = i;
				}
			}

			if ( index == -1 ) {
				idLib::Error( "Invalid uniform %s", token.c_str() );
			}

			shader.parmIndices.Append( static_cast< renderParm_t >( index ) );
		}
	}

	// Get all the layout binding types
	if ( src.ExpectTokenString( "bindings" ) ) {
		src.ExpectTokenString( "[" );

		while ( !src.CheckTokenString( "]" ) ) {
			src.ReadToken( &token );

			int index = -1;
			for ( int i = 0; i < BINDING_TYPE_MAX; ++i ) {
				if ( token == renderProgBindingStrings[ i ] ) {
					index = i;
				}
			}

			if ( index == -1 ) {
				idLib::Error( "Invalid binding %s", token.c_str() );
			}

			shader.bindings.Append( static_cast< rpBinding_t >( index ) );
		}
	}

	// Shove the spirv buffer into the create struct.
	VkShaderModuleCreateInfo shaderModuleCreateInfo = {};
	shaderModuleCreateInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
	shaderModuleCreateInfo.codeSize = sprivLen;
	shaderModuleCreateInfo.pCode = (uint32 *)spirvBuffer;

	// Create the shader module.
	ID_VK_CHECK( vkCreateShaderModule( vkcontext.device, &shaderModuleCreateInfo, NULL, &shader.module ) );

	// We can get rid of the buffers.
	Mem_Free( layoutBuffer );
	Mem_Free( spirvBuffer );
}

Cool, not really anything complicated going on here.  I'm not going to stop and talk about SPIRV really.  Some of you will know what I'm talking about, and some may not.  SPIRV is Vulkan's shader language.  It's actually quite nice to work with compared to other APIs.  Instead of uploading a text buffer at run time you instead precompile the shader module outside the application and upload that binary.

Let's bounce back to the next Vulkan routine from Init; CreateDescriptorSetLayout.  Let's think about this... Descriptor... Set ... Layout.  The name is pretty indicative of its purpose.  It's a layout of descriptor sets.  So it's a collection of descriptors, but of what?  Well if you remember from Part 2, I showed you the graphics pipeline from beginning to end.  Some of those stages are programmable ( you upload shaders for them ) and some are fixed function ( you just toggle some bits for them ).  Now shaders aren't very interesting without some input to work on.  Yes the vertex shader receives, well vertices.  And the fragment shader receivers *drumroll* fragments.  But what if we want to use textures, uniform buffers, or other data?  Well we have to describe a location where that data can be bound for the shader to use; hence descriptor set layout. Think of it as a way of interfacing your resources to your shader once on the GPU.  

Let's look at CreateDescriptorSetLayout now.

/*
========================
CreateDescriptorSetLayout
========================
*/
static void CreateDescriptorSetLayout( 
		const shader_t & vertexShader, 
		const shader_t & fragmentShader, 
		renderProg_t & renderProg ) {
	
	// Descriptor Set Layout
	{
		idList< VkDescriptorSetLayoutBinding > layoutBindings;
		VkDescriptorSetLayoutBinding binding = {};
		binding.descriptorCount = 1;
		
		uint32 bindingId = 0;

		binding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;
		for ( int i = 0; i < vertexShader.bindings.Num(); ++i ) {
			binding.binding = bindingId++;
			binding.descriptorType = GetDescriptorType( vertexShader.bindings[ i ] );
			renderProg.bindings.Append( vertexShader.bindings[ i ] );

			layoutBindings.Append( binding );
		}

		binding.stageFlags = VK_SHADER_STAGE_FRAGMENT_BIT;
		for ( int i = 0; i < fragmentShader.bindings.Num(); ++i ) {
			binding.binding = bindingId++;
			binding.descriptorType = GetDescriptorType( fragmentShader.bindings[ i ] );
			renderProg.bindings.Append( fragmentShader.bindings[ i ] );

			layoutBindings.Append( binding );
		}

		VkDescriptorSetLayoutCreateInfo createInfo = {};
		createInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
		createInfo.bindingCount = layoutBindings.Num();
		createInfo.pBindings = layoutBindings.Ptr();

		vkCreateDescriptorSetLayout( vkcontext.device, &createInfo, NULL, &renderProg.descriptorSetLayout );
	}

	// Pipeline Layout
	{
		VkPipelineLayoutCreateInfo createInfo = {};
		createInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
		createInfo.setLayoutCount = 1;
		createInfo.pSetLayouts = &renderProg.descriptorSetLayout;

		vkCreatePipelineLayout( vkcontext.device, &createInfo, NULL, &renderProg.pipelineLayout );
	}
}

So basically we just take the bindings we loaded for each shader associated with this renderprog and build a list of types paired with indices.  "But why not restart the index at zero for the fragment shader?" you might ask.  Well remember that shaders are just part of a pipeline.  They take inputs, do some processing, and then forward this onto the next stage.  Their output isn't a result unto itself.  To this end we are describing every part of the pipeline where a resource can be bound, over the shaders we supplied.  Let's see what we can bind at these locations ( Note VkNeo only uses 2x types ).  Note all prefixed with VK_DESCRIPTOR_TYPE_.

1.  SAMPLER - Can be used to sample images.
2.  COMBINED_IMAGE_SAMPLER - Combines a sampler and associated image. (VkNeo)
3.  SAMPLED_IMAGE - Used in conjunction with a SAMPLER.
4.  STORAGE_IMAGE - Read/write image memory.
5.  UNIFORM_TEXEL_BUFFER - Read only array of homogeneous data.
6.  STORAGE_TEXEL_BUFFER - Read/write array of homogeneous data.
7.  UNIFORM_BUFFER - Read only structured buffer.  (VkNeo)
8.  STORAGE_BUFFER - Read/write structured buffer.
9.  UNIFORM_BUFFER_DYNAMIC - Same as UNIFORM_BUFFER, but better for storing multiple objects and simply offsetting into them.
10.  STORAGE_BUFFER_DYNAMIC - Same as STORAGE_BUFFER, but better for storing multiple objects and simply offsetting into them.
11.  INPUT_ATTACHMENT - Used for unfiltered image views.  

Let's take a break and look at the whole pipeline again.  

View fullsize

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1501042354206-81U2BEU195QDNAZE1OGV/image-asset.jpeg?format=750w)

So far we've covered the Vertex Shader and Fragment Shader stages.  The other programmable stages, tessellation, geometry, and compute are not used in VkNeo.  Let's return to idRenderProgManager::Init and look at setting up the Vertex Input stage of the pipeline as accomplished through CreateVertexDescriptions.

/*
=============
CreateVertexDescriptions
=============
*/
static void CreateVertexDescriptions() {
	VkPipelineVertexInputStateCreateInfo createInfo = {};
	createInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;
	createInfo.pNext = NULL;
	createInfo.flags = 0;

	VkVertexInputBindingDescription binding = {};
	VkVertexInputAttributeDescription attribute = {};

	{
		vertexLayout_t & layout = vertexLayouts[ LAYOUT_DRAW_VERT ];
		layout.inputState = createInfo;

		uint32 locationNo = 0;
		uint32 offset = 0;

		binding.stride = sizeof( idDrawVert );
		binding.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
		layout.bindingDesc.Append( binding );

		// Position
		attribute.format = VK_FORMAT_R32G32B32_SFLOAT;
		attribute.location = locationNo++;
		attribute.offset = offset;
		layout.attributeDesc.Append( attribute );
		offset += sizeof( idDrawVert::xyz );

		// TexCoord
		attribute.format = VK_FORMAT_R16G16_SFLOAT;
		attribute.location = locationNo++;
		attribute.offset = offset;
		layout.attributeDesc.Append( attribute );
		offset += sizeof( idDrawVert::st );

		// Normal
		attribute.format = VK_FORMAT_R8G8B8A8_UNORM;
		attribute.location = locationNo++;
		attribute.offset = offset;
		layout.attributeDesc.Append( attribute );
		offset += sizeof( idDrawVert::normal );

		// Tangent
		attribute.format = VK_FORMAT_R8G8B8A8_UNORM;
		attribute.location = locationNo++;
		attribute.offset = offset;
		layout.attributeDesc.Append( attribute );
		offset += sizeof( idDrawVert::tangent );

		// Color1
		attribute.format = VK_FORMAT_R8G8B8A8_UNORM;
		attribute.location = locationNo++;
		attribute.offset = offset;
		layout.attributeDesc.Append( attribute );
		offset += sizeof( idDrawVert::color );

		// Color2
		attribute.format = VK_FORMAT_R8G8B8A8_UNORM;
		attribute.location = locationNo++;
		attribute.offset = offset;
		layout.attributeDesc.Append( attribute );
	}
}

"Oh Vulkan's a lot of code."  Yes, but it's mostly this.  You'll fill out those structs and you'll like it!  So DOOM 3 BFG has only three vertex layouts [ DRAW_VERT, DRAW_SHADOW_VERT, DRAW_SHADOW_VERT_SKINNED ].  DRAW_VERT is the most complex and often used, so we'll only go over that one ( others excluded above for brevity ).  Essentially this just describes the vertex data we'll be supplying to the pipeline.  DRAW_VERT is 32 bytes and is laid out as such.

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1501043115105-Z69YBCEGCEUL6XZ2KS5K/image-asset.jpeg?format=750w)

Remember in [Part 4.5](https://www.fasterthan.life/blog/2017/7/16/i-am-graphics-and-so-can-you-idtech) we talked about the job of the front end vs backend?  Well the frontend loads up the vertexCache and the backend draws what's in the vertexCache; simple as that.  Before we get into next steps, let's look at at least one shader example.  ( gui.vert and gui.frag which are used to draw UI elements )

#version 450
#pragma shader_stage( vertex )

#extension GL_ARB_separate_shader_objects : enable

layout( binding = 0 ) uniform UBO {
    vec4 rpMVPmatrixX;
    vec4 rpMVPmatrixY;
    vec4 rpMVPmatrixZ;
    vec4 rpMVPmatrixW;
};

layout( location = 0 ) in vec3 in_Position;
layout( location = 1 ) in vec2 in_TexCoord;
layout( location = 2 ) in vec4 in_Normal;
layout( location = 3 ) in vec4 in_Tangent;
layout( location = 4 ) in vec4 in_Color;
layout( location = 5 ) in vec4 in_Color2;

layout( location = 0 ) out vec2 out_TexCoord0;
layout( location = 1 ) out vec4 out_TexCoord1;
layout( location = 2 ) out vec4 out_Color;

void main() {
    vec4 position = vec4( in_Position, 1.0 );
    gl_Position.x = dot( position, rpMVPmatrixX );
    gl_Position.y = dot( position, rpMVPmatrixY );
    gl_Position.z = dot( position, rpMVPmatrixZ );
    gl_Position.w = dot( position, rpMVPmatrixW );
    
    out_TexCoord0.xy = in_TexCoord.xy;
    out_TexCoord1 = ( in_Color2 * 2 ) - 1;
    out_Color = in_Color;
}

The first thing we run into is layout( binding = 0 ) uniform UBO.  This is taken care of in CreateDescriptorSetLayout, and the UBO is bound upon committing for a draw call.  layout( location = n ) in describes the vertex input parameters setup in CreateVertexDescriptions.  The layout( location = n ) out parameters are what get fed to the next programmable stage of the pipeline, the fragment shader.  

#version 450
#pragma shader_stage( fragment )

#extension GL_ARB_separate_shader_objects : enable

layout( binding = 1 ) uniform sampler2D samp0;

layout( location = 0 ) in vec2 in_TexCoord0;
layout( location = 1 ) in vec4 in_TexCoord1;
layout( location = 2 ) in vec4 in_Color;

layout( location = 0 ) out vec4 out_Color;

void main() {
    vec4 color = ( texture( samp0, in_TexCoord0.xy ) * in_Color ) + in_TexCoord1;
    out_Color.xyz = color.xyz * color.w;
    out_Color.w = color.w ;
}

See how the outputs from the vertex shader feed into the fragment one?  This again reinforces the  reality that all these stages are part of a pipeline for processing data.  The final out_Color variable is what gets applied to the render target ( in VkNeo this will be our swap chain image we acquired earlier ).

The last Vulkan thing we'll look at for idRenderProgManager::Init is CreateDescriptorPools.  The reason for pooling descriptor sets will become clear once we look at CommitCurrent, so just stick with me.

/*
========================
CreateDescriptorPools
========================
*/
static void CreateDescriptorPools( VkDescriptorPool (&pools)[ NUM_FRAME_DATA ] ) {
	const int numPools = 2;
	VkDescriptorPoolSize poolSizes[ numPools ];
	poolSizes[ 0 ].type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
	poolSizes[ 0 ].descriptorCount = MAX_DESC_UNIFORM_BUFFERS;   // 8192
	poolSizes[ 1 ].type = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
	poolSizes[ 1 ].descriptorCount = MAX_DESC_IMAGE_SAMPLERS;    // 12384

	VkDescriptorPoolCreateInfo poolCreateInfo = {};
	poolCreateInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
	poolCreateInfo.pNext = NULL;
	poolCreateInfo.maxSets = MAX_DESC_SETS; // 16384
	poolCreateInfo.poolSizeCount = numPools;
	poolCreateInfo.pPoolSizes = poolSizes;

	for ( int i = 0; i < NUM_FRAME_DATA; ++i ) {
		ID_VK_CHECK( vkCreateDescriptorPool( vkcontext.device, &poolCreateInfo, NULL, &pools[ i ] ) );
	}
}

This is all fairly straightforward.  Again once we get to CommitCurrent, we'll talk about how descriptor sets are used and why we need to pool them.  ( similar to why you'd pool anything really )

# Final Boss

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1501046193810-MP8R3Q0G08YTMNTN11QJ/image-asset.jpeg?format=750w)

It finally comes to it.  This is probably what I would consider the core of Vulkandom.  This is where graphics pipelines get made and used.  I hope by now in the series it has really sunk in what a pipeline is and why I keep referring to it in terms of graphics.  If not this next part will be brutal.  But if you've been paying attention, then you can ride the waves and come out as the sea's master.  Let's do this.

/*
========================
idRenderProgManager::CommitCurrent
========================
*/
void idRenderProgManager::CommitCurrent( uint64 stateBits ) {
	renderProg_t & prog = m_renderProgs[ m_current ];

	// m_current is set through BindProgram in higher level draw code.
	// We then take that renderProg_t and get a Vulkan pipeline 
	// from it which satisfies the glstatebits set by GL_State
	// ( input to this function ) as well as some other bits.
	VkPipeline pipeline = prog.GetPipeline( 
		stateBits,
		m_shaders[ prog.vertexShaderIndex ].module,
		prog.fragmentShaderIndex != -1 ? m_shaders[ prog.fragmentShaderIndex ].module : VK_NULL_HANDLE );

	// Now we take the descriptor set layout we created for the renderProg_t 
	// earlier and setup an allocation for a descriptor set matching it.
	VkDescriptorSetAllocateInfo setAllocInfo = {};
	setAllocInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO;
	setAllocInfo.pNext = NULL;
	// Use the pool we created earlier ( the one dedicated to this frame )
	setAllocInfo.descriptorPool = m_descriptorPools[ m_currentData ];
	// We only need to allocate one
	setAllocInfo.descriptorSetCount = 1;
	setAllocInfo.pSetLayouts = &prog.descriptorSetLayout;

	// Allocate the descriptor set from the pool.
	ID_VK_CHECK( vkAllocateDescriptorSets( vkcontext.device, &setAllocInfo, &m_descriptorSets[ m_currentData ][ m_currentDescSet ] ) );

	// Grab it and increment how many descriptor sets we've used
	VkDescriptorSet descSet = m_descriptorSets[ m_currentData ][ m_currentDescSet ];
	m_currentDescSet++;

	// Used for counting our bound resources
	int writeIndex = 0;
	int bufferIndex = 0;
	int imageIndex = 0;
	int bindingIndex = 0;
	
	// We write resource descriptions to the descriptor set we just created.
	// Those are tracked in these arrays.
	VkWriteDescriptorSet writes[ MAX_DESC_SET_WRITES ];
	VkDescriptorBufferInfo bufferInfos[ MAX_DESC_SET_WRITES ];
	VkDescriptorImageInfo imageInfos[ MAX_DESC_SET_WRITES ];

	// VkNeo at max supports 3x UBOs per pipeline.
	// 1.) vertex shader renderparm UBO
	// 2.) optional skinning data UBO
	// 3.) fragment shader renderparm UBO
	int uboIndex = 0;
	idUniformBuffer * ubos[ 3 ] = { NULL, NULL, NULL };

	// Get the vertex UBO
	idUniformBuffer vertParms;
	if ( prog.vertexShaderIndex > -1 && m_shaders[ prog.vertexShaderIndex ].parmIndices.Num() > 0 ) {
		// Allocate a chunk of this frame's renderparm UBO for this draw call.
		// Fill it with current values set from the higher level draw pass.
		AllocParmBlockBuffer( m_shaders[ prog.vertexShaderIndex ].parmIndices, vertParms );

		ubos[ uboIndex++ ] = &vertParms;
	}

	// If the joint cache handle is set then grab the joint UBO.
	idUniformBuffer jointBuffer;
	if ( prog.usesJoints && vkcontext.jointCacheHandle > 0 ) {
		if ( !vertexCache.GetJointBuffer( vkcontext.jointCacheHandle, &jointBuffer ) ) {
			idLib::Error( "idRenderProgManager::CommitCurrent: jointBuffer == NULL" );
			return;
		}
		assert( ( jointBuffer.GetOffset() & ( vkcontext.gpu->props.limits.minUniformBufferOffsetAlignment - 1 ) ) == 0 );

		ubos[ uboIndex++ ] = &jointBuffer;
	} else if ( prog.optionalSkinning ) {
		// If the shader supports optional skinning simply bind an empy UBO
		// This is an artifact of not being able to change shaders in the 
		// shipped resource files.
		ubos[ uboIndex++ ] = &emptyUBO;
	}

	// Get the fragment UBO
	idUniformBuffer fragParms;
	if ( prog.fragmentShaderIndex > -1 && m_shaders[ prog.fragmentShaderIndex ].parmIndices.Num() > 0 ) {
		// Allocate a chunk of this frame's renderparm UBO for this draw call.
		// Fill it with current values set from the higher level draw pass.
		AllocParmBlockBuffer( m_shaders[ prog.fragmentShaderIndex ].parmIndices, fragParms );

		ubos[ uboIndex++ ] = &fragParms;
	}

	// No go over all the bindings we setup for the renderProg_t
	// We need to bind the appropriate resource at the appropriate
	// index we specified when creating the layout.
	for ( int i = 0; i < prog.bindings.Num(); ++i ) {
		rpBinding_t binding = prog.bindings[ i ];

		switch ( binding ) {
			case BINDING_TYPE_UNIFORM_BUFFER: {
				// The binding is a UBO type.

				idUniformBuffer * ubo = ubos[ bufferIndex ];

				// Get the necessary info from the idUniformBuffer
				// we covered in Part 4
				VkDescriptorBufferInfo & bufferInfo = bufferInfos[ bufferIndex++ ];
				memset( &bufferInfo, 0, sizeof( VkDescriptorBufferInfo ) );
				bufferInfo.buffer = ubo->GetAPIObject();
				bufferInfo.offset = ubo->GetOffset();
				bufferInfo.range = ubo->GetSize();

				// Now we fill out the write to the descriptor set.
				VkWriteDescriptorSet & write = writes[ writeIndex++ ];
				memset( &write, 0, sizeof( VkWriteDescriptorSet ) );
				write.sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
				// The descriptor set we just allocated
				write.dstSet = descSet;
				// The binding number ( as shown in the example shader code )
				write.dstBinding = bindingIndex++;
				write.descriptorCount = 1;
				write.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
				write.pBufferInfo = &bufferInfo;
				
				break;
			}
			case BINDING_TYPE_SAMPLER: {
				// The binding is a combined image/sampler type.

				idImage * image = vkcontext.imageParms[ imageIndex ];

				// Get the necessary info from the idImage
				// we covered in Part 4.
				VkDescriptorImageInfo & imageInfo = imageInfos[ imageIndex++ ];
				memset( &imageInfo, 0, sizeof( VkDescriptorImageInfo ) );
				imageInfo.imageLayout = image->GetLayout();
				imageInfo.imageView = image->GetView();
				imageInfo.sampler = image->GetSampler();
				
				assert( image->GetView() != VK_NULL_HANDLE );

				// Now we fill out the write to the descriptor set.
				VkWriteDescriptorSet & write = writes[ writeIndex++ ];
				memset( &write, 0, sizeof( VkWriteDescriptorSet ) );
				write.sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
				write.dstSet = descSet;
				write.dstBinding = bindingIndex++;
				write.descriptorCount = 1;
				write.descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
				write.pImageInfo = &imageInfo;
				
				break;
			}
		}
	}

	// Submit the writes to the device.
	vkUpdateDescriptorSets( vkcontext.device, writeIndex, writes, 0, NULL );

	// Bind the decriptor set to the graphics bind point.
	vkCmdBindDescriptorSets( 
		vkcontext.commandBuffer[ vkcontext.currentFrameData ], 
		VK_PIPELINE_BIND_POINT_GRAPHICS, 
		prog.pipelineLayout, 0, 1, &descSet, 
		0, NULL );

	// Bind the pipeline to the graphics bind point.
	vkCmdBindPipeline( 
		vkcontext.commandBuffer[ vkcontext.currentFrameData ], 
		VK_PIPELINE_BIND_POINT_GRAPHICS, 
		pipeline );

	// If you recall in Part 4.5, we called draw shortly after this.
	// Now the draw has all the information it needs to properly 
	// render its little chunk of the world.
}

Holy frijoles that was a lot of code.  But wait there's more!  There's a second stage to this boss battle.  Be not dismayed.  Let's recap what's going on here.  ( And please note the last comment in the code block )

1.  Get a pipeline satisfying the GL_State bits set in the higher level renderer.  
2.  Allocate a descriptor set that matches the layout for the renderProg_t.  We pool them because we need a descSet for each draw command.
3.  Grab the shader UBOs that match the layout.
4.  Loop over all the layout's binding points and properly describe and set the resource(s).
5.  Submit the descriptor set writes to the device.
6.  Bind descriptor set
7.  Bind pipeline.

Now for GetPipeline.  I'll skip that actual function because it mainly checks if the state bits match and if the optional separate ( FRONT/BACK )stencil operations match.  If it fails to find a pipeline it will instead create one.  Time for stage two.

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1501047996979-ONX9UQNBBWS09P8ZO6DJ/image-asset.jpeg?format=750w)

/*
========================
CreateGraphicsPipeline
========================
*/
static VkPipeline CreateGraphicsPipeline(
		vertexLayoutType_t vertexLayoutType,
		VkShaderModule vertexShader,
		VkShaderModule fragmentShader,
		VkPipelineLayout pipelineLayout,
		uint64 stateBits ) {

	// Pipeline
	// This actually contains several elements we setup during Init
	// for each fo the vertex layouts.  See Vertex Input
	// below for how they are used.
	vertexLayout_t & vertexLayout = vertexLayouts[ vertexLayoutType ];

	// Vertex Input
	VkPipelineVertexInputStateCreateInfo vertexInputState = vertexLayout.inputState;
	vertexInputState.vertexBindingDescriptionCount = vertexLayout.bindingDesc.Num();
	vertexInputState.pVertexBindingDescriptions = vertexLayout.bindingDesc.Ptr();
	vertexInputState.vertexAttributeDescriptionCount = vertexLayout.attributeDesc.Num();
	vertexInputState.pVertexAttributeDescriptions = vertexLayout.attributeDesc.Ptr();

	// Input Assembly
	// VkNeo just needs triange lists although there are several other
	// topologies one could use.
	VkPipelineInputAssemblyStateCreateInfo assemblyInputState = {};
	assemblyInputState.sType = VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO;
	assemblyInputState.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;

	// Rasterization
	// This configures the fixed function hardware rasterizer stage of the pipeline.
	VkPipelineRasterizationStateCreateInfo rasterizationState = {};
	rasterizationState.sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO;
	// Don't need this
	rasterizationState.rasterizerDiscardEnable = VK_FALSE;
	// If GL_State set a polygon offset, then enable depth bias
	rasterizationState.depthBiasEnable = ( stateBits & GLS_POLYGON_OFFSET ) != 0;
	// Don't need this
	rasterizationState.depthClampEnable = VK_FALSE;
	// For the most part this will always be CCW.
	rasterizationState.frontFace = ( stateBits & GLS_CLOCKWISE ) ? VK_FRONT_FACE_CLOCKWISE : VK_FRONT_FACE_COUNTER_CLOCKWISE;
	// Mainly used in debug features which I'm not dealing with yet.
	rasterizationState.lineWidth = 1.0f;
	// Again am not at the point of dealing with debug features so 
	// all of these really default to FILL.
	rasterizationState.polygonMode = ( stateBits & GLS_POLYMODE_LINE ) ? VK_POLYGON_MODE_LINE : VK_POLYGON_MODE_FILL;

	// What cull mode to use?
	switch ( stateBits & GLS_CULL_BITS ) {
	case GLS_CULL_TWOSIDED:
		// Both sides are culled hurray!
		rasterizationState.cullMode = VK_CULL_MODE_NONE;
		break;
	case GLS_CULL_BACKSIDED:
		// Normally for backsided we would cull the back facing triangles
		// But for mirror views we cull the front facing triangles
		if ( stateBits & GLS_MIRROR_VIEW ) {
			rasterizationState.cullMode = VK_CULL_MODE_FRONT_BIT;
		} else {
			rasterizationState.cullMode = VK_CULL_MODE_BACK_BIT;
		}
		break;
	case GLS_CULL_FRONTSIDED:
	default:
		// Normally for frontsided we would cull the front facing triangles
		// But for mirror views we cull the back facing triangles
		if ( stateBits & GLS_MIRROR_VIEW ) {
			rasterizationState.cullMode = VK_CULL_MODE_BACK_BIT;
		} else {
			rasterizationState.cullMode = VK_CULL_MODE_FRONT_BIT;
		}
		break;
	}

	// Color Blend Attachment
	// This determines how the final fragment output from the fragment shader
	// stage gets blended into the frame buffer.
	VkPipelineColorBlendAttachmentState attachmentState = {};
	{
		// The source factor
		VkBlendFactor srcFactor = VK_BLEND_FACTOR_ONE;
		switch ( stateBits & GLS_SRCBLEND_BITS ) {
			case GLS_SRCBLEND_ZERO:		srcFactor = VK_BLEND_FACTOR_ZERO; break;
			case GLS_SRCBLEND_ONE:			srcFactor = VK_BLEND_FACTOR_ONE; break;
			case GLS_SRCBLEND_DST_COLOR:		srcFactor = VK_BLEND_FACTOR_DST_COLOR; break;
			case GLS_SRCBLEND_ONE_MINUS_DST_COLOR:	srcFactor = VK_BLEND_FACTOR_ONE_MINUS_DST_COLOR; break;
			case GLS_SRCBLEND_SRC_ALPHA:		srcFactor = VK_BLEND_FACTOR_SRC_ALPHA; break;
			case GLS_SRCBLEND_ONE_MINUS_SRC_ALPHA:	srcFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA; break;
			case GLS_SRCBLEND_DST_ALPHA:		srcFactor = VK_BLEND_FACTOR_DST_ALPHA; break;
			case GLS_SRCBLEND_ONE_MINUS_DST_ALPHA:	srcFactor = VK_BLEND_FACTOR_ONE_MINUS_DST_ALPHA; break;
		}

		// The destination factor
		VkBlendFactor dstFactor = VK_BLEND_FACTOR_ZERO;
		switch ( stateBits & GLS_DSTBLEND_BITS ) {
			case GLS_DSTBLEND_ZERO:		dstFactor = VK_BLEND_FACTOR_ZERO; break;
			case GLS_DSTBLEND_ONE:			dstFactor = VK_BLEND_FACTOR_ONE; break;
			case GLS_DSTBLEND_SRC_COLOR:		dstFactor = VK_BLEND_FACTOR_SRC_COLOR; break;
			case GLS_DSTBLEND_ONE_MINUS_SRC_COLOR:	dstFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_COLOR; break;
			case GLS_DSTBLEND_SRC_ALPHA:		dstFactor = VK_BLEND_FACTOR_SRC_ALPHA; break;
			case GLS_DSTBLEND_ONE_MINUS_SRC_ALPHA:	dstFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA; break;
			case GLS_DSTBLEND_DST_ALPHA:		dstFactor = VK_BLEND_FACTOR_DST_ALPHA; break;
			case GLS_DSTBLEND_ONE_MINUS_DST_ALPHA:	dstFactor = VK_BLEND_FACTOR_ONE_MINUS_DST_ALPHA; break;
		}

		// The blend operation to perform
		VkBlendOp blendOp = VK_BLEND_OP_ADD;
		switch ( stateBits & GLS_BLENDOP_BITS ) {
			case GLS_BLENDOP_MIN: blendOp = VK_BLEND_OP_MIN; break;
			case GLS_BLENDOP_MAX: blendOp = VK_BLEND_OP_MAX; break;
			case GLS_BLENDOP_ADD: blendOp = VK_BLEND_OP_ADD; break;
			case GLS_BLENDOP_SUB: blendOp = VK_BLEND_OP_SUBTRACT; break;
		}

		// Enable blend?
		attachmentState.blendEnable = ( srcFactor != VK_BLEND_FACTOR_ONE || dstFactor != VK_BLEND_FACTOR_ZERO );
		attachmentState.colorBlendOp = blendOp;
		attachmentState.srcColorBlendFactor = srcFactor;
		attachmentState.dstColorBlendFactor = dstFactor;
		// Use the same parameters for alpha
		attachmentState.alphaBlendOp = blendOp;
		attachmentState.srcAlphaBlendFactor = srcFactor;
		attachmentState.dstAlphaBlendFactor = dstFactor;

		// Color Mask
		attachmentState.colorWriteMask = 0;
		attachmentState.colorWriteMask |= ( stateBits & GLS_REDMASK ) ?	0 : VK_COLOR_COMPONENT_R_BIT;
		attachmentState.colorWriteMask |= ( stateBits & GLS_GREENMASK ) ? 0 : VK_COLOR_COMPONENT_G_BIT;
		attachmentState.colorWriteMask |= ( stateBits & GLS_BLUEMASK ) ? 0 : VK_COLOR_COMPONENT_B_BIT;
		attachmentState.colorWriteMask |= ( stateBits & GLS_ALPHAMASK ) ? 0 : VK_COLOR_COMPONENT_A_BIT;
	}

	// Color Blend
	// We take the attachment we just filled out and add it here.
	// VkNeo only has the one swap image per frame.
	VkPipelineColorBlendStateCreateInfo colorBlendState = {};
	colorBlendState.sType = VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO;
	colorBlendState.attachmentCount = 1;
	colorBlendState.pAttachments = &attachmentState;

	// Depth / Stencil
	// I went over Depth pretty heavily in 4.5.
	// I'll go over stencil in a separate post about shadow volumes.
	VkPipelineDepthStencilStateCreateInfo depthStencilState = {};
	{
		// How do we compare the depth values?
		VkCompareOp depthCompareOp = VK_COMPARE_OP_ALWAYS;
		switch ( stateBits & GLS_DEPTHFUNC_BITS ) {
			case GLS_DEPTHFUNC_EQUAL:	depthCompareOp = VK_COMPARE_OP_EQUAL; break;
			case GLS_DEPTHFUNC_ALWAYS:	depthCompareOp = VK_COMPARE_OP_ALWAYS; break;
			case GLS_DEPTHFUNC_LESS:	depthCompareOp = VK_COMPARE_OP_LESS_OR_EQUAL; break;
			case GLS_DEPTHFUNC_GREATER:	depthCompareOp = VK_COMPARE_OP_GREATER_OR_EQUAL; break;
		}

		// How do we compare the stencil values?
		VkCompareOp stencilCompareOp = VK_COMPARE_OP_ALWAYS;
		switch ( stateBits & GLS_STENCIL_FUNC_BITS ) {
			case GLS_STENCIL_FUNC_NEVER:	stencilCompareOp = VK_COMPARE_OP_NEVER; break;
			case GLS_STENCIL_FUNC_LESS:	stencilCompareOp = VK_COMPARE_OP_LESS; break;
			case GLS_STENCIL_FUNC_EQUAL:	stencilCompareOp = VK_COMPARE_OP_EQUAL; break;
			case GLS_STENCIL_FUNC_LEQUAL:	stencilCompareOp = VK_COMPARE_OP_LESS_OR_EQUAL; break;
			case GLS_STENCIL_FUNC_GREATER:	stencilCompareOp = VK_COMPARE_OP_GREATER; break;
			case GLS_STENCIL_FUNC_NOTEQUAL: stencilCompareOp = VK_COMPARE_OP_NOT_EQUAL; break;
			case GLS_STENCIL_FUNC_GEQUAL:	stencilCompareOp = VK_COMPARE_OP_GREATER_OR_EQUAL; break;
			case GLS_STENCIL_FUNC_ALWAYS:	stencilCompareOp = VK_COMPARE_OP_ALWAYS; break;
		}

		depthStencilState.sType = VK_STRUCTURE_TYPE_PIPELINE_DEPTH_STENCIL_STATE_CREATE_INFO;
		// We always use the depth test ( some debug code probably disable it, but haven't gotten that far )
		depthStencilState.depthTestEnable = VK_TRUE;
		// If the mask isn't set then enable write.
		depthStencilState.depthWriteEnable = ( stateBits & GLS_DEPTHMASK ) == 0;
		depthStencilState.depthCompareOp = depthCompareOp;
		// If the mask is set then we need to enable 
		depthStencilState.depthBoundsTestEnable = ( stateBits & GLS_DEPTH_TEST_MASK ) != 0;
		// These are just default values.  I set the bounds dynamically if enabled. ( see below )
		depthStencilState.minDepthBounds = 0.0f;
		depthStencilState.maxDepthBounds = 1.0f;
		// Is the stencil test enabled?
		depthStencilState.stencilTestEnable = ( stateBits & ( GLS_STENCIL_FUNC_BITS | GLS_STENCIL_OP_BITS | GLS_SEPARATE_STENCIL ) ) != 0;

		// Get the stencil function ref value
		uint32 ref = uint32( ( stateBits & GLS_STENCIL_FUNC_REF_BITS ) >> GLS_STENCIL_FUNC_REF_SHIFT );
		// Get the stencil function mask value
		uint32 mask = uint32( ( stateBits & GLS_STENCIL_FUNC_MASK_BITS ) >> GLS_STENCIL_FUNC_MASK_SHIFT );

		// Separate stencil operations are stored on the context and not in glstatebits.
		if ( stateBits & GLS_SEPARATE_STENCIL ) {
			// Set the front operation
			depthStencilState.front = GetStencilOpState( vkcontext.stencilOperations[ STENCIL_FACE_FRONT ] );
			depthStencilState.front.writeMask = 0xFFFFFFFF;
			depthStencilState.front.compareOp = stencilCompareOp;
			depthStencilState.front.compareMask = mask;
			depthStencilState.front.reference = ref;

			// Set the back operation
			depthStencilState.back = GetStencilOpState( vkcontext.stencilOperations[ STENCIL_FACE_BACK ] );
			depthStencilState.back.writeMask = 0xFFFFFFFF;
			depthStencilState.back.compareOp = stencilCompareOp;
			depthStencilState.back.compareMask = mask;
			depthStencilState.back.reference = ref;
		} else {
			// The front and back are the same
			depthStencilState.front = GetStencilOpState( stateBits );
			depthStencilState.front.writeMask = 0xFFFFFFFF;
			depthStencilState.front.compareOp = stencilCompareOp;
			depthStencilState.front.compareMask = mask;
			depthStencilState.front.reference = ref;
			depthStencilState.back = depthStencilState.front;
		}
	}

	// Multisample
	// For right now I'm not using multisample.
	// This will change soon.
	VkPipelineMultisampleStateCreateInfo multisampleState = {};
	multisampleState.sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO;
	multisampleState.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;

	// Shader Stages
	// Now we take the shaders we loaded earlier and plug them in here.
	idList< VkPipelineShaderStageCreateInfo > stages;
	VkPipelineShaderStageCreateInfo stage = {};
	stage.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
	stage.pName = "main";

	{
		stage.module = vertexShader;
		stage.stage = VK_SHADER_STAGE_VERTEX_BIT;
		stages.Append( stage );
	}

	// We don't have fragment shaders for shadow renderProgs
	if ( fragmentShader != VK_NULL_HANDLE ) {
		stage.module = fragmentShader;
		stage.stage = VK_SHADER_STAGE_FRAGMENT_BIT;
		stages.Append( stage );
	}

	// Dynamic
	// Dynamic values can be updated without having to create a whole
	// new pipeline.  Viewport and scissor are always dynamic.
	idList< VkDynamicState > dynamic;
	dynamic.Append( VK_DYNAMIC_STATE_SCISSOR );
	dynamic.Append( VK_DYNAMIC_STATE_VIEWPORT );

	// Some shaders change the depth bias regularly.
	if ( stateBits & GLS_POLYGON_OFFSET ) {
		dynamic.Append( VK_DYNAMIC_STATE_DEPTH_BIAS );
	}

	// Some shaders change the depth bounds regularly.
	if ( stateBits & GLS_DEPTH_TEST_MASK ) {
		dynamic.Append( VK_DYNAMIC_STATE_DEPTH_BOUNDS );
	}

	// Add the dynamic state here.
	VkPipelineDynamicStateCreateInfo dynamicState = {};
	dynamicState.sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO;
	dynamicState.dynamicStateCount = dynamic.Num();
	dynamicState.pDynamicStates = dynamic.Ptr();

	// Viewport / Scissor
	// We don't need to supply data other than 1x for each of these,
	// as they are set dynamically.
	VkPipelineViewportStateCreateInfo viewportState = {};
	viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
	viewportState.viewportCount = 1;
	viewportState.scissorCount = 1;

	// Pipeline Create
	// Now shove it all into the final struct!
	VkGraphicsPipelineCreateInfo createInfo = {};
	createInfo.sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO;
	createInfo.layout = pipelineLayout;
	createInfo.renderPass = vkcontext.renderPass;
	createInfo.pVertexInputState = &vertexInputState;
	createInfo.pInputAssemblyState = &assemblyInputState;
	createInfo.pRasterizationState = &rasterizationState;
	createInfo.pColorBlendState = &colorBlendState;
	createInfo.pDepthStencilState = &depthStencilState;
	createInfo.pMultisampleState = &multisampleState;
	createInfo.pDynamicState = &dynamicState;
	createInfo.pViewportState = &viewportState;
	createInfo.stageCount = stages.Num();
	createInfo.pStages = stages.Ptr();

	VkPipeline pipeline = VK_NULL_HANDLE;

	// Create and return the pipeline!!
	ID_VK_CHECK( vkCreateGraphicsPipelines( vkcontext.device, vkcontext.pipelineCache, 1, &createInfo, NULL, &pipeline ) );

	// Mic drop
	return pipeline;
}

Sit on this for a while.  Look at the renderdoc pipeline diagram.  Look at the code.  Look at the diagram again.  Look at the code.  As questions arise, look up answers ( ask me ).  Read up on what each does section or function does.  It will take time for everything to sink in.  But before your eyes lies bare the whole graphics pipeline in a nutshell.  You just need to understand how to plug all the bits in.  

A graphics programmer's work is never done, but you my friend have made it.  You've tackled every major component of Vulkan that VkNeo uses.  Essentially if you understand this series, then you know how to write a Vulkan renderer for an existing game.  There are small bits and pieces that we could ( and will ) cover in future posts.  But like the Millennium Falcon flying through the Death Star, we ripped through Vulkan on this whirl wind tour of the API.  So grab yourself some shawarma, you deserve it.

![](https://images.squarespace-cdn.com/content/v1/594f554b6b8f5be7aade065f/1501050927268-ZFC5FVGAAJFX3WT8KYGV/image-asset.jpeg?format=750w)

P.S. I will have a wrapup post to sow everything together.  I'll also be making future posts as I open source VkNeo.  For now though, bask in the glory that you crossed the finish line.  Now go out there and graphics or vulkan or something.

Cheers