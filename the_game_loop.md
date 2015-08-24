
# The Game Loop 



In this chapter we will start developing our game engine by creating our game loop. The game loop is the core component of every game, it is basically and endless loop which is responsible of periodically  handling user input, updating game state and render to the screen.

The following snippet shows the structure of a game loop:

```
while (keepOnRunning) {
    handleInput();
    updateGameState();
    render();
}
```

So, is that all? Have we finished with game loops? Well, not yet. The above snippet has many pitfalls. First of all the speed that the game loop runs will execute at different speeds depending on the machine it runs on. If the machine is fast enough the user will not even be able to see what is happening in the game. Moreover, that game loop will consume all the machine resources.

Thus, we need the game loop to try run at a constant rate independently of the machine it runs on. Let us suppose that we want our game to run at a constant rate of 50 Frames Per Second (FPS). Our game loop could be something like this: 

```
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

```
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

```
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
