# Audio

Until this moment we have been dealing with graphics, but another key aspect of every game is audio. This capability is going to be addressed in this chapter with the help of [OpenAL](https://www.openal.org "OpenAL") (Open Audio Library). OpenAL is the OpenGL counterpart for audio, it allows us to play sounds through and abstraction layer. That layer isolates us from the underlying complexities of the audio subsystem. Besides that it allows us to “render” sounds in a 3D scene, where sounds can be set up in specific locations, attenuated with the distance and modified according to their velocity (simulating [Doppler ](https://en.wikipedia.org/wiki/Doppler_effect) effect)

LWJL supports OpenAL without requiring any additional download, it’s just ready to use. But before start coding we need to present the main elements involved when dealing with OpenAL, which are:

* Buffers.
* Sources.
* Listener.

Buffers store audio data, that is, music or sound effects. They are similar to the textures in the OpenGL domain. OpenAL expects audio data to be in PCM (Pulse Coded Modulation) format (either in mono or in stereo), so we cannot just dump MP3 or OGG files without converting them first to PCM.

The next elements are sources, which represent a location in a 3D space (a point) that emits sound. A source is associated to a buffer (only one at time) and can be defined the following attributes:

* A position, the location of the source ($$x$$, $$y$$ and $$z$$ coordinates). By the way, OpenAL uses a right handed Cartesian coordinate system as OpenGL, so you can assume (to simplify things) that your world coordinates are equal to the ones in the sound space coordinate system.
* A velocity, which specifies how fast the source is moving. This is used to simulate Doppler effect.
* A gain, which is used to modify the intensity of the sound (it’s like an amplifier factor).

A source has additional attibutes which will be shown later when describing the source code.

And last, but no least, a listener which is where the generated sounds are supposed to be heard. The Listener represents were the microphone is set in a 3D audio scene to receive the sounds. There is only one listener. It’s often said that audio rendering is done form the listener’s perspective. A listener shares some the attributes but it has some additional ones such as the orientation. The orientation represents where the listener is facing.

So an audio 3D scene is composed by a set of sound sources which emit sound and a listener that receives sound. The final perceived sound will depend on the distance of the listener to the different sources, their relative speed and the selected propagation models. Sources can share buffers and play the same data. The following figure depicts a sample 3D scene with the different element types involved.

![OpenAL concepts](/chapter22/openal_concepts.png)

So, let's start coding, we will create a new package under the name ```org.lwjglb.engine.sound``` that will host all the classes responsible of handling audio. We will first start with a class, named ```SoundBuffer``` that will represent an OpenAL buffer. A fragment of the definition of that class is shown below.

```java
package org.lwjglb.engine.sound;

// ... Some inports here

public class SoundBuffer {

    private final int bufferId;

    public SoundBuffer(String file) throws Exception {
        this.bufferId = alGenBuffers();
        try (STBVorbisInfo info = STBVorbisInfo.malloc()) {
            ShortBuffer pcm = readVorbis(file, 32 * 1024, info);

            // Copy to buffer
            alBufferData(buffer, info.channels() == 1 ? AL_FORMAT_MONO16 : AL_FORMAT_STEREO16, pcm, info.sample_rate());
        }
    }

    public int getBufferId() {
        return this.bufferId;
    }

    public void cleanup() {
        alDeleteBuffers(this.bufferId);
    }

    // ....
}
```

The constructor of the class expects a sound file (which may be in the classpath as the rest of resources) and creates a new buffer from it. The first thing that we do is create an OpenAL buffer with the call to ```alGenBuffers```. At the end our sound buffer will be identified by an integer which is like a pointer to it. Once the buffer has been created we dump the audio data in it. The constructor expects a file in OGG format, so we need it to transform it to PCM format. You can check how that's done in te source code, anyway, the source code has been extracted from the LWJGL OpenAL tests.

Previous versions of LWJGL had a helper class named ```WaveData```which was used to load audio files in WAV format. This class is no longer present in LWJGL 3. Nevertheless, you may get the source code from that class and use it in your games (maybe without requiring any changes).

The ```SoundBuffer``` class also provides a ```cleanup``` method to free the resources when we are done with it.

Let's continue by modelling an OpenAl, which will be implemented by class named ```SounSource```. The class is defined below.

```java
package org.lwjglb.engine.sound;

import org.joml.Vector3f;

import static org.lwjgl.openal.AL10.*;

public class SoundSource {

    private final int sourceId;

    public SoundSource(boolean loop, boolean relative) {
        this.sourceId = alGenSources();
        if (loop) {
            alSourcei(sourceId, AL_LOOPING, AL_TRUE);
        }
        if (relative) {
            alSourcei(sourceId, AL_SOURCE_RELATIVE, AL_TRUE);
        }
    }

    public void setBuffer(int bufferId) {
        stop();
        alSourcei(sourceId, AL_BUFFER, bufferId);
    }

    public void setPosition(Vector3f position) {
        alSource3f(sourceId, AL_POSITION, position.x, position.y, position.z);
    }

    public void setSpeed(Vector3f speed) {
        alSource3f(sourceId, AL_VELOCITY, speed.x, speed.y, speed.z);
    }

    public void setGain(float gain) {
        alSourcef(sourceId, AL_GAIN, gain);
    }

    public void play() {
        alSourcePlay(sourceId);
    }

    public boolean isPlaying() {
        return alGetSourcei(sourceId, AL_SOURCE_STATE) == AL_PLAYING;
    }

    public void pause() {
        alSourcePause(sourceId);
    }

    public void stop() {
        alSourceStop(sourceId);
    }

    public void cleanup() {
        stop();
        alDeleteSources(sourceId);
    }
}
```

The sound source class provides some methods to setup its position, the gain, and control methods for playing stopping and pausing it. Sound control actions are made over a source (not over the buffer), remember that several sources can share the same buffer. As in the ```SoundBuffer``` class, a ```SoundSource``` is identified by an id, which is used in each operation. This class also provides a ```cleanup``` method to free the served resources. But let’s examine the constructor. The first thing that we do is to create the source with the ```alGenSources``` call. Then, we set up some interesting properties using the constructor parameters.

The first parameter, ```loop```, indicates if the sound to be played should be in loop mode or not. By default, when a play action is invoked over a source the playing stops when the audio data is consumed. This is fine for some sounds, but some others, like background music, need to be played over and over again. Instead of manually controlling when the audio has stopped and re-launch the play process, we just simply set the looping property to true: “```alSourcei(sourceId, AL_LOOPING, AL_TRUE);```”.

The other parameter, ```relative```, controls if the position of the source is relative to the listener or not. In this case, when we set the position for a source, we basically are defining the distance (with a vector) to the listener, not the position in the OpenAL 3D scene, not the world position. This activated by the “```alSourcei(sourceId, AL_SOURCE_RELATIVE, AL_TRUE);”``` call. But, What can we use this for? This property is interesting for instance for background sounds that should be affected (attenuated) by the distance to the listener. Think, for instance, in background music or sound effects related to player controls. If we set these sources as relative, and set their position to $$(0, 0, 0)$$ they will not be attenuated.

Now it’s turn for the listener which, surprise, is modelled by a class named ```SoundListener```. Here’s the definition for that class.

```java
package org.lwjglb.engine.sound;

import org.joml.Vector3f;

import static org.lwjgl.openal.AL10.*;

public class SoundListener {

    public SoundListener() {
        this(new Vector3f(0, 0, 0));
    }

    public SoundListener(Vector3f position) {
        alListener3f(AL_POSITION, position.x, position.y, position.z);
        alListener3f(AL_VELOCITY, 0, 0, 0);

    }

    public void setSpeed(Vector3f speed) {
        alListener3f(AL_VELOCITY, speed.x, speed.y, speed.z);
    }

    public void setPosition(Vector3f position) {
        alListener3f(AL_POSITION, position.x, position.y, position.z);
    }

    public void setOrientation(Vector3f at, Vector3f up) {
        float[] data = new float[6];
        data[0] = at.x;
        data[1] = at.y;
        data[2] = at.z;
        data[3] = up.x;
        data[4] = up.y;
        data[5] = up.z;
        alListenerfv(AL_ORIENTATION, data);
    }
}
```
A difference you will notice from the previous classes is that there’s no need to create a listener. There will always be one listener, so no need to create one, it’s already there for us, so the constructor just sets a position. For the same reason there’s no need for a ```cleanup``` method. The class has methods also for setting listener position and velocity, as in the ```SoundSource``` class, but we have an extra method for changing the listener orientation. Let’s review it. Listener orientation is defined by two vectors, “at” vector and “up” one, which are shown in the next figure.

![Listener at and up vectors](/chapter22/listener_at_up.png)

The “at” vector basically points where the listener is facing, by default its coordinates are $$(0, 0, -1)$$. The “up” vector determines which direction is up for the listener and, by default it points to $$(0, 1, 0)$$. So the tree components of each of those two vectors are what are set in the ```alListenerfv``` method call. This method is used to transfer a set of floats (a variable number of floats) to a property, in this case, the orientation.

Before continuing I would like to stress out some concepts in relation to source and listener speeds. The relative speed between sources and listener will cause OpenAL to simulate Doppler effect. In case you don’t know Doppler effect is what causes that a moving object that is getting closer to you seems to emit in a higher frequency that it emits when is walking away. The thing, is that, simply by setting a source or listener velocity, OpenAL will not update their position for you. It will use the velocity and the position to calculate the Doppler effect according to those values, but they won’t be modified. So, if you want to simulate a moving source or listener you must take care of updating their positions and velocities in the game loop.

Now that we have modelled the key elements we can set them up to work, we need to initialize OpenAL library, so we will create a new class named ```SoundManager``` that will handle this. Here’s a fragment of the definition of this class.

```java
package org.lwjglb.engine.sound;

// Imports here

public class SoundManager {

    private long device;

    private long context;

    private SoundListener listener;

    private final List<SoundBuffer> soundBufferList;

    private final Map<String, SoundSource> soundSourceMap;

    private final Matrix4f cameraMatrix;

    public SoundManager() {
        soundBufferList = new ArrayList<>();
        soundSourceMap = new HashMap<>();
        cameraMatrix = new Matrix4f();
    }

    public void init() throws Exception {
        this.device = alcOpenDevice((ByteBuffer) null);
        if (device == NULL) {
            throw new IllegalStateException("Failed to open the default OpenAL device.");
        }
        ALCCapabilities deviceCaps = ALC.createCapabilities(device);
        this.context = alcCreateContext(device, (IntBuffer) null);
        if (context == NULL) {
            throw new IllegalStateException("Failed to create OpenAL context.");
        }
        alcMakeContextCurrent(context);
        AL.createCapabilities(deviceCaps);
    }
```


CHAPTER IN PROGRESS




