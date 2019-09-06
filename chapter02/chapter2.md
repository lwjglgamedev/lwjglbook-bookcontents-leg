# The Game Loop

In this chapter we will start developing our game engine by creating our game loop. The game loop is the core component of every game. It is basically an endless loop which is responsible for periodically handling user input, updating game state and rendering to the screen.

The following snippet shows the structure of a game loop:

```java
while (keepOnRunning) {
    handleInput();
    updateGameState();
    render();
}
```

So, is that all? Are we finished with game loops? Well, not yet. The above snippet has many pitfalls. First of all the speed that the game loop runs at will be different depending on the machine it runs on. If the machine is fast enough the user will not even be able to see what is happening in the game. Moreover, that game loop will consume all the machine resources.

Thus, we need the game loop to try running at a constant rate independently of the machine it runs on. Let us suppose that we want our game to run at a constant rate of 50 Frames Per Second \(FPS\). Our game loop could be something like this:

```java
double secsPerFrame = 1.0d / 50.0d;

while (keepOnRunning) {
    double now = getTime();
    handleInput();
    updateGameState();
    render();
    sleep(now + secsPerFrame – getTime());
}
```

This game loop is simple and could be used for some games but it also presents some problems. First of all, it assumes that our update and render methods fit in the available time we have in order to render at a constant rate of 50 FPS \(that is, `secsPerFrame` which is equal to 20 ms.\).

Besides that, our computer may be prioritizing other tasks that prevent our game loop from executing for a certain period of time. So, we may end up updating our game state at very variable time steps which are not suitable for game physics.

Finally, sleep accuracy may range to tenth of a second, so we are not even updating at a constant frame rate even if our update and render methods take no time. So, as you see the problem is not so simple.

On the Internet you can find tons of variants for game loops. In this book we will use a not too complex approach that can work well in many situations. So let us move on and explain the basis for our game loop. The pattern used here is usually called Fixed Step Game Loop.

First of all we may want to control separately the period at which the game state is updated and the period at which the game is rendered to the screen. Why do we do this? Well, updating our game state at a constant rate is more important, especially if we use some physics engine. On the contrary, if our rendering is not done in time it makes no sense to render old frames while processing our game loop. We have the flexibility to skip some frames.

Let us have a look at how our game loop looks like:

```java
double secsPerUpdate = 1.0d / 30.0d;
double previous = getTime();
double steps = 0.0;
while (true) {
  double loopStartTime = getTime();
  double elapsed = loopStartTime - previous;
  previous = loopStartTime;
  steps += elapsed;

  handleInput();

  while (steps >= secsPerUpdate) {
    updateGameState();
    steps -= secsPerUpdate;
  }

  render();
  sync(loopStartTime);
}
```

With this game loop we update our game state at fixed steps. But how do we control that we do not exhaust the computer's resources by rendering continuously? This is done in the sync method:

```java
private void sync(double loopStartTime) {
   float loopSlot = 1f / 50;
   double endTime = loopStartTime + loopSlot; 
   while(getTime() < endTime) {
       try {
           Thread.sleep(1);
       } catch (InterruptedException ie) {}
   }
}
```

So what are we doing in the above method? In summary we calculate how many seconds our game loop iteration should last \(which is stored in the `loopSlot` variable\) and we wait for that amount of time taking into consideration the time we spent in our loop. But instead of doing a single wait for the whole available time period we do small waits. This will allow other tasks to run and will avoid the sleep accuracy problems we mentioned before. Then, what we do is:  
1.    Calculate the time at which we should exit this wait method and start another iteration of our game loop \(which is the variable `endTime`\).  
2.    Compare the current time with that end time and wait just one millisecond if we have not reached that time yet.

Now it is time to structure our code base in order to start writing our first version of our Game Engine. But before doing that we will talk about another way of controlling the rendering rate. In the code presented above, we are doing micro-sleeps in order to control how much time we need to wait. But we can choose another approach in order to limit the frame rate. We can use v-sync \(vertical synchronization\). The main purpose of v-sync is to avoid screen tearing. What is screen tearing? It’s a visual effect that is produced when we update the video memory while it’s being rendered. The result will be that part of the image will represent the previous image and the other part will represent the updated one. If we enable v-sync we won’t send an image to the GPU while it is being rendered onto the screen.

When we enable v-sync we are synchronizing to the refresh rate of the video card, which at the end will result in a constant frame rate. This is done with the following line:

```java
glfwSwapInterval(1);
```

With that line we are specifying that we must wait, at least, one screen update before drawing to the screen. In fact, we are not directly drawing to the screen. We instead store the information to a buffer and we swap it with this method:

```java
glfwSwapBuffers(windowHandle);
```

So, if we enable v-sync we achieve a constant frame rate without performing the micro-sleeps to check the available time. Besides that, the frame rate will match the refresh rate of our graphics card. That is, if it’s set to 60Hz \(60 times per second\), we will have 60 Frames Per Second. We can scale down that rate by setting a number higher than 1 in the `glfwSwapInterval` method \(if we set it to 2, we would get 30 FPS\).

Let’s get back to reorganize the source code. First of all we will encapsulate all the GLFW Window initialization code in a class named `Window` allowing some basic parameterization of its characteristics \(such as title and size\). That `Window` class will also provide a method to detect key presses which will be used in our game loop:

```java
public boolean isKeyPressed(int keyCode) {
    return glfwGetKey(windowHandle, keyCode) == GLFW_PRESS;
}
```

