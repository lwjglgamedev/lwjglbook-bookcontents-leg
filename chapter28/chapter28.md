# Deferred Shading

Up to now the way that we are rendering a 3D scene is called forward rendering. We first render the  3D objects and apply the texture and lightning effects in a fragment shader.    This method is not very efficient if we have a complex fragment shader pass with many lights and complex effects. In addition to that we may end up applying these effects to fragments that may be later on discarded due to depth testing \(although this is not exactly true if we enable [early fragment testing](https://www.khronos.org/opengl/wiki/Early_Fragment_Test)\).

In order to alleviate the problems described above we may change thy way that we render the scene by using a technical called deferred shading. With deferred shading we first render the geometry information that is required in later stages \(in the fragment shader\) to a buffer.  The complex calculus required by the fragment shader are postponed, deferred, to a later stage when using the information stored in those buffers.

Hence, with deferred shading we perform two rendering passes. The first one, is the geometry pass, where we render the scene to a buffer that will contain the following information:

* The positions \(in our case in light view coordinate system, although you may see other samples where world coordinates are used\).
* The diffuse colours for each position.
* The specular component for each position.
* The normals at each position \(also in light view coordinate system\).
* Shadow map for the directional light \(you may find that this step is done separately in other implementations\).

All that information is stored in a buffer called G-Buffer.

The second pass, is called, the lightning pass. This pass takes a quad that fills up all the screen and generates the colour information for each fragment using the information contained in the G-Buffer.  When we will be performing the lightning pass, the depth test will have already removed all the scene data that would not be seen. Hence, the number of operations to be done are restricted to what will be displayed on the screen.

You may be asking if performing additional rendering passes will result in an increase of performance or not. The answer is that it depends. Deferred shading is usually used when you have many different light passes. In this case, the additional rendering steps are compensated by the reduction of operations that will be done in the fragment shader.

So let’s start coding. The first task that we will be doing is create a new class for the G-Buffer. The class, named `GBuffer`, is defined like this:

```java
package org.lwjglb.engine.graph;

import org.lwjgl.system.MemoryStack;
import org.lwjglb.engine.Window;
import java.nio.ByteBuffer;
import java.nio.IntBuffer;
import static org.lwjgl.opengl.GL11.*;
import static org.lwjgl.opengl.GL20.*;
import static org.lwjgl.opengl.GL30.*;

public class GBuffer {

    private static final int TOTAL_TEXTURES = 6;

    private int gBufferId;

    private int[] textureIds;

    private int width;

    private int height;
```

The class defines a constant that models the maximum number of buffers to be used. The identifier associated to the G-Buffer itself and an array for the individual buffers. The size of the screen is also stored.

Let’s review the constructor:

```java
public GBuffer(Window window) throws Exception {
    // Create G-Buffer
    gBufferId = glGenFramebuffers();
    glBindFramebuffer(GL_DRAW_FRAMEBUFFER, gBufferId);

    textureIds = new int[TOTAL_TEXTURES];
    glGenTextures(textureIds);

    this.width = window.getWidth();
    this.height = window.getHeight();

    // Create textures for position, diffuse color, specular color, normal, shadow factor and depth
    // All coordinates are in world coordinates system
    for(int i=0; i<TOTAL_TEXTURES; i++) {
        glBindTexture(GL_TEXTURE_2D, textureIds[i]);
        int attachmentType;
        switch(i) {
            case TOTAL_TEXTURES - 1:
                // Depth component
                glTexImage2D(GL_TEXTURE_2D, 0, GL_DEPTH_COMPONENT32F, width, height, 0, GL_DEPTH_COMPONENT, GL_FLOAT,
                        (ByteBuffer) null);
                attachmentType = GL_DEPTH_ATTACHMENT;
                break;
            default:
                glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB32F, width, height, 0, GL_RGB, GL_FLOAT, (ByteBuffer) null);
                attachmentType = GL_COLOR_ATTACHMENT0 + i;
                break;
        }
        // For sampling
        glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
        glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);

        // Attach the the texture to the G-Buffer
        glFramebufferTexture2D(GL_FRAMEBUFFER, attachmentType, GL_TEXTURE_2D, textureIds[i], 0);
    }

    try (MemoryStack stack = MemoryStack.stackPush()) {
        IntBuffer intBuff = stack.mallocInt(TOTAL_TEXTURES);
        int values[] = {GL_COLOR_ATTACHMENT0, GL_COLOR_ATTACHMENT1, GL_COLOR_ATTACHMENT2, GL_COLOR_ATTACHMENT3, GL_COLOR_ATTACHMENT4, GL_COLOR_ATTACHMENT5};
        for(int i = 0; i < values.length; i++) {
            intBuff.put(values[i]);
        }
        intBuff.flip();
        glDrawBuffers(intBuff);
    }

    // Unbind
    glBindFramebuffer(GL_FRAMEBUFFER, 0);
}
```

The first thing that we do is create a frame buffer. Remember that a frame buffer is just an OpenGL objects that can be used to render operations instead of rendering to the screen. Then we generate a set of textures \(6 textures\), that will be associated to the frame buffer.

After that, we use a foor loop to initialize the textures. We have the following types:

* “Regular textures”, that will store positions, normals, the diffuse component, etc.
* A texture for storing the depth buffer. This will be our last texture.

Once the texture has been initialized, we enable sampling for them and attach them to the frame buffer. Each texture is attached using and identifier which starts at `GL_COLOR_ATTACHMENT0`. Each texture increments by one that id, so the positions are attached using  `GL_COLOR_ATTACHMENT0`, the normals use  `GL_COLOR_ATTACHMENT1` \(which is  `GL_COLOR_ATTACHMENT0 + 1`\), and so on.

After all the textures have been created, we need to enable them to be used by the fargment shader for rendering. This is done with the  `glDrawBuffers`call. We just pass the array with the idefintifiers of the colour attachments used \(`GL_COLOR_ ATTACHMENT0` to  `GL_COLOR_ATTACHMENT5`\).

The rest of the class are just getter methods and the cleanup one.

```java
public int getWidth() {
    return width;
}

public int getHeight() {
    return height;
}

public int getGBufferId() {
    return gBufferId;
}

public int[] getTextureIds() {
    return textureIds;
}

public int getPositionTexture() {
    return textureIds[0];
}

public int getDepthTexture() {
    return textureIds[TOTAL_TEXTURES-1];
}

public void cleanUp() {
    glDeleteFramebuffers(gBufferId);

    if (textureIds != null) {
        for (int i=0; i<TOTAL_TEXTURES; i++) {
            glDeleteTextures(textureIds[i]);
        }
    }
}
```

Once the `Gbuffer `class is defined we can start using it. In the Renderer class, we will no longer be using the forward rendering shaders we were using for rendering the scene \(named `“scene_vertex.vs”` and `“scene_fragment.fs”`\).

In the `init` method of the `Renderer` class you may see that a `GBuffer` instance is created and that we initialize and another set of shaders for the geometry pass \(by calling the `setupGeometryShader` method\) and the light pass \(by calling the `setupDirLightShader` and `setupPointLightShader` methods\). An utility matrix named `bufferPassModelMatrix` is also instantiated \(it will be used when performing the geometry pass\). You can see that we create a new `Mesh` at the end of the init method. This will be used in the light pass. More on this will be explained later.

```java
public void init(Window window) throws Exception {
    shadowRenderer.init(window);
    gBuffer = new GBuffer(window);
    sceneBuffer = new SceneBuffer(window);
    setupSkyBoxShader();
    setupParticlesShader();
    setupGeometryShader();
    setupDirLightShader();
    setupPointLightShader();
    setupFogShader();

    bufferPassModelMatrix =  new Matrix4f();
    bufferPassMesh = StaticMeshesLoader.load("src/main/resources/models/buffer_pass_mess.obj", "src/main/resources/models")[0];
}
```



