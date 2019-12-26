# HUD Revisited - NanoVG

In previous chapters we explained how a HUD can be created by rendering shapes and textures over the top of the scene using an orthographic projection. In this chapter we will learn how to use the [NanoVG](https://github.com/memononen/nanovg) library to be able to render antialiased vector graphics to construct more complex HUDs in an easy way.

There are many other libraries out there that you can use to accomplish this task, such as [Nifty GUI](https://github.com/nifty-gui/nifty-gui), [Nuklear](https://github.com/vurtun/nuklear), etc. In this chapter we will focus on Nanovg since it’s very simple to use, but if you’re looking for developing complex GUI interactions with buttons, menus and windows you should probably look at [Nifty GUI](https://github.com/nifty-gui/nifty-gui).

The first step in order to start using [NanoVG](https://github.com/memononen/nanovg) is adding the dependencies in the `pom.xml` file \(one for the dependencies required at compile time and the other one for the natives required at runtime\).

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

Before we start using [NanoVG](https://github.com/memononen/nanovg) we must set up some things in the OpenGL side so the samples can work correctly. We need to enable support for stencil test. Until now we have talked about colour and depth buffers, but we have not mentioned the stencil buffer. This buffer stores a value \(an integer\) for every pixel which is used to control which pixels should be drawn. This buffer is used to mask or discard drawing areas according to the values it stores. It can be used, for instance, to cut out some parts of the scene in an easy way. We enable stencil test by adding this line to the `Window` class \(after we enable depth testing\):

```java
glEnable(GL_STENCIL_TEST);
```

Since we are using another buffer we must take care also of removing its values before each render call. Thus, we need to modify the clear method of the `Renderer` class:

```java
public void clear() {
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT);
}
```

We will also add a new window option for activating antialiasing. Thus, in the `Window` class we will enable it by this way:

```java
if (opts.antialiasing) {
    glfwWindowHint(GLFW_SAMPLES, 4);
}
```

Now we are ready to use the [NanoVG](https://github.com/memononen/nanovg) library. The first thing that we will do is get rid off the HUD artifacts we have created, that is the shaders, the `IHud` interface, the hud rendering methods in the `Renderer` class, etc. Yo can check this out in the source code.

In this case, the new `Hud` class will take care of its rendering, so we do not need to delegate it to the `Renderer` class. Let’s start by defining that class, It will have an `init` method that sets up the library and the resources needed to build the HUD. The method is defined like this:

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

    posx = MemoryUtil.memAllocDouble(1);
    posy = MemoryUtil.memAllocDouble(1);

    counter = 0;
}
```

The first thing we do is create a NanoVG context. In this case we are using an OpenGL 3.0 backend since we are referring to the `org.lwjgl.nanovg.NanoVGGL3` namespace. If antialiasing is activated we set up the flag `NVG_ANTIALIAS`.

Next, we create a font by using a True Type font previously loaded into a `ByteBuffer`. We assign it a name so we can later on use it while rendering text. One important thing about this is that the `ByteBuffer` used to load the font must be kept in memory while the font is used. That is, it cannot be garbage collected, otherwise you will get a nice core dump. This is why it is stored as a class attribute.

Then, we create a colour instance and some helpful variables that will be used while rendering. That  method is called in the game init method, just before the rendered is initialized:

```java
@Override
public void init(Window window) throws Exception {
    hud.init(window);
    renderer.init(window);
    ...
```

The `Hud` class also defines a render method, which should be called after the scene has been rendered so the HUD is drawn on top of it.

```java
@Override
public void render(Window window) {
    renderer.render(window, camera, scene);
    hud.render(window);
}
```

The `render` method of the Hud class starts like this:

```java
public void render(Window window) {
    nvgBeginFrame(vg, window.getWidth(), window.getHeight(), 1);
```

The first thing that we must do is call the `nvgBeginFrame` method. All the NanoVG rendering operations must be enclosed between a `nvgBeginFrame` and `nvgEndFrame` calls. The `nvgBeginFrame` accepts the following parameters:

* The NanoVG context.
* The size of the window to render \(width and height\).
* The pixel ratio. If you need to support Hi-DPI, you can change this value. For this sample we just set it to 1.

Then we create several ribbons that occupy the whole screen with. The first one is drawn like this:

```java
// Upper ribbon
nvgBeginPath(vg);
nvgRect(vg, 0, window.getHeight() - 100, window.getWidth(), 50);
nvgFillColor(vg, rgba(0x23, 0xa1, 0xf1, 200, colour));
nvgFill(vg);
```

While rendering a shape, the first method that will be invoked is `nvgBeginPath`, which instructs NanoVG to start drawing a new shape. Then we define what to draw, a rect, the fill colour and by invoking the `nvgFill` we draw it.

You can check the rest of the source code to see how the rest of the shapes are drawn. When rendering text is not necessary to call `nvgBeginPath` before rendering it.

After we have finished drawing all the shapes, we just call the `nvgEndFrame` to end rendering, but there’s one important thing to be done before leaving the method: we must restore the OpenGL state. NanoVG modifies the OpenGL state in order to perform their operations; if the state is not correctly restored, you may see that the scene is not correctly rendered or even that it's been wiped out. Thus, we need to restore the relevant OpenGL status that we need for our rendering. This is delegated in the `Window` class:

```java
// Restore state
window.restoreState();
```

The method is defined like this:

```java
public void restoreState() {
    glEnable(GL_DEPTH_TEST);
    glEnable(GL_STENCIL_TEST);
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
    if (opts.cullFace) {
        glEnable(GL_CULL_FACE);
        glCullFace(GL_BACK);
    }
}
```

And that’s all \(besides some additional methods to clear things up\), the code is completed. When you execute the sample you will get something like this:

![Hud](hud.png)

