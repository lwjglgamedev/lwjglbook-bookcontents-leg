# Particles

## The basics

In this chapter we will add particle effects to the game engine. With this effect we will be able to simulate rays, fire, dust and clouds.  It’s a simple to implement effect that will improve the graphical aspect of any game.

There are many ways to implement particle effects with different, results. In this case we will use billboard particles. This technique uses moving texture quads to represent a particle with the peculiarity that they are always always facing the observer, in our case, the camera. You can also use billboarding technique to show information panels over game items like a mini HUDs.

Let’s start by defining what is a particle. A particle can de defined by the following attributes:
1. A mesh that represents the quad vertices.
2. A texture.
3. A position at a given instant.
4. A scale factor.
5. Speed.
6. A movement direction.
7. A life time or time to live. Once this time has expired the particle ceases to exist. 

The first four items are part of the ```GameItem``` class, but the last three are not. Thus, we will create a new class named Particle that extends a ```GameItem``` instance and that is defined like this.

```java
package org.lwjglb.engine.graph.particles;

import org.joml.Vector3f;
import org.lwjglb.engine.graph.Mesh;
import org.lwjglb.engine.items.GameItem;

public class Particle extends GameItem {

    private Vector3f speed;

    /**
     * Time to live for particle in milliseconds.
     */
    private long ttl;

    public Particle(Mesh mesh, Vector3f speed, long ttl) {
        super(mesh);
        this.speed = new Vector3f(speed);
        this.ttl = ttl;
    }

    public Particle(Particle baseParticle) {
        super(baseParticle.getMesh());
        Vector3f aux = baseParticle.getPosition();
        setPosition(aux.x, aux.y, aux.z);
        aux = baseParticle.getRotation();
        setRotation(aux.x, aux.y, aux.z);
        setScale(baseParticle.getScale());
        this.speed = new Vector3f(baseParticle.speed);
        this.ttl = baseParticle.geTtl();
    }

    public Vector3f getSpeed() {
        return speed;
    }

    public void setSpeed(Vector3f speed) {
        this.speed = speed;
    }

    public long geTtl() {
        return ttl;
    }

    public void setTtl(long ttl) {
        this.ttl = ttl;
    }
    
    /**
     * Updates the Particle's TTL
     * @param elapsedTime Elapsed Time in milliseconds
     * @return The Particle's TTL
     */
    public long updateTtl(long elapsedTime) {
        this.ttl -= elapsedTime;
        return this.ttl;
    }
}
```

As you can particle speed and movement direction can be expressed as a vector. The direction of that vector models the movement direction and its module the speed. The Particle Time To Live (TTL) is modelled as milliseconds counter that will be decreased whenever the game state is updated. The class has also a copy constructor, that is, a constructor that takes an instance of another Particle to make a copy.

Now, we need to create a particle generator or particle emitter, that is, a class that generates the particles dynamically, controls their life cycle and updates their position according to a specific model.  We can create many implementations that vary in how particles and created and how their positions are updated (for instance, taking in consideration the gravity or not). So, in order to keep our game engine generic, we will create an interface that all the Particle emitters must implement. This interface, named ```IParticleEmitter```, is defined like this:

```java
package org.lwjglb.engine.graph.particles;

import java.util.List;
import org.lwjglb.engine.items.GameItem;

public interface IParticleEmitter {

    void cleanup();
    
    Particle getBaseParticle();
    
    List<GameItem> getParticles();
}
```

The ```IParticleEmitter``` interface has a method to clean up resources, named ```cleanup```, and a method to get the list of Particles, named ```getParticles```. It also as a method named ```getBaseParticle```, but What’s this method for? A particle emitter will create many particles dynamically. Whenever a particle expires, new ones will be created. That particle renewal will use a base particle, like a pattern, to create new instances. This is what this base particle is used for, This is also the reason why the ```Particle``` class defines a copy constructor.

In the game engine code we will refer only to the ```IParticleEmitter``` interface so the base code will not be dependent on the specific implementations. Nevertheless we can create a implementation that simulates a flow of particles that are not affected by gravity. This implementation can be used to simulate rays or fire and is named ```FlowParticleEmitter```.

