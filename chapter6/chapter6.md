# Transformations

## Projecting

Let’s get back to our nice coloured quad we created in previous chapter. If you look carefully at it, it resembles more to a rectangle. You can even change the width of the window from 600 pixels to 900 and the distortion will be more evident. What’s happening here?

If you revisit our vertex shader code we are just passing our coordinates directly, that is when we say that a vertex has a value for coordinate x of 0.5 we are saying to OpenGL to draw it in x position 0.5 in our screen. The following figure shows OpenGL coordinates \(just for x and y axis\).

![Coordinates](coordinates.png)

Those coordinates are mapped, considering our window size, to window coordinates \(which have the origin at top-left corner of t he previous figure\), so if our window has a size of 900x480, OpenGL coordinates \(1,0\) will be mapped to coordinates \(900, 0\) creating a rectangle instead of a quad.

![Rectangle](rectangle.png)

But, the problem is more serious than that. Modify the z cords of our quad from 0.0 to 1.0 and to -1.0. What do see ? The quad is exactly drawn in the same place no matter if it’s displaced along the z axis. Why is happening this ? Objects that are further away should be drawn smaller than objects that are closer, but we are drawing them with the same x and y coordinates.

But, wait. Should not be this handled by the z-coord? The answer is yes an no, The z coordinate tells OpenGL that an object is closer or far away, but OpenGL does not know nothing about the size of your object you could have two objects of different sizes, one closer and smaller and one bigger and further that could be projected, correctly into the screen with the same size \(those would have same x and y coordinates and different z\). OpenGL just uses the coordinates we are passing, so we must take care of this, we need to correctly project our coordinates.

Now that we have diagnosed the problem, how do we do this ? The answer is by using a projection matrix or frustum. The projection matrix will take care of the aspect ratio \(the relation between size and height\) of our drawing area so objects won’t be distorted. It also will handle the distance so objects far away from us will be drawn smaller. The projection matrix will also consider our field of view and how far is the maximum distance that should be displayed.

For those not familiar with matrices, a matrix is a bi-dimensional array of numbers arranged in columns and rows, each number inside a matrix is called an element. A matrix order is the number of rows and columns. For instance, here you can see a 2x2 matrix \(2 rows and 2 columns\).

![2x2 Matrix](2_2_matrix.png)

Matrices have a number of basic operations that can be applied to them \(such as addition, multiplication, etc.\) that you can consult in any maths book. The main characteristics of matrices, related to 3D graphics, is that they are very useful to transform points in the space.

You can think about the projection matrix as a camera, which has a field of view and a minimum and maximum distance. The vision area of that camera will be a truncated pyramid, the following picture shows a top view of that area.

![Projection Matrix concepts](projection_matrix.png)

A projection matrix will correctly map 3D coordinates so they can be correctly represented into a 2D screen. The mathematical representation of that matrix is as follows \(don’t be scared\).

![Projection Matrix](projection_matrix_eq.png)

Where aspect ratio is the relation between our screen width and our screen height \($$a=width/height$$\). In order to obtain the projected coordinates of a given point we just need to multiply the projection matrix to the original coordinates. The result will be another vector that will contain the projected version.

So we need to handle a set of mathematical entities such as vectors, matrices and include the operations that can be done between them. We could chose to warite all that code by our own from scracth or use an already existing library. We will choose the easy path and use a specific library for dealing with math operations sin LWJGL which is called JOML \(Java OpenGL Math Library\). In order to use that librray we just need to add another dependency to our `pom.xml` file.

```xml
        <dependency>
            <groupId>org.joml</groupId>
            <artifactId>joml</artifactId>
            <version>${joml.version}</version>
        </dependency>
```

And define the version of the library to use.

```xml
    <properties>
        [...]
        <joml.version>1.7.1</joml.version>
        [...]
    </properties>
```

