# 3D Object Picking

One of the key aspects of every game is the ability to interact with the environment. This capability requires to be able to select objects in the 3D scene. In this chapter we will explore how this can be achieved.

But, before we start talking about the steps to be performed to select objects, we need a way to represent selected objects. Thus, the first thing that we must do, is add another attribute to the GameItem class, which will allow us to tag selected objects:

```private boolean selected;```

Then, we need to be able to use that value in the scene shaders. Let’s start with the fragment shader (```scene_fragment.fs```). In this case, we will assume that we will receive a flag, from the vertex shader, that will determine if the fragment to be rendered belongs to a selected object or not.

```in float outSelected;``` 

Then, at the end of the fragment shader, we will modify the final fragment colour, byt setting the Blue component to 1 if it’s selected.

```glsl
if ( outSelected > 0 ) {
    fragColor = vec4(fragColor.x, fragColor.y, 1, 1);
}
```
