# Appendix A - OpenGL Debugging

Debugging an OpenGL program can be a daunting task. Most of the times you end up with a black screen and you have no means of knowing what’s going on. In order to alleviate this problem we can use some existing tools that will provide more information about the rendering process.

In this annex we will describe how to use the [RenderDoc](https://renderdoc.org/ "RenderDoc") tool to debug our LWJGL programs. RenderDoc is a graphics debugging tool that can be used with Direct3D, Vulkan and OpenGL. In the case of OpenGL it only supports the core profile from 3.2 up to 4.5.

So let’s get started. You need to download and install the RenderDoc version for your OS. Once installed, when you launch it you will see something similar to this.

![](/appendixa/renderdoc.png)

The first step is to configure RenderDoc to execute and monitor our samples. In the “Capture Executable” tab we need to setup the following parameters:

* **Executable path**: In our case this should point to the JVM launcher \(For instance, “C:\Program Files\Java\jdk-9\bin\java.exe”\).
* **Working Directory**: This is the working directory that will be setup for your program. In our case it should be set to the target directory where maven dumps the result. By setting this way, the dependencies will be able to be found.
* **Command line arguments**: This will contain the arguments required by the JVM to execute our sample. In our case, just passing the jar to be executed \(For instance, “-jar game-c28-1.0.jar”\).

![](/appendixa/exec_arguments.png)

You must remember that 3D models are loaded now with [Assimp](http://assimp.sourceforge.net/ "Assimp"), and we need real file path \(no more `CLASSPATH`related paths\), so you need to check the routes from the working directory you have set up. In this case, the easiest approach to quickly test is to copy the src folder into the target directory.

There are many other options int this tab to configure the capture options. You can consult their purpose in [RenderDoc documentation](https://renderdoc.org/docs/index.html "RenderDoc documentation"). Once everything has been setup you can execute your program by clicking on the “Launch” button. You will see something like this:

![](/appendixa/sample.png)

You may see a Warning since RenderDoc can only work with OpenGL core profile. In the sample we’ve enabled compatibiñity profile, but it should work even with that warning. Once the program is being executed you can trigger snapshots of it. You will see that a new tab has been added which is named “java \[PID XXXX\]” \(where the XXXX number represents the PID, the process identifier, of the java process\).

![](/appendixa/java_process.png)

From that tab you can capture the state of your program by pressing the “Trigger capture” button. Once a capture has been generated, you will see a little snapshot in that same tab.

![](/appendixa/capture.png)

If you double click on that capture, all the data collected will be loaded and you can start inspecting it. The “Event Browser” panel will be populated will all the relevant OpenGL calls executed during one rendering cycle.

![](/appendixa/event_browser.png)

You can see, for the first rendering pass, how the floor is drawn and later on the mesh that models the house. If you click over a glDrawELements event, and select the “Mesh” tab you can see the mesh that was drawn, its input and output for the vertex shader.

You can also view the input textures used for that drawing operation \(by clicking the “Texture Viewer” tab\).

![](/appendixa/texture_inputs.png)

In the center panel, you can see the output, and on the right panel you can see the list of textures used as an input. You can also view the output textures one by one. This is very illustrative to show how deferred shading works.

![](/appendixa/texture_outputs.png)

As you can see, this tool provides valuable information about what’s happening when rendering. It can save precious time while debugging rendering problems. It can even display information about the shaders used in the rendering pipeline.

![](/appendixa/pipeline_state.png)