The behaviour of this class can be tuned with the following attributes:
* A maximum number of particles that can be alive at a time.
* A minimum period to create particles. Particles will be created one be one with a minimum period to avoid creating particles in bursts.
* A range to randomize particles speed and starting position. New particles will use  base particle position and speed with can be randomized with values between those ranges to spread the beam.

The implementation of this class is as follows:

```java
package org.lwjglb.engine.graph.particles;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import org.joml.Vector3f;
import org.lwjglb.engine.items.GameItem;

public class FlowParticleEmitter implements IParticleEmitter {

    private int maxParticles;

    private boolean active;

    private final List<GameItem> particles;

    private final Particle baseParticle;

    private long creationPeriodMillis;

    private long lastCreationTime;

    private float speedRndRange;

    private float positionRndRange;

    private float scaleRndRange;

    public FlowParticleEmitter(Particle baseParticle, int maxParticles, long creationPeriodMillis) {
        particles = new ArrayList<>();
        this.baseParticle = baseParticle;
        this.maxParticles = maxParticles;
        this.active = false;
        this.lastCreationTime = 0;
        this.creationPeriodMillis = creationPeriodMillis;
    }

    @Override
    public Particle getBaseParticle() {
        return baseParticle;
    }

    public long getCreationPeriodMillis() {
        return creationPeriodMillis;
    }

    public int getMaxParticles() {
        return maxParticles;
    }

    @Override
    public List<GameItem> getParticles() {
        return particles;
    }

    public float getPositionRndRange() {
        return positionRndRange;
    }

    public float getScaleRndRange() {
        return scaleRndRange;
    }

    public float getSpeedRndRange() {
        return speedRndRange;
    }

    public void setCreationPeriodMillis(long creationPeriodMillis) {
        this.creationPeriodMillis = creationPeriodMillis;
    }

    public void setMaxParticles(int maxParticles) {
        this.maxParticles = maxParticles;
    }

    public void setPositionRndRange(float positionRndRange) {
        this.positionRndRange = positionRndRange;
    }

    public void setScaleRndRange(float scaleRndRange) {
        this.scaleRndRange = scaleRndRange;
    }

    public boolean isActive() {
        return active;
    }

    public void setActive(boolean active) {
        this.active = active;
    }

    public void setSpeedRndRange(float speedRndRange) {
        this.speedRndRange = speedRndRange;
    }

    public void update(long ellapsedTime) {
        long now = System.currentTimeMillis();
        if (lastCreationTime == 0) {
            lastCreationTime = now;
        }
        Iterator<? extends GameItem> it = particles.iterator();
        while (it.hasNext()) {
            Particle particle = (Particle) it.next();
            if (particle.updateTtl(ellapsedTime) < 0) {
                it.remove();
            } else {
                updatePosition(particle, ellapsedTime);
            }
        }

        int length = this.getParticles().size();
        if (now - lastCreationTime >= this.creationPeriodMillis && length < maxParticles) {
            createParticle();
            this.lastCreationTime = now;
        }
    }

    private void createParticle() {
        Particle particle = new Particle(this.getBaseParticle());
        // Add a little bit of randomness of the parrticle
        float sign = Math.random() > 0.5d ? -1.0f : 1.0f;
        float speedInc = sign * (float)Math.random() * this.speedRndRange;
        float posInc = sign * (float)Math.random() * this.positionRndRange;        
        float scaleInc = sign * (float)Math.random() * this.scaleRndRange;        
        particle.getPosition().add(posInc, posInc, posInc);
        particle.getSpeed().add(speedInc, speedInc, speedInc);
        particle.setScale(particle.getScale() + scaleInc);
        particles.add(particle);
    }

    /**
     * Updates a particle position
     * @param particle The particle to update
     * @param elapsedTime Elapsed time in milliseconds
     */
    public void updatePosition(Particle particle, long elapsedTime) {
        Vector3f speed = particle.getSpeed();
        float delta = elapsedTime / 1000.0f;
        float dx = speed.x * delta;
        float dy = speed.y * delta;
        float dz = speed.z * delta;
        Vector3f pos = particle.getPosition();
        particle.setPosition(pos.x + dx, pos.y + dy, pos.z + dz);
    }

    @Override
    public void cleanup() {
        for (GameItem particle : getParticles()) {
            particle.cleanup();
        }
    }
}
```

Now we can extend the information that’s contained in the ```Scene``` class to include an array of ```ParticleEmitter``` instances.

