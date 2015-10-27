# Sky Box and some optimizations

## Skybox

A skybox will allow us to set a background to give the illusion that our 3D world is bigger. That background is wrapped around the camera position and covers the whole space.  The technique that we are going to use here is to construct a big cube that will be displayed around the 3D scene, that is,  the centre of the camera position will be the centre of the cube. The sides of that cube will be a texture with hills a blue sky and clouds that will be mapped in a way that the image looks a continuous landscape.

The following picture depicts the skybox concept.

![Sky Box](skybox.png) 

So the process of creating a sky box is basically as follows:
* Create a big cube.
* Apply a texture to  it that provides the illusion that we are seeing a giant landscape with no edges.
* Render the cube so its sides are at a far distance and with the origin located at the centre  of the camera.

Then, let’s start with the texture. You will find that there are lots of textures pre-generated for you to use in the internet. The one used in the sample has been downloaded from here: [http://www.custommapmakers.org/skyboxes.php](http://www.custommapmakers.org/skyboxes.php). The concrete sample that we have used is this one: [http://www.custommapmakers.org/skyboxes/zips/ely_hills.zip](http://www.custommapmakers.org/skyboxes/zips/ely_hills.zip) and has been created by Colin Lowndes. 

The textures from that site are composed by separate TGA files, one for each side of the cube.  The texture loader that we have created expects a single file in PNG format so we need to compose a single PNG image with the images of each face. We could apply other techniques, such us cube mapping, in order to apply those texture method but they will be explained in later chapters. The result image is something like this.

![Sky Box Texture](skybox_texture.png) 

Then we need to create a .obj file which contains a cube with the correct texture coordinates for each face. The picture below shows the tiles associated to each face (you can find the .obj file used in this chapter in the book’s source code).

![Sky Box cube faces](skybox_cube_faces.png) 

We will create a new class named ```SkyBox``` with a constructor that receives the  path to the OBJ model  that contains the sky box cube and the texture file. This class will inherit from ```GameItem``` as the HUD class from the previous chapter. Why it should inherit from ```GameItem``` ? First of all, for convenience,  we can reuse most of the code that deals with meshes and textures. Secondly, because, although the skybox will not move we will be interested in applying rotations and scaling to it. If you think about it a ```SkyBox``` is indeed a game item. The definition of the ```SkyBox``` class is as follows.

```java
package org.lwjglb.engine;

import org.lwjglb.engine.graph.Material;
import org.lwjglb.engine.graph.Mesh;
import org.lwjglb.engine.graph.OBJLoader;
import org.lwjglb.engine.graph.Texture;

public class SkyBox extends GameItem {

    public SkyBox(String objModel, String textureFile) throws Exception {
        super();
        Mesh skyBoxMesh = OBJLoader.loadMesh(objModel);
        Texture skyBoxtexture = new Texture(textureFile);
        skyBoxMesh.setMaterial(new Material(skyBoxtexture, 0.0f));
        setMesh(skyBoxMesh);
        setPosition(0, 0, 0);
    }
}
```

If you check the source code for this chapter you will see that we have done some refactoring. We have created a class named ```Scene``` which groups all the information related to the 3D world. This the definition and the attributes of the ```Scene``` class, that contains an instance of the ```SkyBox``` class.

```java
package org.lwjglb.engine;

public class Scene {

    private GameItem[] gameItems;
    
    private SkyBox skyBox;
    
    private SceneLight sceneLight;

    public GameItem[] getGameItems() {
        return gameItems;
    }

    // More code here...
```

The next step is to create another set of vertex and fragment shaders for the skybox. But, why not reuse the scene shaders that we already have. The answer is that, actually, the shaders that we will need a simplified version of those shaders, we will not be applying lights to the light box (or to be more precise, we don’t need point, spot or directional lights).  Below you can see the skybox vertex shader.

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec2 texCoord;
layout (location=2) in vec3 vertexNormal;

out vec2 outTexCoord;

uniform mat4 modelViewMatrix;
uniform mat4 projectionMatrix;

void main()
{
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    outTexCoord = texCoord;
}
```

You can see that we still use the model view matrix. You may see some other implementations that increase the size of the cube that models the sky box at start time and do not need to multiply the model and the view matrix. We have chosen this approach because it’s more flexible and it allow s to change the size of the skybox at runtime, but you can easily switch to the other approach if you want. 

The fragment shader is also very simple.

```glsl
#version 330

in vec2 outTexCoord;
in vec3 mvPos;
out vec4 fragColor;

uniform sampler2D texture_sampler;
uniform vec3 ambientLight;

void main()
{
    fragColor = vec4(ambientLight, 1) * texture(texture_sampler, outTexCoord);
}
```

As you can see, we added an ambient light uniform to the shader. The purpose of this uniform is to modify the colour of the texture to simulate day and night (If not, the skybox will look like if were at midday when the rest of the world is dark).


In the  ```Renderer``` class we just have added a new method to use those shaders and setup the uniforms (nothing new here).

```java
private void setupSkyBoxShader() throws Exception {
    skyBoxShaderProgram = new ShaderProgram();
    skyBoxShaderProgram.createVertexShader(Utils.loadResource("/shaders/sb_vertex.vs"));
    skyBoxShaderProgram.createFragmentShader(Utils.loadResource("/shaders/sb_fragment.fs"));
    skyBoxShaderProgram.link();

    skyBoxShaderProgram.createUniform("projectionMatrix");
    skyBoxShaderProgram.createUniform("modelViewMatrix");
    skyBoxShaderProgram.createUniform("texture_sampler");
    skyBoxShaderProgram.createUniform("ambientLight");
}
```

And of course, we need to create a new render method for the skybox that will be invoked in the global render method.

```java
private void renderSkyBox(Window window, Camera camera, Scene scene) {
    skyBoxShaderProgram.bind();

    skyBoxShaderProgram.setUniform("texture_sampler", 0);

    // Update projection Matrix
    Matrix4f projectionMatrix = transformation.getProjectionMatrix(FOV, window.getWidth(), window.getHeight(), Z_NEAR, Z_FAR);
    skyBoxShaderProgram.setUniform("projectionMatrix", projectionMatrix);
    SkyBox skyBox = scene.getSkyBox();
    Matrix4f viewMatrix = transformation.getViewMatrix(camera);
    viewMatrix.m30 = 0;
    viewMatrix.m31 = 0;
    viewMatrix.m32 = 0;
    Matrix4f modelViewMatrix = transformation.getModelViewMatrix(skyBox, viewMatrix);
    skyBoxShaderProgram.setUniform("modelViewMatrix", modelViewMatrix);
    skyBoxShaderProgram.setUniform("ambientLight", scene.getSceneLight().getAmbientLight());
                
    scene.getSkyBox().getMesh().render();

    skyBoxShaderProgram.unbind();
}
```

The method above  is quite similar to the other render ones but there’s a difference that needs to be explained. As you can see pass the projection matrix and the model view matrix as usual. But, when we get the view matrix, we set some of the components to 0. Why we do this ? The reason behind that is that we do not want translation to be applied to the sky box.

Remember  that when we move the camera, what we are actually doing is moving the whole world. So if we just multiply the view matrix as it is, the skybox will be displaced. But we do not want this, we want to stick it at the coordinates origin at (0, 0, 0). This achieved by removing setting to 0 the parts of the view matrix that contain the translation increments (the ```m30```, ```m31``` and ```m32``` components). 

You may think that you could avoid using the view matrix at all since the sky box must be fixed at the origin. In that case what you will see is that the skybox will not rotate with the camera, which is not what we want. We need it to rotate but not translate.

And that’s all, you can check in the source code for this chapter that in the ```DummyGame``` class that we have created more block instances to simulate a ground and the skybox. You can also check that we now change the ambient light to simulate light and day. What we get is something like this.

![Sky Box result](skybox_result.png) 

The sky box is a small one so can easily see the effect of moving through the world (in a real game it should be much bigger).  You can see also that the world space objects, the blocks that form the terrain are larger than the skybox, so as you move through it you will see block appearing through the mountains. This is more evident because of the relative small size of the sky box we have set, but anyway we will need to smooth that by adding an effect that hides or blur distant objects (for instance applying a fog effect).

Another reason for not creating a bigger sky box is because we need to apply several optimizations in order to be more efficient (they will be explained later on).

You can play with the render method an comment the lines that prevent the skybox from moving. Then you will be able to get out of the box and see something like this.

![Sky Box displaced](skybox_displaced.png) 

Although it is not what a sky box should do it can help you out to understand the sky box technique. This is a simple example, we will need to add other effects such as a sun moving through the sky or moving clouds. Also, in order to create bigger worlds we will need to split our world into fragments and only load the ones that are contiguous to the fragment where the player is currently in. We will try to introduce all these techniques later on.

## Some optimizations

From the previous example, the fact that the skybox is relative small makes the effect a little bit weird (you can see objects appearing magically from the hills). So, ok, let’s increase the skybox size and thus let’s increase the size of our world. Let’s scale the size of our skybox by a factor of 50 so our world will be composed by 40,000 GameItem instances (cubes).

If you change the scale factor and rerun the example you will see that performance problem starts to arise and the movement through the 3D world is not smooth. It’s time to focus a little on performance (you may know the old saying that states that “premature optimization is the root of all evil”, but since this chapter 13, I hope nobody will say that this premature).

Let’s start with a concept that will reduce the amount of data that is being rendered, we will explain face culling. In our examples we are rendered thousands of cubes, and a cube is made of six faces. We are rendering the six faces for each cube even if they are not visible. You can check this if you zoom in to a cube, you will see its interior like this.

![Cube interior](cube_interior.png) 

Faces that cannot be seen should be discarded immediately and this is what face culling does. In fact, for a cube you can only see 3 faces at the same time, so we can just discard half of the faces (40,000 * 3 * 2 triangles) just by applying face culling (this will only be valid if your game does not require you to dive into the inner side of a model, you can see why later on).

Face culling checks, for every triangle if its facing towards us and discards the ones that are not facing that direction. But, how do we know if a face is facing towards us or not ? Well, the way that OpenGL does this is by the winding order of the vertices that compose a triangle.

Remember from the first chapters that we may define of vertices in clockwise or counter-clockwise order. In OpenGL, by default, triangles that are in counter-clockwise order are facing towards the viewer and triangles that are in clockwise order are facing backwards. The key thing here, is that this order is checked while rendering and taking into consideration the point of view. So a triangle that has been defined in counter-clock wise order can be seen, at rendering, as clockwise because of the point of view.
Let’s put it in practice, in the ```init``` method of the ```Window``` class add the following lines:

```java
glEnable(GL_CULL_FACE);
glCullFace(GL_BACK);
```
The first line will enable face culling and the second line states that faces that are facing backwards should be culled (removed). With that line if you look upwards you will see something like this.

![Sky box with face culling applied](skybox_face_culling.png)

What’s happening ? if you review the vertices order for the top face you will see that is has been defined in counter-clockwise order.  Well, it was, but remember that the winding refers to the point of view. In fact, if you apply translation also to the skybox so you are able to see it form the upside you will see that the top face is rendered again once you are outside it.
 
![Skybox with face culling from the outside](skybox_face_culling_exterior.png)

Let’s sketch what’s happening. The following picture shows on of the triangles of the top face of  the skybox cube, which is defined by three vertices defined in counter-clockwise order.

![Vertices defined in counter-clockwise order](cube_counter_clockwise.png) 

But remember that we are inside the skybox, if we look at the cube form the interior, what we will see is that the vertices are defined in clockwise order.

![Cube seen from the interior](cube_clockwise.png) 

This is because, the skybox was defined to be looked from the outside. So we need to flip the definition for some of the faces in order to be viewed correctly and we will have face culling working properly.