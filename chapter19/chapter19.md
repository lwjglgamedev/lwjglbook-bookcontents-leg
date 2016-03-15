# Animations

By now we have just loaded static 3D models, in this chapter we will learn how to animate them. When thinking about animations the first approach is to create different meshes for each model positions, load them up into the GPU and draw them sequentially to create the illusion of animation. Although this approach is perfect for some games it's not very efficient (in terms of memory consumption).

This where skeletal animation comes to play. In skeletal animation the way a model animates is defined by its underlying skeleton. A skeleton is defined by a hierarchy of special points called joints. Those points are defined by their position and rotation. Since it's a hierarchy the final position for each joint is affected by their parents. For instance, think on a wrist, the position of a wrist is modified if a character moves the elbow and also if it moves the shoulder.

Joints do not need to represent a physical bone or articulation, they are artifacts that allows the creatives to model an animation. Besides joints we still have vertices, the points that define the triangles that compose a 3D model. But, in skeletal animation, vertices are drawn based on the position of the joints it is related to.

In this chapter we will use MD5 format to load animated models. MD5 format was create by ID Software, the creators of Doom, and it’s basically a text based file format which is well understood.  Another approach would be to use the [Collada](https://en.wikipedia.org/wiki/COLLADA) format, which is a public standard supported by many tools. Collada is an XML based format but as a downside it’s very complex (The specification for the 1.5 version has more than 500 pages). So, we will stick which a much more simple format, MD5, that will allow us to focus in the concepts of the skeletal animation and to create a working sample.

You can also export some models from Blnder to MD5 format via specific addons that you can find on the Internet ([http://www.katsbits.com/tools/#md5]())

In this chapter I’ve consulted many different sources, but I  have found two that provide a very good explanation about how t create an animated model using MD5 files. Theses sources can be consulted at:
* [http://www.3dgep.com/gpu-skinning-of-md5-models-in-opengl-and-cg/](http://www.3dgep.com/gpu-skinning-of-md5-models-in-opengl-and-cg/)
* [http://ogldev.atspace.co.uk/www/tutorial38/tutorial38.html](http://ogldev.atspace.co.uk/www/tutorial38/tutorial38.html)

So let’s start by writing the code that parses MD5 files. The MD5 format defines two types of files:
* The mesh definition file: which defines the joints and the vertices and textures that compose the set of meshes that form the 3D model. This file usually has a extension named “.md5mesh”. 
* The animation definition file: which defines the animations that can be applied to the model. This file usually has a extension named “.md5anim”. 

A MD5 file is composed by a header an different sections contained between braces. Let’s start examining the mesh definition file. In the resources folder you will find several models in MD5 format. If you open one of them you can see a structure similar like this.

![MD5 Structure](md5_structure.png) 

The first structure that you can find in the mesh definition file is the header. You can see below header’s content from one of the samples provided:

```text
MD5Version 10
commandline ""

numJoints 33
numMeshes 6
```

The header defines the following attributes:
* The version of the MD5 specification that it complies to.
* The command used to generate this file (from a 3D modelling tool).
* The number of Joints that are defined in the joints section
* The number of Meshes (the number of meshes sections expected).
The Joints sections defines the joints, as it names states, their positions and their relationships. A fragment of the joints section of one of the sample models is shown below.

```text
joints {
	"origin"	-1 ( -0.000000 0.016430 -0.006044 ) ( 0.707107 0.000000 0.707107 )		// 
	"sheath"	0 ( 11.004813 -3.177138 31.702473 ) ( 0.307041 -0.578614 0.354181 )		// origin
	"sword"	1 ( 9.809593 -9.361549 40.753730 ) ( 0.305557 -0.578155 0.353505 )		// sheath
	"pubis"	0 ( 0.014076 2.064442 26.144581 ) ( -0.466932 -0.531013 -0.466932 )		// origin
              ……
}
```

A Joint is defined by the following attributes:
* Joint name, a textual attribute between quotes.
* Joint parent, using an index which points to the  parent joint using its position in the joints list. The root joint has a parent equals to -1.
* Joint position, defined in  model space coordinate system.
* Joint orientation, defined also in model space coordinate system. The orientation in fact is a quaternion whose w-component is not included.

Before continuing explaining the rest of the file let’s talk about quaternions. Quaternions are four component elements that are used to represent rotation. Up to now, we have been using Euler angles (yaw, pitch roll) to define rotations, which basically define rotation around the x, y and z angles. Euler angles present some problems when working with rotations, specifically you must be aware of the correct order to apply de rotations and some operations can get very complex.
This where quaternions come to help in order to solve this complexity. As it has been said before a quaternion is defined as a set of 4 numbers (x, y, z, w). Quaternions define a rotation axis and the rotation angle around that axis.

![Quaternion](quaternion.png)
 
You can check in the web the mathematical definition of each of the components but the good news is that JOML, the math library we are using, provides support for them. We can construct rotation matrices based on quaternions and perform some transformation to vectors with them.

Let’s get back to the joints definition, the w component is missing but it can be easily calculated with the help of the rest of the values. We will review it when the code that parses MD5 files is explained.
After the joints definition you can find the definition of the different meshes that compose a model. Below you can find a fragment of a Mesh definition form one of the samples.

```text
mesh {
	shader "/textures/bob/guard1_body.png"

	numverts 494
	vert 0 ( 0.394531 0.513672 ) 0 1
	vert 1 ( 0.447266 0.449219 ) 1 2
	...
	vert 493 ( 0.683594 0.455078 ) 864 3

	numtris 628
	tri 0 0 2 1
	tri 1 0 1 3
	...
	tri 627 471 479 493

	numweights 867
	weight 0 5 1.000000 ( 6.175774 8.105262 -0.023020 )
	weight 1 5 0.500000 ( 4.880173 12.805251 4.196980 )
	...
	weight 866 6 0.333333 ( 1.266308 -0.302701 8.949338 )
}
```

Let’s review the structure presented above:
* A Mesh starts by defining a texture file. Keep in mind that the path that you will find here is the one used by the tool that created that model. That path may not match the one that is used to load those files. You have two approaches here, either you change the base path dynamically or either you change that path by hand. I’ve chosen the last one, the simplest one.
* Next you can find the vertices definition. A vertex is defined by the following attributes:
* * Vertex index.
* * Texture coordinates.
* * The index of the first weight definition that affects this vertex.
* * The number of weights to consider.
* After the vertices, the triangles that form this mesh are defined. The triangles define the way that vertices are organized using their indices.
* Finally, the weights are defined. A Weight definition is composed by:
* * A Weight index.
* * A Joint index, that points to the joint related to this weight.
* * A bias factor, that is uses to modulate the effect of this weight.
* * A position of this weight.

The following picture tries to depict the relation between the components described above using sample data.
 
![Mesh elements](mesh_elements.png)