Now that everything has been set up let’s define our projection matrix. We will create a instance of the class `Matrix4f` \(provided by the JOML library\) in our `Renderer` class. The `Matrix4f` provides a method to set up a projection matrix named `perspective`. This methods need the following parameters:

* Field of View: The Field of View angle in radians. We will define a constant that holds that value
* Aspect Ratio.
* Distance to the near plane \(z-near\)
* Distance to the far plane \(z-far\).

We will instantiate  that matrix in our `init` method so we need to pass a reference to our `Window`  instance to get its size \(you can see it in the source code\). The new constants and variables are:

```java
    /**
     * Field of View in Radians
     */
    private static final float FOV = (float) Math.toRadians(60.0f);

    private static final float Z_NEAR = 0.01f;

    private static final float Z_FAR = 1000.f;

    private Matrix4f projectionMatrix;
```

The projection matrix is created as follows:

```java
float aspectRatio = (float) window.getWidth() / window.getHeight();
projectionMatrix = new Matrix4f().perspective(FOV, aspectRatio,
    Z_NEAR, Z_FAR);
```

At this moment we will ignore that the aspect ratio can change \(by resizing our window\). This could be checked in the `render` method and change our projection matrix accordingly.

Now that we have our matrix, how do we use it? We need to use it in our shader, and it should be applied to all the vertices. At first, you could think in bundling it in the vertex input \(like the coordinates and the colours\). In this case we would be wasting lots of space since the projection matrix should not change even between several render calls. You may also think on multiplying the vertices by the matrix in the java code but then, our VBOs would be useless and we will not be using the process power available in the graphics card.

The answer is to use “uniforms”. Uniforms are global GLSL variables that shaders can use and that we will employ to communicate with them.

So we need to modify our vertex shader code and declare a new uniform called projectionMatrix and use it to calculate the projected position.

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec3 inColour;

out vec3 exColour;

uniform mat4 projectionMatrix;

