
# Rendering 

In this chapter we will learn the processes that takes place while rendering a scene using OpenGL. If you are used to older versions of OpenGL, that is fixed-function pipeline, you may end this chapter wondering why it needs to be so complex. You may end up thinking that drawing a simple shape to the screen should not require so many concepts and line of codes. Let me give you an advice for those of you that think that way, it is actually simpler and much more flexible. You only need to give it a chance. Modern OpenGL lets you think in one problem at a time and it lets you organize your code and processes in a more logical way.

The sequence of steps that ends up drawing a 3D representation into your 2D screen is called the graphics pipeline. First versions of OpenGL employed a model which was called fixed-function pipeline. This model employed a set of steps in the rendering process which defined a fixed set of operations. The programmer was constrained to the set of functions available for each step. Thus, the effects and operations that could be applied were limited by the API itself (for instance, “set fog” or “add light”, but the implementation of those functions were fixed and could not be changed).

The graphics pipeline was composed by these steps:

![Graphics Pipeline](rendering_pipeline.png)


Open GL 2.0 introduced the concept of programmable pipeline. In this model, the different steps that compose the graphics pipeline can be controlled or programmed by using a set of specific programs called shaders. The following picture depicts a simplified version of the OpenGL programmable pipeline:

![Programmable pipeline](rendering_pipeline_2.png) 

The rendering starts taking as its input a list of vertices in the form of Vertex Buffers. But, what is a vertex? A vertex is a data structure that describes a point in 2D or 3D space. And how do you describe a point in a 3D space? By specifying it’s coordinates x, y and z. And what is a Vertex Buffer? A Vertex Buffer is another data structure that packs all the vertices that need to be rendered, by using vertex arrays, and makes that information available to the shaders in the graphics pipeline.

Those vertices are processed by the vertex shader which main purpose is to calculate the projected position of each vertex into the screen space. This shader can generate also other outputs related to colour or texture, but it’s main goal is to project the vertices into the screen space, that is, to generate dots.

The geometry processing stage connects the vertices that are transformed by the vertex shader to form triangles. It does so by taking into consideration the order in which the vertices were stored and grouping them using different models. Why triangles? Triangles is like the basic work unit for graphic cards, it’s a simple geometric shape that can be combined and transformed to construct complex 3D scenes. This stage can also use a specific shader to group the vertices.

The rasterization stage takes the triangles generated in the previous stages, clips them and transforms them into pixel-sized fragments.

Those fragments are used during the fragment processing stage by the fragment shader to generate pixels assigning them the final that get into the framebuffer. The framebuffer is the final result of the graphics pipeline it holds the value of each pixel that should be drawn to the screen.

Keep in mind that 3D cards are designed to parallelize all the operations described above. The input data can be processes in parallel in order to generate the final scene.

So let uss start writing our first shader program. Shaders are written by using the GLSL language (OpenGL Shading Language) which is based on ANSI C. First we will create a file named “*vertex.vs*” (The extension is for Vertex Shader) under the resources directory with the following content:

```
#version 330

layout (location=0) in vec3 pos;

void main()
{
	gl_Position = vec4(position, 1.0);
}
```

