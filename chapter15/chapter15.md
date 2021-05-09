# Terrain Collisions

Once we have created a terrain, the next step is to detect collisions to avoid traversing through it. If you recall from previous chapter, a terrain is composed by blocks, and each of those blocks is constructed from a height map. The height map is used to set the height of the vertices of the triangles that form the terrain.

In order to detect a collision we must compare current position $$y$$ value with the $$y$$ value of the point of the terrain we are currently in. If we are above terrain’s $$y$$ value there’s no collision, otherwise we need to get back. It's a simple concept, isn't it? Indeed it is, but we need to perform several calculations before we are able to do that comparison.

The first thing we need to define is what we understand for the term "current position". Since we do not have yet a player concept the answer is easy, the current position will be the camera position. So we already have one of the components of the comparison, thus, the next thing to calculate is terrain height at the current position.

As it's been said before, the terrain is composed by a grid of terrain blocks as shown in the next figure.

![](terrain_grid.png)
 
Each terrain block is constructed from the same height map mesh, but is scaled and displaced precisely to form a terrain grid that looks like a continuous landscape.

So first we need to determine which terrain block the camera is in. In order to do that, we will calculate the bounding box of each terrain block taking into consideration the displacement and the scaling. Since the terrain will not be displaced or scaled at runtime, we can perform those calculations in the ```Terrain``` class constructor. By doing so we can access them later at any time without repeating those operations in each game loop cycle.

We will create a new method that calculates the bounding box of a terrain block, named ```getBoundingBox```.

```java
private Box2D getBoundingBox(GameItem terrainBlock) {
    float scale = terrainBlock.getScale();
    Vector3f position = terrainBlock.getPosition();

    float topLeftX = HeightMapMesh.STARTX * scale + position.x;
    float topLeftZ = HeightMapMesh.STARTZ * scale + position.z;
    float width = Math.abs(HeightMapMesh.STARTX * 2) * scale;
    float height = Math.abs(HeightMapMesh.STARTZ * 2) * scale;
    Box2D boundingBox = new Box2D(topLeftX, topLeftZ, width, height);
    return boundingBox;
}
```

The ```Box2D``` class is a simplified version of the ```java.awt.Rectangle2D.Float``` class which we created to avoid using AWT.

Now we need to calculate the world coordinates of the terrain blocks. In the previous chapter you saw that all of our terrain meshes were created inside a quad with its origin set to ```[STARTX, STARTZ]```. Thus, we need to transform those coordinates to the world coordinates taking into consideration the scale and the displacement as shown in the next figure.

![Model to world coordinates](model_to_world_coordinates.png)
 
As it’s been said above, this can be done in the Terrain class constructor since it won't change at run time. So we need to add a new attribute which will hold the bounding boxes:

```java
private final Box2D[][] boundingBoxes;
```

In the ```Terrain``` constructor, while we are creating the terrain blocks, we just need to invoke the method that calculates the bounding box.

```java
public Terrain(int terrainSize, float scale, float minY, float maxY, String heightMapFile, String textureFile, int textInc) throws Exception {
    this.terrainSize = terrainSize;
    gameItems = new GameItem[terrainSize * terrainSize];

    ByteBuffer buf = null;
    int width;
    int height;
    try (MemoryStack stack = MemoryStack.stackPush()) {
        IntBuffer w = stack.mallocInt(1);
        IntBuffer h = stack.mallocInt(1);
        IntBuffer channels = stack.mallocInt(1);

        URL url = Texture.class.getResource(heightMapFile);
        File file = Paths.get(url.toURI()).toFile();
        String filePath = file.getAbsolutePath();
        buf = stbi_load(filePath, w, h, channels, 4);
        if (buf == null) {
            throw new Exception("Image file [" + filePath  + "] not loaded: " + stbi_failure_reason());
        }

        width = w.get();
        height = h.get();
    }

    // The number of vertices per column and row
    verticesPerCol = width - 1;
    verticesPerRow = height - 1;

    heightMapMesh = new HeightMapMesh(minY, maxY, buf, width, height, textureFile, textInc);
    boundingBoxes = new Box2D[terrainSize][terrainSize];
    for (int row = 0; row < terrainSize; row++) {
        for (int col = 0; col < terrainSize; col++) {
            float xDisplacement = (col - ((float) terrainSize - 1) / (float) 2) * scale * HeightMapMesh.getXLength();
            float zDisplacement = (row - ((float) terrainSize - 1) / (float) 2) * scale * HeightMapMesh.getZLength();

            GameItem terrainBlock = new GameItem(heightMapMesh.getMesh());
            terrainBlock.setScale(scale);
            terrainBlock.setPosition(xDisplacement, 0, zDisplacement);
            gameItems[row * terrainSize + col] = terrainBlock;

            boundingBoxes[row][col] = getBoundingBox(terrainBlock);
        }
    }
	
	stbi_image_free(buf);
}
```

So, with all the bounding boxes pre-calculated, we are ready to create a new method that will return the height of the terrain taking as a parameter the current position. This method will be named ```getHeightVector``` and is defined like this.

