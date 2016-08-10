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

So, let's start coding, we will cerate a new package under the name ```org.lwjglb.engine.sound``` that will host all the clases responsible oh handling audio. We will first start with a class, named ```SoundBuffer``` that will represent an OpenAL buffer.

CHAPTER IN PROGRESS




