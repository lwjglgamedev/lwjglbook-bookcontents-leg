
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