void main()
{
    gl_Position = projectionMatrix * vec4(position, 1.0);
    exColour = inColour;
}
```

As you can see we define our `projectionMatrix` as matrix of 4x4 elements and the position is obtained by multiplying it by our original coordinates. Now we need to pass the values of the projection matrix to our shader. First, we need to get a reference to the place where the uniform will hold its values.

This done with the method `glGetUniformLocation` which receives two parameters:

* The Shader program Identifier.
* The name of the uniform \(it should match the once defined in the shader code\).

This method returns an identifier holding the uniform location, since we may have more than one uniform, we will store those locations in a Map indexing by its name \(We will need that location number later\). So in the `ShaderProgram` class we create a new variable that holds those identifiers:

```java
private final Map<String, Integer> uniforms;
```

This variable will be initialized in our constructor:

```java
uniforms = new HashMap<>();
```

And finally we create a method to set up new uniforms and store the obtained location.

```java
public void createUniform(String uniformName) throws Exception {
    int uniformLocation = glGetUniformLocation(programId,
        uniformName);
    if (uniformLocation < 0) {
        throw new Exception("Could not find uniform:" +
            uniformName);
    }
    uniforms.put(uniformName, uniformLocation);
}
```

Now, in our `Renderer` class we can invoke the `createUniform` method once the shader program has been compiled \(in this case, we will do it once the porjection matrix has been instantiated\).

```java
shaderProgram.createUniform("projectionMatrix");
```

At this moment, we already have a holder ready to be set up with data to be used as our projection matrix. Since the projection matrix won’t change between rendering calls we may set up the values right after the creation of the uniform, but we will do it in our render method. You will see later that we may reuse that uniform to do additional operations that need to be done en each render call.

We will create another method in our `ShaderProgram` class to setup the data named `setUniform`. Basically we transform our matrix into a 4x4 `FloatBuffer` by using the utility methods provided by JOML library and send them to the location we stored  in our locations map.

```java
public void setUniform(String uniformName, Matrix4f value) {
    // Dump the matrix into a float buffer
    try (MemoryStack stack = MemoryStack.stackPush()) {
        FloatBuffer fb = stack.mallocFloat(16);
        value.get(fb);
        glUniformMatrix4fv(uniforms.get(uniformName), false, fb);
    }
}
```

As you can see we are creating buffers in a different way here. We are using auto managed buffers, and allocating them in the stack. This due to the fact that the size of this buffer is small and that it will not be used beyond this method. Thus, we use the `MemoryStack`class.

Now we can use that method in the `Renderer` class in the `render` method, after the shader program has been binded:

```java
shaderProgram.setUniform("projectionMatrix", projectionMatrix);
```

We are almost over, we can now show the quad correctly rendered, so you can now launch your program and will obtain a.... black background without any coloured quad. What’s happening? Did we break something? Well, actually no, remember that we are now simulating the effect of camera looking at our scene, and that we provided to distances, one to the farthest plane \(equal to 1.000f\) and one to the closest plane \(equal to 0.01f\). Our coordinates are:

```java
float[] positions = new float[]{
    -0.5f,  0.5f, 0.0f,
    -0.5f, -0.5f, 0.0f,
     0.5f, -0.5f, 0.0f,
     0.5f,  0.5f, 0.0f,
};
```

That is, our z coordinates are outside the visible zone. Let’s assign them a value of `-0.05f`. Now you will see a giant green square like this:

![Square 1](square_1.png)

What is happening now is that we are drawing the quad to close to our camera, we are actually zooming into it. If we assign now a value of `-1.05f` to the z coordinate we can see now our coloured quad.

![Square coloured](square_coloured.png)

If we continue pushing backwards the quad we will see it smaller. Notice also that our quad does not resemble to a rectangle anymore.

## Applying Transformations

Let’s recall what we’ve done so far. We have learned how to pass data in an efficient format to our graphic card. How to project that data and assign them colours using vertex and fragments shaders. Now we should start drawing more complex models in our 3D space. But in order to do that we must be able to load an arbitrary model and represent it in our 3D space in a specific position,  with the appropriate size and the required rotation.

So right now, in order to that representation we need to provide some basic operations to act upon any model:

* Translation: Move an object by some amount in any of the three axis.
* Rotation: Rotate an object by some amount of degrees over any of the three axis.
* Scale: Adjust the size of an object.

![Transformations](transformations.png)

The operations described above are known as transformations. And you probable may be guessing that the way we are we going to achieve that is by multiplying our coordinates by a set of matrices \(one for translation, one for rotation and one for scaling\). Those three matrices will be combined into a single matrix called world matrix and passed as a uniform to our vertex shader.

The reason why it is called world matrix is because we are transforming from model coordinates to world coordinates. When you will learn about loading 3D models you will see that those models are defined using it’s own coordinate systems, they don’t know the size of your 3D space and they need to be placed in it so when we multiply our coordinates by our matrix what we are doing is transforming from a coordinate systems \(the model one\) to another coordinate systems \(the one for our 3D world\).

That world matrix will be calculated like this \(The order is important since multiplication using matrices is not commutative\):


$$
World Matrix\left[Translation Matrix\right]\left[Rotation Matrix\right]\left[Scale Matrix\right]
$$


If we include our projection matrix to the transformation matrix it would be like this:


$$
Transf=\left[Proj Matrix\right]\left[Translation Matrix\right]\left[Rotation Matrix\right]\left[Scale Matrix\right]=\left[Proj Matrix\right]\left[World Matrix\right]
$$


The translation matrix  is defined like this:


$$
\begin{bmatrix}
1 & 0 & 0 & dx \\
0 & 1 & 0 & dy \\
0 & 0 & 1 & dz \\
0 & 0 & 0 & 1
\end{bmatrix}
$$


Translation Matrix Parameters:

* dx: Displacement along the x axis.
* dy: Displacement along the y axis.
* dz: Displacement along the z axis.

The scale matrix is defined like this:


$$
\begin{bmatrix}
sx & 0 & 0 & 0 \\
0 & sy & 0 & 0 \\
0 & 0 & sz & 0 \\
0 & 0 & 0 & 1
\end{bmatrix}
$$


Scale Matrix Parameters:

* sx: Scaling along the x axis.
* sy: Scaling along the y axis.
* sz: Scaling along the z axis.

The rotation matrix is much more complex, but keep in mind that it can be constructed by the multiplication of 3 rotation matrices for a single axis.

Now, in order to apply those concepts we need to refactor our code a little bit. In our game we will be loading a set of models which can be used to render many objects in different positions according to our game logic \(imagine a FPS game which loads three models for different enemies, there are only three models but using these models we can draw as many enemies as we want\). Do we need to create a VAO and the set of VBOs for each of those objects ? The answer is no, we only need to load it once per model. What we need to do is draw it independently according to its position, size and rotation. We need to transform those models when we are rendering it.

So we will create a new class named `GameItem` that will hold a reference to a model, to a `Mesh` instance. A `GameItem` instance will have variables for storing its position, its rotation state and its scale. This is the definition of that class.

```java
package org.lwjglb.engine;

