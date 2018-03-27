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

So let’s start coding. The first task that we will be doing is create a new class for the G-Buffer. The class, named GBuffer, is defined like this:



