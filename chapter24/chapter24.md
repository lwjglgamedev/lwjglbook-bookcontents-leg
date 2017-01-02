# Hud revisited - NanoVG

In previous chapters we explained how a HUD can be created renderings shapes and textures over the top of the scene using an orthographic projection.  In this chapter we will learn how to use the [NanoVG](https://github.com/memononen/nanovg) library to be able to render antialiased vector graphics to construct more complex HUDs in an easy way.

There are many other libraries out there that you can use to accomplish this task, such as [Nifty GUI](https://github.com/nifty-gui/nifty-gui), [Nuklear](https://github.com/vurtun/nuklear), etc. In this chapter we will focus on Nanovg since it’s very simple to use, but if you’re looking for developing complex GUI interactions with buttons, menus and windows you should probably look for [Nifty GUI](https://github.com/nifty-gui/nifty-gui).

The first step in order to start using [NanoVG](https://github.com/memononen/nanovg)is adding the dependences in the ```pom.xml``` file (one  for the dependencies required at compile time and the other one for the natives required at runtime).

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

