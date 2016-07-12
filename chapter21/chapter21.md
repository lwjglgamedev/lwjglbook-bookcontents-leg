# Instanced Rendering

## Lots of Instances

When drawing a 3D scene is frequent to have many models represented by the same mesh but with different transformations. In this case, even they may be simple objects with just a few triangles, performance can suffer. The cause behind this is the way we are rendering them.

We are basically iterating through a loop and performing a call to the function glDrawElements. As it has been said in previous chapters, calls to OpenGL library should be minimized. Each call to the glDrawElements function imposes an overhead that is reperetad again and again for each GameItem instance.

When dealing with lots of similar objects it would be more efficient to render all of them using a single call. This technicque is called instanced rendering which allows us to do that, OpenGL provides functions named dlDrawXXXInstanced to render a set of elements at once. They can be arrays or elements. In our case, since we are drawing elements we will use the function named glDrawElementsInstanced. This function receives the same arguments as the glDrawElements plus one additional parameter which sets the number of instances to be drawn.

This is a sample of how the glDrawElements is used.

`glDrawElements(GL_TRIANGLES, numVertices, GL_UNSIGNED_INT, 0); `

And this is how the instanced version can be used:

`glDrawElementsInstanced(GL_TRIANGLES, numVertices, GL_UNSIGNED_INT, 0, numInstances);`



But you may be wondering now how can you set the different transformations for each of those instances. Now, before we draw each instance we pass the different transformations and instance related data using uniforms. Before a render call is made we need to setup the specific data for each data. How can we do this when rendering all of them at once ?

When using instanced rendering, in the vertex shader we can use an input variable that holds the index of the instance that is currently being drawn. With that built-in variable we can, for instance, pass an array of uniforms containing the transformations to be applied to each instance and use a single render call.

The problem with this approach is that it’s still imposes too much overhead. Besides that, the number of uniforms that we can pass is limited. Instead of using lists of uniforms we will use instanced arrays.

If you recall from the first chapters, the data for each Mesh is defined by a set of arrays of data named VBOs. The data store in those VBOs is unique per Mesh instance.

