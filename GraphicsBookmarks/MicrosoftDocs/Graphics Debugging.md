# Visual Studio Graphics Diagnostics
 Note

This article applies to Visual Studio 2015. If you're looking for the latest Visual Studio documentation, see [Visual Studio documentation](https://docs.microsoft.com/en-us/visualstudio/windows). We recommend upgrading to Visual Studio 2019. [Download it here](https://visualstudio.microsoft.com/downloads)

Visual Studio_Graphics Diagnostics_ is a set of tools for recording and then analyzing rendering and performance problems in Direct3D apps. Graphics Diagnostics can be used on apps that are running locally on your Windows PC, in a Windows device emulator, or on a remote PC or device.

The Graphics Diagnostics workflow begins by capturing a record of how your app uses Direct3D—live, as it runs—so that its behavior can be analyzed immediately, shared, or saved for later. Capture sessions can be initiated and controlled manually from Visual Studio or with the command-line capture tool **dxcap.exe**. Capture sessions can also be initiated and controlled programmatically by using the Graphics Diagnostics capture APIs.

After a capture session has been recorded its contents can be played back by Visual Studio _Graphics Analyzer_ at any time, recreating the captured frames by using the exact same resources and rendering commands the app used. Then, using the tools provided in the Graphics Analyzer window, any of the captured frames can be analyzed in detail. These tools can be used to examine any Direct3D API call, resource, pipeline state object, pipeline stage, or even the complete history of any pixel in a captured frame. By using these tools in concert, a rendering problem can be explored intuitively, starting from how it appears in a captured frame and drilling down to its root cause in the app's source code, shaders, or graphics assets.

To diagnose performance problems, a captured frame can be analyzed by using the _Frame Analysis_ tool. This tool explores potential performance optimizations by automatically changing the way the app uses Direct3D and benchmarking all the variations for you. In the past, you might have made and benchmarked these kinds of changes manually just to find out which ones made a difference. With Frame Analysis, you only need to make the changes you already know will pay off.

Graphics Diagnostics helps your graphically-rich Direct3D app look and run its best.

Continue to [Overview](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/overview-of-visual-studio-graphics-diagnostics?view=vs-2015) to learn more about what Visual Studio Graphics Diagnostics offers.

## [](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/visual-studio-graphics-diagnostics?view=vs-2015#in-this-section)In This Section

[Overview](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/overview-of-visual-studio-graphics-diagnostics?view=vs-2015)  
Introduces the Graphics Diagnostics workflow and tools.

[Getting Started](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/getting-started-with-visual-studio-graphics-diagnostics?view=vs-2015)  
In this section, you'll learn how to install Visual Studio Graphics Diagnostics and how to start using Graphics Diagnostics with your Direct3D app.

[Capturing Graphics Information](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/capturing-graphics-information?view=vs-2015)  
To use Graphics Diagnostics to examine a rendering problem in your app, you first record information about how the app uses DirectX. During the recording session, as your app runs normally, you _capture_ (that is, select) the frames that you're interested in. The captures contain detailed information about how the frames are rendered. You can save the captured information as a graphics log document to examine later or share with other members of your team.

[GPU Usage](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/gpu-usage?view=vs-2015)  
To use Graphics Diagnostics to profile your app, use the GPU Usage tool. GPU usage can be used in concert with other profiling tools, such as CPU Usage, to correlate CPU and GPU activity that might contribute to performance problems in your app.

[Graphics Log Document](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/graphics-log-document?view=vs-2015)  
To start the examination of a recorded graphics log, you use the Graphics Log document window to select a captured frame—or even a specific pixel—so that you can examine in detail the _events_ (that is, the DirectX API calls) that affect it.

[Frame Analysis](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/graphics-frame-analysis?view=vs-2015)  
After you select a frame, you use Graphics Frame Analysis to examine and tune your rendering performance.

[Event List](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/graphics-event-list?view=vs-2015)  
After you select a frame, you use the **Graphics Event List** to examine its events to determine whether they are related to the rendering problem.

[State](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/graphics-state?view=vs-2015)  
The State window helps you understand the graphics state that is active at the time of the current event.

[Pipeline Stages](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/graphics-pipeline-stages?view=vs-2015)  
In the **Graphics Pipeline Stages** window, you investigate how the currently selected event is processed by each stage of the graphics pipeline so that you can identify where the rendering problem first appears. Examining the pipeline stages is particularly helpful when an object doesn't appear because of an incorrect transformation, or when one of the stages produces output that doesn't match what the next stage expects.

[Event Call Stack](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/graphics-event-call-stack?view=vs-2015)  
You use the **Graphics Event Call Stack** to examine the call stack of the currently selected event so that you can navigate to app code that's related to the rendering problem.

[Pixel History](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/graphics-pixel-history?view=vs-2015)  
By using the **Graphics Pixel History** window to analyze how the currently selected pixel is affected by the events that influenced it, you can identify the event or combination of events that cause certain kinds of rendering problems. The pixel history is particularly helpful when an object is rendered incorrectly because pixel shader output is either incorrect or has been combined incorrectly with the frame buffer, or when an object doesn't even appear because its pixels have been discarded before they reach the frame buffer.

[Object Table](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/graphics-object-table?view=vs-2015)  
You use the **Graphics Object Table** to examine the properties and contents of specific Direct3D objects and resources that are in effect for the currently selected event. The object table can help you determine the graphics device context that's active during an event, and examine the contents of graphics resources such as constant buffers, vertex buffers, and textures.

[HLSL Debugger](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/hlsl-shader-debugger?view=vs-2015)  
To examine how the shader code for the currently selected event and graphics pipeline stage behaves, you use the **HLSL Debugger** to step through code, examine the contents of variables, and perform other typical debugging tasks. You can also use the HLSL debugger to examine compute shader code, regardless of whether the results are further processed by the graphics pipeline or are just read back by your app.

[Command-Line Capture Tool](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/command-line-capture-tool?view=vs-2015)  
Use the command-line capture tool to quickly capture and play back graphics information without using Visual Studio or programmatic capture. In particular, you can use the command-line capture tool for automation, or in a test environment.

[Examples](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/graphics-diagnostics-examples?view=vs-2015)  
Several examples demonstrate how to use the Graphics Diagnostics tools together to diagnose different kinds of rendering problems.