The first line is a directive that states the version of the GLSL language we are using. The following table relates the GLSL version, the OpenGL that matches that version and the directive to use (Wikipedia: https://en.wikipedia.org/wiki/OpenGL_Shading_Language#Versions).

| GLS Version | OpenGL Version | Shader Preprocessor |
| -- | -- | -- |
| 1.10.59 | 2.0 | #version 110 |
| 1.20.8 | 2.1 | #version 120 |
| 1.30.10 | 3.0 | #version 130 |
| 1.40.08 | 3.1 | #version 140 |
| 1.50.11| 3.2 | #version 150 |
| 3.30.6 | 3.3 | #version 330 |
| 4.00.9 | 4.0 | #version 400 |
| 4.10.6 | 4.1 | #version 410 |
| 4.20.11 | 4.2 | #version 420 |
| 4.30.8 | 4.3 | #version 430 |
| 4.40 | 4.4 | #version 440 |
| 4.50 | 4.5 | #version 450 |


The second line specifies the input format for this shader. Data in an OpenGL buffer can be whatever we want, that is, the language does not force you to pass a specific data structure with a predefined semantic. From the point of view of the shader it is expecting to receive a buffer with data. It can be a position, a position with some additional information or whatever we want. The vertex is just receiving an array of floats, when we fill the buffer, we define the buffer chunks that are going to be processed by the shader.

So, first we need to get that chunk into something that’s meaningful to us. In this case we are saying that, starting from the position 0, we are expecting to receive a vector composed by 3 attributes (x, y, z). 

The shader has a main block like any other C program which in this case is very simple. It is just returning the received position in the output variable *gl_Position* without applying any transformation. You now may be wondering why the vector of three attributes has been converted into a vector of four attributes (vec4). This is because *gl_Position* is expecting the result in vec4 format since it is using homogeneous coordinates. That is, is expecting something in the form (x, y, z, w), where w represents an extra dimensions that allows us to properly deal with points at infinity.

Let us now have a look about our first fragment shader. We will create a file named “*fragment.fs*” (The extension is for Fragment Shader) under the resources directory with the following content:


#version 330

out vec4 fragColor;

void main()
{
	fragColor = vec4(0.0, 0.5, 0.5, 1.0);
}

The structure is quite similar to our vertex shader. In this case we will set a fixed colour for each fragment. The output variable is defined in second line and set as a vec4  fragColor.
Now that we have our shaders created, how do we use them? This is the sequence of steps we need to follow:
1.	Create a OpenGL Program
2.	Load the vertex and shader code files.
3.	For each shader, create a new shader program and specify its type (vertex, shader).
4.	Compile the shader.
5.	Attach the shader to the program.
6.	Link the program.
At the end the shader will be loaded in the graphics card and we can use by referencing an identifier, the program identifier.
package org.lwjglb.engine.graph;

import static org.lwjgl.opengl.GL20.*;

public class ShaderProgram {

    private final int programId;

    private int vertexShaderId;

    private int fragmentShaderId;

    public ShaderProgram() throws Exception {
        programId = glCreateProgram();
        if (programId == 0) {
            throw new Exception("Could not create Shader");
        }
    }

    public void createVertexShader(String shaderCode) throws Exception {
        vertexShaderId = createShader(shaderCode, GL_VERTEX_SHADER);
    }

    public void createFragmentShader(String shaderCode) throws Exception {
        fragmentShaderId = createShader(shaderCode, GL_FRAGMENT_SHADER);
    }

    protected int createShader(String shaderCode, int shaderType) throws Exception {
        int shaderId = glCreateShader(shaderType);
        if (shaderId == 0) {
            throw new Exception("Error creating shader. Code: " + shaderId);
        }

        glShaderSource(shaderId, shaderCode);
        glCompileShader(shaderId);

        if (glGetShaderi(shaderId, GL_COMPILE_STATUS) == 0) {
            throw new Exception("Error compiling Shader code: " + glGetShaderInfoLog(shaderId, 1024));
        }

        glAttachShader(programId, shaderId);

        return shaderId;
    }

    public void link() throws Exception {
        glLinkProgram(programId);
        if (glGetProgrami(programId, GL_LINK_STATUS) == 0) {
            throw new Exception("Error linking Shader code: " + glGetShaderInfoLog(programId, 1024));
        }

        glValidateProgram(programId);
        if (glGetProgrami(programId, GL_VALIDATE_STATUS) == 0) {
            throw new Exception("Error validating Shader code: " + glGetShaderInfoLog(programId, 1024));
        }

    }

    public void bind() {
        glUseProgram(programId);
    }

    public void unbind() {
        glUseProgram(0);
    }

    public void cleanup() {
        unbind();
        if (programId != 0) {
            if (vertexShaderId != 0) {
                glDetachShader(programId, vertexShaderId);
            }
            if (fragmentShaderId != 0) {
                glDetachShader(programId, fragmentShaderId);
            }
            glDeleteProgram(programId);
        }
    }
}

The constructor of the ShaderProgram creates a new program in OpenGL and provides methods to add vertex and fragment shaders. Those shaders are compiled and attached to the OpenGL program. When all shaders are attached the link method should be invoked which links all the code and verifies that everything has been done correctly. ShaderProgram also provides methods to activate this program for rendering (bind) and to stop using it (unbind). Finally it provides a cleanup method to free all the resources when they are no longer needed.
Since we have a cleanup method, let’s change our IGameLogic interface class to add a cleanup method:
      void cleanup();

This method will be invoked when the game loop finishes, so we need to modify the run method of the GameEngine class:
    @Override
    public void run() {
        try {
            init();
            gameLoop();
        } catch (Exception excp) {
            excp.printStackTrace();
        } finally {
            cleanup();
        }
    }

Now we can use or shaders in order to display a triangle. We will do this in the init method of our Renderer class. First of all, we create the Shader program:
    public void init() throws Exception {
        shaderProgram = new ShaderProgram();
        shaderProgram.createVertexShader(
		    Utils.loadResource("/vertex.vs"));
        shaderProgram.createFragmentShader(
		    Utils.loadResource("/fragment.fs"));
        shaderProgram.link();

We have created an utility class which by now provides a method to retrieve the contents of a file from the class path. This method is used to retrieve the contents of our shaders.
Now we can define our triangle as an array of floats. We create a single float array which will define the vertices of the triangle. As you can see there’s no structure in that array, as it is right now, OpenGL cannot know the structure of that data, it’s just a sequence of floats:
        float[] vertices = new float[]{
            0.0f, 0.5f, 0.0f,
            -0.5f, -0.5f, 0.0f,
            0.5f, -0.5f, 0.0f
        };

The following picture depicts the triangle in our coordinate system:

 
Now that we have our coordinates, we need to store them into our graphics card and tell OpenGL about the structure. We will introduce now two important concepts Vertex Array Objects (VAOs) and Vertex Buffer Object (VBOs). If you get lost in the next code fragments remember that at the end what we are doing is sending the data that models the objects we want to draw to the graphics card memory. When we store it we get an identifier that serves us later to refer to it while drawing.
Let’s first start with Vertex Buffer Object (VBOs). A VBO is just a memory buffer stored in the graphics card memory that stores vertices. This is where we will transfer our array of floats that model a triangle. As we have said before OpenGL does not know anything about our data structure, in fact it can hold not just coordinates but other information, such as textures, colour, etc.
A Vertex Array Objects (VAOs). A VAO is an object that contains one or more VBOs which are usually called attribute lists. Each attribute list can hold one type of data: position, colour, texture, etc. You are free to store whichever you want in each slot.
 
A VAO is like a wrapper that groups a set of definitions for the data is going to be stored in the graphics card. When we create a VAO we  get an identifier, we use that identifier to render it and the elements it contains using the definitions we specified during its creation.
So let’s continue coding our example. The first thing that we must do with is to store our array of floats into a FloatBuffer. This is mainly due to the fact that we must interface with OpenGL library, which is C-bases, so we must transform our array of floats into something that can be managed by the library.
FloatBuffer verticesBuffer =
	BufferUtils.createFloatBuffer(vertices.length);
verticesBuffer.put(vertices).flip();

We use a utility class to create the buffer and after we have stored the data (with the put method) we need to reset the position of the buffer to the 0 position with the flip method (that is, we say that we’ve finishing writing on it).
Now  we need to create the VAO and bind to it
vaoId = glGenVertexArrays();
glBindVertexArray(vaoId);


Then, we need to create or VBO, bind to it and put the data into it:
vboId = glGenBuffers();
glBindBuffer(GL_ARRAY_BUFFER, vboId);
glBufferData(GL_ARRAY_BUFFER, verticesBuffer, GL_STATIC_DRAW);

Now it comes the most important part, we need to define the structure of our data and store in one of the attribute lists of the VAO, this is done with the following sentence:
glVertexAttribPointer(0, 3, GL_FLOAT, false, 0, 0);

The parameters are:
•	index: Specifies the location where the shader expects this data.
•	size: Specifies then number of components per vertex attribute (from 1 to 4). In this case, we are passing 3D coordinates, so it should be 3.
•	type: Specifies the type of each component in the array, in this case a float.
•	normalized: Specifies if the values should be normalized or not.
•	stride: Specifies the byte offset between consecutive generic vertex attributes. (We will explain it later).
•	offset: Specifies a offset of the first component of the first component in the array in the data store of the buffer.
After we have finished with our VBO we can unbind it and the VAO (bind them to 0)
// Unbind the VBO
glBindBuffer(GL_ARRAY_BUFFER, 0);

// Unbind the VAO
glBindVertexArray(0);

That’s all the code that should be in our init method. Our data is already in the graphical card, ready to be used. We only need to modify our render method to use it each render step during our game loop.
    public void render() {
        clear();

        shaderProgram.bind();

        // Bind to the VAO
        glBindVertexArray(vaoId);
        glEnableVertexAttribArray(0);

        // Draw the vertices
        glDrawArrays(GL_TRIANGLES, 0, 3);

        // Restore state
        glDisableVertexAttribArray(0);
        glBindVertexArray(0);

        shaderProgram.unbind();
    }

As you can see we just clear the window, bind the shader program, bind the VAO, draw the vertices stored in the VBO associated to the VAO and restore the state. That’s it.
We also added a cleanup method to our Renderer class which frees acquired resources:
    public void cleanup() {
        if (shaderProgram != null) {
            shaderProgram.cleanup();
        }

        glDisableVertexAttribArray(0);

        // Delete the VBO
        glBindBuffer(GL_ARRAY_BUFFER, 0);
        glDeleteBuffers(vboId);

        // Delete the VAO
        glBindVertexArray(0);
        glDeleteVertexArrays(vaoId);
    }
}

And, that’s all ! If you have followed the steps carefully you will see something like this.
 
Our first triangle! You may think that this won’t make it into the top ten game list, and you will be totally right. You may also think that this has been too much work for drawing a boring triangle, but keep in mind that we are introducing key concepts and preparing the base infrastructure to do more complex things. Please be patience and continue reading.







