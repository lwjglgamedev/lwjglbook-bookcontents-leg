# Hud revisited - NanoVG

In previous chapters we explained how a HUD can be created renderings shapes and textures over the top of the scene using an orthographic projection.  In this chapter we will learn how to use the [NanoVG](https://github.com/memononen/nanovg) library to be able to render antialiased vector graphics to construct more complex HUDs in an easy way.

There are many other libraries out there that you can use to accomplish this task, such as [Nifty GUI](https://github.com/nifty-gui/nifty-gui), [Nuklear](https://github.com/vurtun/nuklear), etc. In this chapter we will focus on Nanovg since it’s very simple to use, but if you’re looking for developing complex GUI interactions with buttons, menus and windows you should probably look for [Nifty GUI](https://github.com/nifty-gui/nifty-gui).

The first step in order to start using [NanoVG](https://github.com/memononen/nanovg) is adding the dependences in the ```pom.xml``` file (one  for the dependencies required at compile time and the other one for the natives required at runtime).

```xml
...
<dependency>
	<groupId>org.lwjgl</groupId>
	<artifactId>lwjgl-nanovg</artifactId>
	<version>${lwjgl.version}</version>
</dependency>
...
<dependency>
	<groupId>org.lwjgl</groupId>
	<artifactId>lwjgl-nanovg</artifactId>
	<version>${lwjgl.version}</version>
	<classifier>${native.target}</classifier>
	<scope>runtime</scope>
</dependency>
```

Before we start using [NanoVG](https://github.com/memononen/nanovg) we must set up some things in the OpenGL side so the samples can work correctly. We need to enable support for stencil buffer test. Until now we have talked about colour and depth buffers, but we have not mentioned the stencil buffer. This buffer stores a value (an integer) for every pixel which is used to control which pixels should be drawn. This buffer is used to mask or discard drawing areas according to the values it stores. It can be used, for instance, to cut out some parts of the scene in an easy way. We enable stencil buffer test by adding this line to the ```Window``` class (after we enable depth testing):

```java
glEnable(GL_STENCIL_TEST);
```

Since we are using another buffer we must take care also of removing its values before each render call. Thus, we need to modify the clear method of the ```Renderer``` class:

```java
public void clear() {
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT);
}
```

We will also add a new window option for activating antialiasing. Thus, in the Window class we will enable it by this way:

```java
if (opts.antialiasing) {
    glfwWindowHint(GLFW_SAMPLES, 4);
}
```

Now we are ready to use the [NanoVG](https://github.com/memononen/nanovg) library. The firs thing that we will do is get rid off the HUD artefacts we have created, that is the shaders the ```IHud``` interface, the hud rendering methods in the ```Renderer``` class, etc. Yo can check this out in the source code. 

In this case, the new ```Hud``` class will take care of its rendering, so we do not need to delegate it to the ```Renderer``` class. Let’s tart by defining that class, It will have an ```init``` method that sets up the library and the resources needed to build the HUD. The method is defined like this:

```java
public void init(Window window) throws Exception {
    this.vg = window.getOptions().antialiasing ? nvgCreate(NVG_ANTIALIAS | NVG_STENCIL_STROKES) : nvgCreate(NVG_STENCIL_STROKES);
    if (this.vg == NULL) {
        throw new Exception("Could not init nanovg");
    }

    fontBuffer = Utils.ioResourceToByteBuffer("/fonts/OpenSans-Bold.ttf", 150 * 1024);
    int font = nvgCreateFontMem(vg, FONT_NAME, fontBuffer, 0);
    if (font == -1) {
        throw new Exception("Could not add font");
    }
    colour = NVGColor.create();

    posx = BufferUtils.createDoubleBuffer(1);
    posy = BufferUtils.createDoubleBuffer(1);

    counter = 0;
}
```
