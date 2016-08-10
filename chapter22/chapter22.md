# Audio

Until this moment we have been dealing with graphics, but another key aspect of every game is audio. This capability is going to be addressed in this chapter with the help of [OpenAL](https://www.openal.org "OpenAL") (Open Audio Library). OpenAL is the OpenGL counterpart for audio, it allows us to play sounds through and abstraction layer. That layer isolates us from the underlying complexities of the audio subsystem. Besides that it allows us to “render” sounds in a 3D scene, where sounds can be set up in specific locations, attenuated with the distance and modified according to their velocity (simulating [Doppler ](https://en.wikipedia.org/wiki/Doppler_effect) effect)

LWJL supports OpenAL without requiring any additional download, it’s just ready to use. But before start coding we need to present the main elements involved when dealing with OpenAL, which are:

* Buffers.
* Sources.
* Listener.

Buffers store audio data, that is, music or sound effects. They are similar to the textures in the OpenGL domain. OpenAL expects audio data to be in PCM (Pulse Coded Modulation) format (either in mono or in stereo), so we cannot just dump MP3 or OGG files without converting them first to PCM.

The next elements are sources, which represent a location in a 3D space (a point) that emits sound. A source is associated to a buffer (only one at time) and can be defined the following attributes:

* A position, the location of the source ($$x$$, $$y$$ and $$z$$ coordinates).
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
CHAPTER IN PROGRESS