import org.joml.Vector3f;
import org.lwjglb.engine.graph.Mesh;

public class GameItem {

    private final Mesh mesh;

    private final Vector3f position;

    private float scale;

    private final Vector3f rotation;

    public GameItem(Mesh mesh) {
        this.mesh = mesh;
        position = new Vector3f(0, 0, 0);
        scale = 1;
        rotation = new Vector3f(0, 0, 0);
    }

    public Vector3f getPosition() {
        return position;
    }

    public void setPosition(float x, float y, float z) {
        this.position.x = x;
        this.position.y = y;
        this.position.z = z;
    }

    public float getScale() {
        return scale;
    }

    public void setScale(float scale) {
        this.scale = scale;
    }

    public Vector3f getRotation() {
        return rotation;
    }

    public void setRotation(float x, float y, float z) {
        this.rotation.x = x;
        this.rotation.y = y;
        this.rotation.z = z;
    }

    public Mesh getMesh() {
        return mesh;
    }
}
```

We will create another class which will deal with transformations named `Transformation`.

```java
package org.lwjglb.engine.graph;

import org.joml.Matrix4f;
import org.joml.Vector3f;

public class Transformation {

    private final Matrix4f projectionMatrix;

    private final Matrix4f worldMatrix;

    public Transformation() {
        worldMatrix = new Matrix4f();
        projectionMatrix = new Matrix4f();
    }

    public final Matrix4f getProjectionMatrix(float fov, float width, float height, float zNear, float zFar) {
        float aspectRatio = width / height;        
        projectionMatrix.identity();
        projectionMatrix.perspective(fov, aspectRatio, zNear, zFar);
        return projectionMatrix;
    }

    public Matrix4f getWorldMatrix(Vector3f offset, Vector3f rotation, float scale) {
        worldMatrix.identity().translate(offset).
                rotateX((float)Math.toRadians(rotation.x)).
                rotateY((float)Math.toRadians(rotation.y)).
                rotateZ((float)Math.toRadians(rotation.z)).
                scale(scale);
        return worldMatrix;
    }
}
```

As you can see this class groups the projection and world matrix. Given a set of vectors that model the displacement, rotation and scale it returns the world matrix. The method `getWorldMatrix` returns the matrix that will be used to transform the coordinates for each `GameItem` instance. That class also provides a method that, based on the Field Of View, the aspect ratio and the near and far distance gets the projection matrix.

An important thing notice is that the the `mul` method of the  `Matrix4f` class modifies the matrix instance that applies to. So if we directly multiply the projection matrix with the transformation matrix we will be modifying the projection matrix itself. This is why we are always initializing each matrix to the identity matrix upon each call.

In the `Renderer` class, in the constructor method, we just instantiate the `Transformation` with no arguments and in the `init` method we just create the uniform. The uniform name has been renamed to transformation to better match its purpose.

```java
public Renderer() {
    transformation = new Transformation();
}

