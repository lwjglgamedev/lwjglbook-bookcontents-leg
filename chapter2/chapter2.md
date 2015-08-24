
# The Game Loop 



In this chapter we will start developing our game engine by creating our game loop. The game loop is the core component of every game, it is basically and endless loop which is responsible of periodically  handling user input, updating game state and render to the screen.

The following snippet shows the structure of a game loop:

```java
while (keepOnRunning) {
    handleInput();
    updateGameState();
    render();
}
```

So, is that all? Have we finished with game loops? Well, not yet. The above snippet has many pitfalls. First of all the speed that the game loop runs will execute at different speeds depending on the machine it runs on. If the machine is fast enough the user will not even be able to see what is happening in the game. Moreover, that game loop will consume all the machine resources.

Thus, we need the game loop to try run at a constant rate independently of the machine it runs on. Let us suppose that we want our game to run at a constant rate of 50 Frames Per Second (FPS). Our game loop could be something like this: 

```java
double secsPerFrame = 1 / 50;

while (keepOnRunning) {
    double now = getTime();
    handleInput();
    updateGameState();
    render();
    sleep(now + secsPerFrame – getTime());
}
```

This game loop is simple and could be used for some games but it also presents some problems.  First of all, it assumes that our update and render methods fit in the available time we have in order to render at a constant rate of 50 FPS (that is, *secsPerFrame* which is equals to 20 ms.).

Besides that, our computer may be prioritizing another tasks that prevent our game loop to execute for certain period of time. So, we may end up updating our game state at very variable time steps which are not suitable for game physics.

Finally, sleep accuracy may range to tenth of a second, so we are not even updating at a constant frame rate even if our update and render methods are no time. So, as you see the problem is not so simple.

In the Internet you can find tons of variants for game loops, in this book we will use a not too complex approach that can work well in many situations. So let us move on and explain the basis for our game loop. The pattern used her is usually called as Fixed Step Game Loop.

First of all we may want to control separately the period at which the game state is update and the period at which the game is rendered to the screen. Why we do this ? Well, updating our game state at a constant rate is more important, especially if we use some physics engine. On the contraire, if our rendering is not done on time it makes no sense to render old frames while processing our game loop, we have the flexibility to skip some ones.

Let us have a look at how our game loop looks like:

```java
double secsPerUpdate = 1 / 30;
double previous = getTime();
double steps = 0.0;
while (true) {
  double loopStartTime = getTime();
  double elapsed = loopStartTime - previous;
  previous = current;
  steps += elapsed;

  handleInput();

  while (steps >= secsPerUpdate) {
    updateGameState();
    steps -= secsPerUpdate;
  }

  render();
  sync(current);
}
```

With this game loop we update our game state at fixed steps, but, How do we control that we do not exhaust computer resources by rendering continuously? This is done in the sync method:

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

So What are we doing in the above method ? In summary we calculate how many seconds our game loop iteration should last (which is stored in the *loopSlot* variable) and we wait for that time taking into consideration the time we have spent in our loop. But instead of doing a single wait for the whole available time period we do small waits. This will allow other tasks to run and will avoid the sleep accuracy problems we mentioned before. Then, what we do is: 
1.	Calculate the time at which we should exit this wait method and start another iteration of our game loop (which is the variable endTi**me).
2.	Compare current time with that end time and wait just one second if we have not reached that time yet.

Now  it is time to structure our code base in order to start writing our first version of our Game Engine. First of all we will encapsulate all the GLFW Window initialization code in a class named Window allowing some basic parameterization of its characteristics (such as title and size). That Window class will also provide a method to detect key presses which will be used in our game loop:

```java
public boolean isKeyPressed(int keyCode) {
    return glfwGetKey(windowHandle, keyCode) == GLFW_PRESS;
}
```

We will also create a *Renderer* class which will do our game render logic. By now, it will just have an empty *init* method and another method to clear the screen with the configured clear color:

```java
public void init() throws Exception {        
}

public void clear() {
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT
}
```

Then we will create an interface named *IGameLogic* which will encapsulate our game logic. By doing this we will make our game engine reusable across different titles. This interface will have methods to get the input, to update the game state and to render game specific data.

```java
public interface IGameLogic {

    void init() throws Exception;

    void input(Window window);

    void update(float interval);
    
    void render(Window window);
}
```

Then we will create a class named *GameEngine* which will contain our game loop code. This class will implement the *Runnable* interface since the game loop will be run inside a separate thread:

```java
public class GameEngine implements Runnable {

    //..[Removed code]..

    private final Thread gameLoopThread;

    public GameEngine(String windowTitle, int width, int height, IMageLogoc gameLogic) throws Exception {
        gameLoopThread = new Thread(this, "GAME_LOOP_THREAD");
        window = new Window(windowTitle, width, height);
        this.gameLogic = gameLogic;
        //..[Removed code]..
    }

```

As you can see we create a new Thread which will execute the run method of our *GameEngine* class which will contain our game loop:

```java
public void start() {
    gameLoopThread.start();
}    

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
Our *GameEngine* class provides a start method which just starts our Thread so run method will be executed asynchronously. That method will perform the initialization tasks and will run the game loop until  our window is closed. It is very important to initialize GLFW code inside the thread that is going to update it later. Thus, in that *init* method our Window and *Renderer* instances are initialized.

In the source code you will see that we have created other auxiliary classes such as Timer (which will provide utility methods for calculating elapsed time) and will be used by our game loop logic.

Our *GameEngine* class just delegates the input and update methods to the *IGameLogic* instance. In the render method it delegates also to the *IGameLogic*  instance an updates the window.

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

Our starting point, our class that contains the main method will just only create a *GameEngine* instance and start it.

```java
public class Main {

    public static void main(String[] args) {
        try {
            IGameLogic gameLogic = new DummyGame();
            GameEngine gameEng = new GameEngine("GAME",
                600, 480, gameLogic);
            gameEng.start();
        } catch (Exception excp) {
            excp.printStackTrace();
            System.exit(-1);
        }
    }

}
```
At the end we only need to create or game logic class, which for this chapter will be a simpler one. It will just update the increase / decrease the clear color of the window whenever the user presses the up / down key. The render method will just clear the window with that color.

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
        if ( window.isKeyPressed(GLFW_KEY_UP) ) {
            direction = 1;
        } else if ( window.isKeyPressed(GLFW_KEY_DOWN) ) {
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
        window.setClearColor(color, color, color, 0.0f);
        renderer.clear();
    }    
}
```

The class hierarchy that we have created will help us to separate our game engine code from the code of a specific game. Although it may seem necessary at this moment we need to isolate generic tasks that every game will use from the state logic, artwork and resources of an specific game in order to reuse our game engine. In later chapters we will need to restructure this class hierarchy as our game engine gets more complex.