The `Window` class besides providing the initialization code also needs to be aware of resizing. So it needs to setup a callback that will be invoked whenever the window is resized. The callback will receive the width and height, in pixels, of the framebuffer \(the rendering area, in this sample, the display area\). If you want the width, height of the framebuffer in screen coordinates you may use the `glfwSetWindowSizeCallback` method. Screen coordinates don't necessarily correspond to pixels \(for instance, on a Mac with Retina display\). Since we are going to use that information when performing some OpenGL calls, we are interested in pixels not in screen coordinates. You can get more information in the GLFW documentation.

```java
// Setup resize callback
glfwSetFramebufferSizeCallback(windowHandle, (window, width, height) -> {
    Window.this.width = width;
    Window.this.height = height;
    Window.this.setResized(true);
});
```

We will also create a `Renderer` class which will handle our game render logic. By now, it will just have an empty `init` method and another method to clear the screen with the configured clear colour:

```java
public void init() throws Exception {
}

public void clear() {
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
}
```

Then we will create an interface named `IGameLogic` which will encapsulate our game logic. By doing this we will make our game engine reusable across different titles. This interface will have methods to get the input, to update the game state and to render game-specific data.

```java
public interface IGameLogic {

    void init() throws Exception;

    void input(Window window);

    void update(float interval);

    void render(Window window);
}
```

Then we will create a class named `GameEngine` which will contain our game loop code. This class will implement to hold the game loop:

```java
public class GameEngine implements Runnable {

    //..[Removed code]..

    public GameEngine(String windowTitle, int width, int height, boolean vSync, IGameLogic gameLogic) throws Exception {
        window = new Window(windowTitle, width, height, vSync);
        this.gameLogic = gameLogic;
        //..[Removed code]..
    }
```

The `vSync` parameter allows us to select if we want to use v-sync or not. You can see we implement the run method of our `GameEngine` class which will contain our game loop:

```java
@Override
public void run() {
    try {
        init();
        gameLoop();
    } catch (Exception excp) {
        excp.printStackTrace();
    }
}
```

Our `GameEngine` class provides a `run` method that will perform the initialization tasks and will run the game loop until our window is closed. A little bit note on threading. GLFW requires to be initialized from the main thread. Polling of events should also be done in that thread. Therefore, instead of creating a separate thread for the game loop, which is what you would see commonly in games, we will execute everything from the main thread.

In the source code you will see that we created other auxiliary classes such as Timer \(which will provide utility methods for calculating elapsed time\) and will be used by our game loop logic.

Our `GameEngine` class just delegates the input and update methods to the `IGameLogic` instance. In the render method it delegates also to the `IGameLogic` instance and updates the window.

```java
protected void input() {
    gameLogic.input(window);
}

protected void update(float interval) {
    gameLogic.update(interval);
}

protected void render() {
    gameLogic.render(window);
    window.update();
}
```

Our starting point, our class that contains the main method will just only create a `GameEngine` instance and run it.

```java
public class Main {

    public static void main(String[] args) {
        try {
            boolean vSync = true;
            IGameLogic gameLogic = new DummyGame();
            GameEngine gameEng = new GameEngine("GAME",
                600, 480, vSync, gameLogic);
            gameEng.run();
        } catch (Exception excp) {
            excp.printStackTrace();
            System.exit(-1);
        }
    }

}
```

At the end we only need to create our game logic class, which for this chapter will be a simpler one. It will just increase / decrease the clear colour of the window whenever the user presses the up / down key. The `render` method will just clear the window with that colour.

```java
public class DummyGame implements IGameLogic {

    private int direction = 0;

    private float color = 0.0f;

    private final Renderer renderer;

    public DummyGame() {
        renderer = new Renderer();
    }

    @Override
    public void init() throws Exception {
        renderer.init();
    }

    @Override
    public void input(Window window) {
        if (window.isKeyPressed(GLFW_KEY_UP)) {
            direction = 1;
        } else if (window.isKeyPressed(GLFW_KEY_DOWN)) {
            direction = -1;
        } else {
            direction = 0;
        }
    }

    @Override
    public void update(float interval) {
        color += direction * 0.01f;
        if (color > 1) {
            color = 1.0f;
        } else if ( color < 0 ) {
            color = 0.0f;
        }
    }

    @Override
    public void render(Window window) {
        if (window.isResized()) {
            glViewport(0, 0, window.getWidth(), window.getHeight());
            window.setResized(false);
        }
        window.setClearColor(color, color, color, 0.0f);
        renderer.clear();
    }    
}
```

In the `render` method we get notified when the window has been resized in order to update the viewport to locate the center of the coordinates to the center of the window.

The class hierarchy that we have created will help us to separate our game engine code from the code of a specific game. Although it may seem unnecessary at this moment, we need to isolate generic tasks that every game will use from the state logic, artwork and resources of a specific game in order to reuse our game engine. In later chapters we will need to restructure this class hierarchy as our game engine gets more complex.

## Platform Differences \(OSX\)

You will be able to run the code described above on Windows or Linux, but we still need to do some modifications for OSX. As it's stated in the GLFW documentation:

> The only OpenGL 3.x and 4.x contexts currently supported by OS X are forward-compatible, core profile contexts. The supported versions are 3.2 on 10.7 Lion and 3.3 and 4.1 on 10.9 Mavericks. In all cases, your GPU needs to support the specified OpenGL version for context creation to succeed.

So, in order to support features explained in later chapters we need to add these lines to the `Window` class before the window is created:

```java
        glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
        glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 2);
        glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
        glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
```

This will make the program use the highest OpenGL version possible between 3.2 and 4.1. If those lines are not included, a Legacy version of OpenGL is used.