public void init(Window window) throws Exception {
    // .... Some code before ...
    // Create uniforms for world and projection matrices
    shaderProgram.createUniform("projectionMatrix");
    shaderProgram.createUniform("worldMatrix");

    window.setClearColor(0.0f, 0.0f, 0.0f, 0.0f);
}
```

In the render method of our `Renderer` class we receive now an array of GameItems:

```java
public void render(Window window, GameItem[] gameItems) {
    clear();

        if ( window.isResized() ) {
            glViewport(0, 0, window.getWidth(), window.getHeight());
            window.setResized(false);
        }

    shaderProgram.bind();

    // Update projection Matrix
    Matrix4f projectionMatrix = transformation.getProjectionMatrix(FOV, window.getWidth(), window.getHeight(), Z_NEAR, Z_FAR);
    shaderProgram.setUniform("projectionMatrix", projectionMatrix);        

    // Render each gameItem
    for(GameItem gameItem : gameItems) {
        // Set world matrix for this item
        Matrix4f worldMatrix =
            transformation.getWorldMatrix(
                gameItem.getPosition(),
                gameItem.getRotation(),
                gameItem.getScale());
        shaderProgram.setUniform("worldMatrix", tMatrix);
        // Render the mes for this game item
        gameItem.getMesh().render();
    }

    shaderProgram.unbind();
}
```

We update the projection matrix once per `render` call. By doing this way we can deal with window resize operations. Then we iterate over the `GameItem` array and create a transformation matrix according to the position, rotation and scale of each of them. This matrix is pushed to the shader and the Mesh is drawn. The projection matrix is the same for all the items to be rendered, this is the reason why it it’s a separate variable in our Transformation class.

We have moved the rendering code to draw a Mesh to this class:

```java
public void render() {
    // Draw the mesh
    glBindVertexArray(getVaoId());
    glEnableVertexAttribArray(0);
    glEnableVertexAttribArray(1);

    glDrawElements(GL_TRIANGLES, getVertexCount(), GL_UNSIGNED_INT, 0);

    // Restore state
    glDisableVertexAttribArray(0);
    glDisableVertexAttribArray(1);
    glBindVertexArray(0);
}
```

Our vertex shader simply adds a new the called `worldMatrix`  and uses it with the `projectionMatrix` to calculate the position:

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec3 inColour;

out vec3 exColour;

uniform mat4 worldMatrix;
uniform mat4 projectionMatrix;


void main()
{
    gl_Position = projectionMatrix * worldMatrix * vec4(position, 1.0);
    exColour = inColour;
}
```

As you can see the code is exactly the same, we are using the uniform to correctly project our coordinates taking into consideration our frustrum, position, scale and rotation information.

Another important thing to think about is, why don’t we pass the translation, rotation and scale matrices instead of  combining them into a world matrix ? The reason is that we should try to limit the matrices we use in our shaders. Also keep in mind that the matrix multiplication that we do in our shader is done once per each vertex. The projection matrix does not change between render calls and the world matrix does not change per `GameItem` instance. If we passed the translation, rotation and scale matrices independently we will be doing many more matrices multiplication. Think about a model with tons of vertices and that’s a lot of extra operations.

But you may now think, that if the world matrix does not change per `GameItem` instance, why we don’t do the matrix multiplication in our Java class. We would be by multiplying the projection matrix and the world matrix just once per GameItem and we would send it as single uniform. In this case we would be saving many more operations. The answer is that this is a valid point right now but when we add more features to our game engine we will need to operate with world coordinates in the shaders anyway, so it’s better to handlen in and independet way those two matrices.

Finally we only need to change the `DummyGame` class to create a instance of `GameItem` with its associated `Mesh` and add some logic to translate, rotate and scale our quad. Since it’s only a test example and does not add too much you can find it in the source code that accompanies this book.

