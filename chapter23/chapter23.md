# 3D Object Picking

One of the key aspects of every game is the ability to interact with the environment. This capability requires to be able to select objects in the 3D scene. In this chapter we will explore how this can be achieved.

But, before we start talking about the steps to be performed to select objects, we need a way to represent selected objects. Thus, the first thing that we must do, is add another attribute to the GameItem class, which will allow us to tag selected objects:

```private boolean selected;```

Then, we need to be able to use that value in the scene shaders. Let’s start with the fragment shader (```scene_fragment.fs```). In this case, we will assume that we will receive a flag, from the vertex shader, that will determine if the fragment to be rendered belongs to a selected object or not.

```in float outSelected;``` 

Then, at the end of the fragment shader, we will modify the final fragment colour, byt setting the blue component to $$1$$ if it’s selected.

```glsl
if ( outSelected > 0 ) {
    fragColor = vec4(fragColor.x, fragColor.y, 1, 1);
}
```

Then, we need to be able to set that value for each . If you recall from previous chapters we have two scenarios:

* Rendering of non instanced meshes.
* Rendering of instanced meshes.

In the first case, the data for each ```GameItem``` is passed through uniforms, so we just need to add a new uniform for that in the vertex shader. In the second case, we need to create a new instanced attribute. You can see bellow the additions that need to be integrated into the vertex shader for both cases.

```glsl
layout (location=14) in float selectedInstanced;
...
uniform float selectedNonInstanced;
...
    if ( isInstanced > 0 )
    {
        outSelected = selectedInstanced;
...
    }
    else
    {
    outSelected = selectedNonInstanced;
...
```

Now that the infrastructure has been set-up we just need to define how objects will be selected. Before we continue you may notice, if you look at the source code, that the View matrix is now stored in the ```Camera``` class. This is due to the fact that we werrecalculating the view matrix sin several classes in the source code. Previously, it was stored in the ```Transformation``` and in the ```SoundManager``` classes. In order to calculate intersections we would need to cerate another replica. Instead of that, we centralize that in the ```Camera``` class. This change also, requires that the view matrix is updated in our main game loop.

Let’s continue with the picking discussion. In this sample, we will follow a simple approach, selection will be done automatically using the camera. The closes object to where the camera is facing will be selected. Let’s discuss how this can be done.

The following picture depicts the situation we need to solve.

![Object Picking](/chapter23/object_picking.png)

We have the camera, placed in some coordinates in world-space, facing a specific direction. Any object that intersects with a ray cast from camera’s position following camera’s forward direction will be a candidate. Between all the candidates we just need to chose the closest one.

In our sample, game items are cubes, so we need to calculate the intersection of the camera’s forward vector with cubes. It may seem to be a very specific case, but indeed is very frequent. In many games, the game items have associated what’s called a bounding box. A bounding box is a rectangle box, that contains all the vertices for that object. This bounding box is used also, for instance, for collision detection. In fact, in the animation chapter, you saw that each animation frame defined a bounding box, that helps to set the boundaries at any given time.

So, let’s start coding. We will create a new class named `` `BoxSelectionDetector```, which will have a method named ```selectGameItem``` which will receive a list of game items and a reference to the camera. The method is defined like this.

```java
public void selectGameItem(GameItem[] gameItems, Camera camera) {
    GameItem selectedGameItem = null;
    float closestDistance = Float.POSITIVE_INFINITY;

    dir = camera.getViewMatrix().positiveZ(dir).negate();
    for (GameItem gameItem : gameItems) {
        gameItem.setSelected(false);
        min.set(gameItem.getPosition());
        max.set(gameItem.getPosition());
        min.add(-gameItem.getScale(), -gameItem.getScale(), -gameItem.getScale());
        max.add(gameItem.getScale(), gameItem.getScale(), gameItem.getScale());
        if (Intersectionf.intersectRayAab(camera.getPosition(), dir, min, max, nearFar) && nearFar.x < closestDistance) {
            closestDistance = nearFar.x;
            selectedGameItem = gameItem;
        }
    }

    if (selectedGameItem != null) {
        selectedGameItem.setSelected(true);
    }
}
```

The method, iterates over the game items trying to get the ones that interesect with the ray cast form the camera. It first defines a vafiable named ```closestDistance```. This vaioabkle will hold the closest distance. For game items that intersect, the distance from the camera to the intersection point will be calculated, If it’s lower than the value stored in ```closestDistance```, then this item will be the new candidate.

Before entering into the loop, we need to get the direction vector that points where the camera is facing. This is easy, just use the view matrix to get the z direction taking into consideration camera’s rotation. Remember that positive z points out of the screen, so we need the opposite direction vector, this is why we negate it.

![Camera](/chapter23/camera.png)

In the game loop intersection calculations are done per each GameItem. But, how do we do this? This is where the glorious [JOML](https://github.com/JOML-CI/JOML "JOML") library comes to the rescue. We are using [JOML](https://github.com/JOML-CI/JOML "JOML")’s ```Intersectionf``` class, which provides several methods to calculate intersection sin 2D and 3D. Specifically, we are using the ```intersectRayAab``` method.

This method implements the algorithm that test intersection for Axis Aligned Boxes. You can check the details, as pointed out in the JOML documentation, [here](http://people.csail.mit.edu/amy/papers/box-jgt.pdf "here").

The method tests if a ray, defined by an origin and a direction, intersects a box, defines by minimum and maximum corner. This algorithm is valid, because our cubes, are aligned with the axis, if they were rotated, this method would not work. Thus, the method receives the following parameters:

* An origin: In our case, this will be our camera position.
* A direction: This is where the camera is facing, the forward vector.
* The minimum corner of the box. In our case, the cubes are centered around the GameItem position, the minimum corner will be those coordinates minus the scale. (In its original size, cubes have a length of 2 and a sacle of 1).
* The maximum corner. Self explanatory.
* A result vector. This will contain the near and far distances of the intersection points.

The method will return true if there is an intersection. If true, we check the closes distance and update it if needed, and store a reference of the candidate selected ```GameItem```. The next figure shows all the elements involved in this method.

![Intersection](/chapter23/intersection.png)

Once the loop have finished, the candidate ```GameItem``` is marked as selected.

And that’s, all. The ```selectGameItem``` will be invoked in the update method of the DummyGame class, along with the view matrix update.

```java
// Update view matrix
camera.updateViewMatrix();

// Update sound listener position;
soundMgr.updateListenerPosition(camera);

this.selectDetector.selectGameItem(gameItems, camera);
```
 
Besides that, a cross-hair has been added to the rendering process to check that everytihn is working properly. The result is shown in the next figure.

![Object Picking result](/chapter23/object_picking_result.png)

Obviously, the method presented here is far form optimal but it will give you the basis to develop more sophisticated methods by your own. Some parts of the scene could be easily discarded, like objects behind the camera, since they are not going to be intersected. Besides that, you may want to order your items according to the distance to the camera to speed up calculations. In addition to that, calculations only need to be done if the camera has moved or. rotated from previous update.