```java
public float getHeight(Vector3f position) {
    float result = Float.MIN_VALUE;
    // For each terrain block we get the bounding box, translate it to view coordinates
    // and check if the position is contained in that bounding box
    Box2D boundingBox = null;
    boolean found = false;
    GameItem terrainBlock = null;
    for (int row = 0; row < terrainSize && !found; row++) {
        for (int col = 0; col < terrainSize && !found; col++) {
            terrainBlock = gameItems[row * terrainSize + col];
            boundingBox = boundingBoxes[row][col];
            found = boundingBox.contains(position.x, position.z);
        }
    }

    // If we have found a terrain block that contains the position we need
    // to calculate the height of the terrain on that position
    if (found) {
        Vector3f[] triangle = getTriangle(position, boundingBox, terrainBlock);
        result = interpolateHeight(triangle[0], triangle[1], triangle[2], position.x, position.z);
    }

    return result;
}
```

The first thing that to we do in that method is to determine the terrain block that we are in. Since we already have the bounding box for each terrain block, the algorithm is simple. We just simply need to iterate over the array of bounding boxes and check if the current position is inside (the class 
```Box2D``` provides a method for this).

Once we have found the terrain block, we need to calculate the triangle we are in. This is done in the ```getTriangle``` method that will be described later on. After that, we have the coordinates of the triangle that we are in, including its height. But we need the height of a point that is not located at any of those vertices but in a place in between. This is done in the $$interpolateHeight$$ method. We will also explain how this is done later on.

Let’s first start with the process of determining the triangle that we are in. The quad that forms a terrain block can be seen as a grid in which each cell is formed by two triangles Let’s define some variables first:

* $$boundingBox.x$$ is the $$x$$ coordinate of the origin of the bounding box associated to the quad.
* $$boundingBox.y$$ is the $$z$$ coordinates  of the origin of the bounding box associated to the quad (Although you see a “$$y$$”, it models the $$z$$ axis).
* $$boundingBox.width$$ is the width of the quad.
* $$boundingBox.height$$ is the height of the quad.
* $$cellWidth$$ is the width  of a cell.
* $$cellHeight$$ is the height of a cell.

All of the variables defined above are expressed in world coordinates.  To calculate the width of a cell we just need to divide the bounding box width by the number of vertices per column:

$$cellWidth = \frac{boundingBox.width}{verticesPerCol}$$

And the variable ```cellHeight``` is calculated analogously:

$$cellHeight = \frac{boundingBox.height}{verticesPerRow}$$

Once we have those variables we can calculate the row and the column of the cell we are currently in, which is quite straightforward:

$$col = \frac{position.x - boundingBox.x}{boundingBox.width}$$

$$row = \frac{position.z - boundingBox.y}{boundingBox.height}$$

The following picture shows all the variables  described above for a sample terrain block.

![Terrain block variables](terrain_block_variables_n.png)

With all that information we are able to calculate the positions of the vertices of the triangles contained in the cell. How we can do this? Let’s examine the triangles that form a single cell.
 
![Cell](cell.png)

You can see that the cell is divided by a diagonal that separates the two triangles. To determine which is the triangle associated to the current position, we check if the $$z$$ coordinate is above or below that diagonal. In our case, if current position $$z$$ value is less than the $$z$$ value of the diagonal setting the $$x$$ value to the $$x$$ value of current position we are in T1. If it's greater than that we are in T2.

We can determine that by calculating the line equation that matches the diagonal.

If you remember your school math classes, the equation of a line that passes from two points (in 2D) is:

$$y-y1=m\cdot(x-x1),$$

where m is the line slope, that is, how much the height changes when moving through the $$x$$ axis. Note that, in our case, the $$y$$ coordinates are the $$z$$ ones. Also note that we are using 2D coordinates because we are not calculating heights here. We just want to select the proper triangle and to do that $$x$$ an $$z$$ coordinates are enough. So, in our case the line equation should be rewritten like this.

$$z-z1=m\cdot(x-x1)$$

The slope can be calculated in the following way:

$$m=\frac{z1-z2}{x1-x2}$$

So the equation of the diagonal to get the $$z$$ value given a $$x$$ position is like this:

$$z=m\cdot(xpos-x1)+z1=\frac{z1-z2}{x1-x2}\cdot(xpos-x1)+z1$$

Where $$x1$$, $$x2$$, $$z1$$ and $$z2$$ are the $$x$$ and $$z$$ coordinates of the vertices $$V1$$ and $$V2$$, respectively.

So the method to get the triangle that the current position is in, named ```getTriangle```, applying all the calculations described above can be implemented like this:

```java
protected Vector3f[] getTriangle(Vector3f position, Box2D boundingBox, GameItem terrainBlock) {
    // Get the column and row of the heightmap associated to the current position
    float cellWidth = boundingBox.width / (float) verticesPerCol;
    float cellHeight = boundingBox.height / (float) verticesPerRow;
    int col = (int) ((position.x - boundingBox.x) / cellWidth);
    int row = (int) ((position.z - boundingBox.y) / cellHeight);

    Vector3f[] triangle = new Vector3f[3];
    triangle[1] = new Vector3f(
        boundingBox.x + col * cellWidth,
        getWorldHeight(row + 1, col, terrainBlock),
        boundingBox.y + (row + 1) * cellHeight);
    triangle[2] = new Vector3f(
        boundingBox.x + (col + 1) * cellWidth,
        getWorldHeight(row, col + 1, terrainBlock),
        boundingBox.y + row * cellHeight);
    if (position.z < getDiagonalZCoord(triangle[1].x, triangle[1].z, triangle[2].x, triangle[2].z, position.x)) {
        triangle[0] = new Vector3f(
            boundingBox.x + col * cellWidth,
            getWorldHeight(row, col, terrainBlock),
            boundingBox.y + row * cellHeight);
    } else {
        triangle[0] = new Vector3f(
            boundingBox.x + (col + 1) * cellWidth,
            getWorldHeight(row + 2, col + 1, terrainBlock),
            boundingBox.y + (row + 1) * cellHeight);
    }

    return triangle;
}

protected float getDiagonalZCoord(float x1, float z1, float x2, float z2, float x) {
    float z = ((z1 - z2) / (x1 - x2)) * (x - x1) + z1;
    return z;
}

protected float getWorldHeight(int row, int col, GameItem gameItem) {
    float y = heightMapMesh.getHeight(row, col);
    return y * gameItem.getScale() + gameItem.getPosition().y;
}
```

You can see that we have two additional methods. The first one, named ```getDiagonalZCoord```, calculates the $$z$$ coordinate of the diagonal given a $$x$$ position and two vertices. The other one, named ```getWorldHeight```, is used to retrieve the height of the triangle vertices, the $$y$$ coordinate. When the terrain mesh is constructed the height of each vertex is pre-calculated and stored, we only need to translate it to world coordinates.

At this point we have the triangle coordinates that the current position is in. Finally, we are ready to calculate terrain height at the current position. How can we do this? Well, our triangle is contained in a plane, and a plane can be defined by three points, in this case, the three vertices that define a triangle.

The plane equation is as follows:
$$a\cdot x+b\cdot y+c\cdot z+d=0$$

The values of the constants of the previous equation are:

$$a=(B_{y}-A_{y}) \cdot (C_{z} - A_{z}) - (C_{y} - A_{y}) \cdot (B_{z}-A_{z})$$

$$b=(B_{z}-A_{z}) \cdot (C_{x} - A_{x}) - (C_{z} - A_{z}) \cdot (B_{z}-A_{z})$$

$$c=(B_{x}-A_{x}) \cdot (C_{y} - A_{y}) - (C_{x} - A_{x}) \cdot (B_{y}-A_{y})$$

Where $$A$$, $$B$$ and $$C$$ are the three vertices needed to define the plane.

Then, with previous equations and the values of the $$x$$ and $$z$$ coordinates for the current position we are able to calculate the y value, that is the height of the terrain at the current position:

$$y = (-d - a \cdot x - c \cdot z) / b$$

The method that performs the previous calculations is the following:

```java
protected float interpolateHeight(Vector3f pA, Vector3f pB, Vector3f pC, float x, float z) {
    // Plane equation ax+by+cz+d=0
    float a = (pB.y - pA.y) * (pC.z - pA.z) - (pC.y - pA.y) * (pB.z - pA.z);
    float b = (pB.z - pA.z) * (pC.x - pA.x) - (pC.z - pA.z) * (pB.x - pA.x);
    float c = (pB.x - pA.x) * (pC.y - pA.y) - (pC.x - pA.x) * (pB.y - pA.y);
    float d = -(a * pA.x + b * pA.y + c * pA.z);
    // y = (-d -ax -cz) / b
    float y = (-d - a * x - c * z) / b;
    return y;
}
```

And that’s all! We are now able to detect the collisions, so in the ```DummyGame``` class we can change the following lines when we update the camera position:

```java
// Update camera position
Vector3f prevPos = new Vector3f(camera.getPosition());
camera.movePosition(cameraInc.x * CAMERA_POS_STEP, cameraInc.y * CAMERA_POS_STEP, cameraInc.z * CAMERA_POS_STEP);        
// Check if there has been a collision. If true, set the y position to
// the maximum height
float height = terrain.getHeight(camera.getPosition());
if ( camera.getPosition().y <= height )  {
    camera.setPosition(prevPos.x, prevPos.y, prevPos.z);
}
```

As you can see, the concept of detecting terrain collisions is easy to understand, but we need to carefully perform a set of calculations and be aware of the different coordinate systems we are dealing with.

Besides that, although the algorithm presented here is valid in most of the cases, there are still situations that need to be handled carefully. One effect that you may observe is the one called tunnelling. Imagine the following situation: we are travelling at a fast speed through our terrain and because of that, the position increment gets a high value. This value can get so high that, since we are detecting collisions with the final position, we may have skipped obstacles that lay in between.

![Tunnelling](tunnelling.png)

There are many solutions to the tunnelling effect, the simplest one being to split the calculation to be performed in smaller increments.
