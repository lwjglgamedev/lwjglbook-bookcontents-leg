# Cascaded Shadow Maps

In the shadows chapter we presented the shadow map technique to be able to display shadows using directional lights when rendering a 3D scene. The solution presented there, required you to manually tweak some of the parameters in order to improve the results. In this chapter we are going to change that technique to automate all the process and to improve the r results for open spaces. In order to achieve that goal we are going to use a technique called Cascaded Shadow Maps \(CSM\).

Let’s first start by examining how we can automate the construction of the light view matrix and the orthographic projection matrix used to render the shadows. If you recall from the shadows chapter, we need to draw the scene form the light’s perspective. This implies the creation of a light view matrix, which acts like a camera for light and a projection matrix. Since light is directional, and is supposed to be located at the infinity, we chose an orthographic projection.

We want all the visible objects to fit into the light view projection matrix. Hence, we need to fit the view frustum into the light frustum. The following picture depicts what we want to achieve.

![](/chapter26/view_frustum.png)

How can we construct that? The first step is to calculate the frustum corners of the view projection matrix. We get the coordinates in world space. Then we calculate the centre of that frustum. This can be calculating by adding the coordinates for all the corners and dividing the result by the number of corners.

![](/chapter26/frustum_center.png)

With that information we can set the position of the light. That position and its direction will be used to construct the light view matrix. In order to calculate the position, we start form the centre of the view frustum obtained before. We then go back to the direction of light an amount equal to the distance between the near and far z planes of the view frustum.

![](/chapter26/light_position.png)

Once we have constructed the light view matrix, we need to setup the orthographic projection matrix. In order to calculate them we transform the frustum corners to light view space, just by multiplying them by the light view matrix we have just constructed. The dimensions of that projection matrix will be minimum and maximum x and y values. The near z plane can be set up to the same value used by our standard projection matrices and the far value will be the distance between the maximum and minimum z values of the frustum corners in light view space.

However, if you implement the algorithm described above over the shadows sample, you may be disappointed by the shadows quality.

![](/chapter26/low_quality_shadows.png)

The reason for that is that shadows resolution is limited by the texture size. We are covering now a potentially huge area, and textures we are using to store depth information have not enough resolution in order to get good results. You may think that the solution is just to increase texture resolution, but this is not sufficient to completely fix the problem. You would need huge textures for that.

There’s a smarter solution for that. The key concept is that, shadows of objects that are closer to the camera need to have a higher quality than shadows for distant objects. One approach could be to just render shadows for objects close to the camera, but this would cause shadows to appear / disappear as long as we move through the scene.

The approach that Cascaded Shadow Maps \(CSMs\) use is to divide the view frustum into several splits. Splits closer to the camera cover a smaller amount spaces whilst distant regions cover a much wider region of space. The next figure shows a view frustum divided into three splits.

![](/chapter26/cascade_splits.png)

For each of these splits, the depth map is rendered, adjusting the light view and projection matrices to cover fit to each split. Thus, the texture that stores the depth map covers a reduced area of the view frustum. And, since the split closest to the camera covers less space, the depth resolution is increased.

As it can be deduced form explanation above, We will need as many depth textures as splits, and we will also change the light view and projection matrices for each of the, Hence, the steps to be done in order to apply CSMs are:

* Divide the view frustum into n splits.

* While rendering the depth map, for each split:

  * Calculate light view and projection matrices.

  * Render the scene from light’s perspective into a separate depth map

* While rendering the scene:

  * Use the depths maps calculated above.

  * Determine the split that the fragment to be drawn belongs to.

  * Calculate shadow factor as in shadow maps.

As you can see, the main drawback of CSMs is that we need to render the scene, from light’s perspective, for each split. This is why is often only used for open spaces. Anyway, we will see how we can easily reduce that overhead.

So let’s start examining the code, but before we continue a little warning, I will not include the full source code here since it would be very tedious to red. Instead, I will present the main classes their responsibilities and the fragments that may require further explanation in order to get a good understanding. All the shading related classes have been moved to a new package called`org.lwjglb.engine.graph.shadow`.

The code that renders shadows, that is, the scene from light’s perspective has been moved to the `ShadowRenderer`class. \(That code was previously contained in the `Renderer`class\).

The class defines the following constants:

```java
public static final int NUM_CASCADES = 3;
public static final float[] CASCADE_SPLITS = new float[]{Window.Z_FAR / 20.0f, Window.Z_FAR / 10.0f, Window.Z_FAR};
```

The first one is the number of cascades or splits. The second one defines where the far z plane is located for each of these splits. As you can see they are not equally spaced. The split that is closer to the camera has the shortest distance in the z plane.

The class also stores the reference to the shader program used to render the depth map, a list with the information associated to each split, modelled by the `ShadowCascade`class, and a reference to the object that whill host the depth mapth information \(tetxures\), modelled by the `ShadowBuffer`class.

The `ShadowRenderer`class has methods for setting up the shaders and the required attributes and a render method . The render method is defined like this.

```java
public void render(Window window, Scene scene, Camera camera, Transformation transformation, Renderer renderer) {
    update(window, camera.getViewMatrix(), scene);

    // Setup view port to match the texture size
    glBindFramebuffer(GL_FRAMEBUFFER, shadowBuffer.getDepthMapFBO());
    glViewport(0, 0, ShadowBuffer.SHADOW_MAP_WIDTH, ShadowBuffer.SHADOW_MAP_HEIGHT);
    glClear(GL_DEPTH_BUFFER_BIT);

    depthShaderProgram.bind();

    // Render scene for each cascade map
    for (int i = 0; i < NUM_CASCADES; i++) {
        ShadowCascade shadowCascade = shadowCascades.get(i);

        depthShaderProgram.setUniform("orthoProjectionMatrix", shadowCascade.getOrthoProjMatrix());
        depthShaderProgram.setUniform("lightViewMatrix", shadowCascade.getLightViewMatrix());

        glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_TEXTURE_2D, shadowBuffer.getDepthMapTexture().getIds()[i], 0);
        glClear(GL_DEPTH_BUFFER_BIT);

        renderNonInstancedMeshes(scene, transformation);

        renderInstancedMeshes(scene, transformation);
    }

    // Unbind
    depthShaderProgram.unbind();
    glBindFramebuffer(GL_FRAMEBUFFER, 0);
}
```
As you can see, I similar to the previous render method for shadow maps, except that we are performing several rendering passes, one per split.  In each pass we change the light view matrix and the orthographic projection matrix with the information contained in the associated ShadowCascade instande.

Also, in each pass, we need to change the texture we are using. Each pass will render the depth information to a different texture. This information is stored in the ShadowBuffer class, and is setup to be used by the FBO with this line:

```java
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_TEXTURE_2D, shadowBuffer.getDepthMapTexture().getIds()[i], 0);
```

 