```java
package org.lwjglb.engine;

// Imports here

public class Scene {

    // More attributes here

    private IParticleEmitter[] particleEmitters;
```

At this stage we can start rendering the particles. Particles will not be affected by lights and will not cast any shadow. They will not have any skeletal animation, so it makes sense to have specific shaders to render them. The shaders will be very simple, they will just render the vertices using the projection and modelview matrices and use a texture to set the colours.

The vertex shader is defined like this.

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec2 texCoord;
layout (location=2) in vec3 vertexNormal;

out vec2 outTexCoord;

uniform mat4 modelViewMatrix;
uniform mat4 projectionMatrix;

void main()
{
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    outTexCoord = texCoord;
}
```

The fragment shader is defined like this:

```glsl
#version 330

in vec2 outTexCoord;
in vec3 mvPos;
out vec4 fragColor;

uniform sampler2D texture_sampler;

void main()
{
    fragColor = texture(texture_sampler, outTexCoord);
}
```

As you can see  they are very simple, they resemble the pair of shaders used in the first chapters. Now, as in other chapters, we need to setup and use those shaders in the ```Renderer``` class. The shaders setup will be done in a method named ```setupParticlesShader``` which is defined like this:

```java
private void setupParticlesShader() throws Exception {
    particlesShaderProgram = new ShaderProgram();
    particlesShaderProgram.createVertexShader(Utils.loadResource("/shaders/particles_vertex.vs"));
    particlesShaderProgram.createFragmentShader(Utils.loadResource("/shaders/particles_fragment.fs"));
    particlesShaderProgram.link();

    particlesShaderProgram.createUniform("projectionMatrix");
    particlesShaderProgram.createUniform("modelViewMatrix");
    particlesShaderProgram.createUniform("texture_sampler");
}
```

And now we can create the render method named ```renderParticles``` in the ```Renderer``` class which is defined like this:

```java
private void renderParticles(Window window, Camera camera, Scene scene) {
    particlesShaderProgram.bind();

    particlesShaderProgram.setUniform("texture_sampler", 0);
    Matrix4f projectionMatrix = transformation.getProjectionMatrix();
    particlesShaderProgram.setUniform("projectionMatrix", projectionMatrix);

    Matrix4f viewMatrix = transformation.getViewMatrix();
    IParticleEmitter[] emitters = scene.getParticleEmitters();
    int numEmitters = emitters != null ? emitters.length : 0;

    for (int i = 0; i < numEmitters; i++) {
        IParticleEmitter emitter = emitters[i];
        Mesh mesh = emitter.getBaseParticle().getMesh();

        mesh.renderList((emitter.getParticles()), (GameItem gameItem) -> {
            Matrix4f modelViewMatrix = transformation.buildModelViewMatrix(gameItem, viewMatrix);
            particlesShaderProgram.setUniform("modelViewMatrix", modelViewMatrix);
        }
        );
    }
    particlesShaderProgram.unbind();
}
```

The fragment above should be self explanatory if you managed to get to this point, it just renders each particle setting up the required uniforms. We have now created all the methods we need to test the implementation of the particle effect. We just need to modify the ```DummyGame``` class we can setup a particle emitter and the characteristics of the base particle.

```java
Vector3f particleSpeed = new Vector3f(0, 1, 0);
particleSpeed.mul(2.5f);
long ttl = 4000;
int maxParticles = 200;
long creationPeriodMillis = 300;
float range = 0.2f;
float scale = 0.5f;
Mesh partMesh = OBJLoader.loadMesh("/models/particle.obj");
Texture texture = new Texture("/textures/particle_tmp.png");
Material partMaterial = new Material(texture, reflectance);
partMesh.setMaterial(partMaterial);
Particle particle = new Particle(partMesh, particleSpeed, ttl);
particle.setScale(scale);
particleEmitter = new FlowParticleEmitter(particle, maxParticles, creationPeriodMillis);
particleEmitter.setActive(true);
particleEmitter.setPositionRndRange(range);
particleEmitter.setSpeedRndRange(range);
this.scene.setParticleEmitters(new FlowParticleEmitter[] {particleEmitter});
```

We are using a plain filled circle as the particle’s texture by now, to better understand what’s happening. If you execute it you will see something like this.


