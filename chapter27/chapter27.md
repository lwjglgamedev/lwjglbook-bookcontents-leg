# ![](/assets/model.png)Assimp

The capability of loading complex 3d models in different formats is crucial in order to write a game. The task of writing parsers for some of them would require lots of work. Even just supporting a single format can be time consuming. For instance, the wavefront loader described in chapter 9, only parses a small subset of the specification \(materials are not handled at all\).

Fortunately, the [Assimp ](http://assimp.sourceforge.net/)library already can be used to parse many common 3D formats. It’s a C++ library which can load static and animated models in a variety of formats. LWJGL provides the bindings to use them from Java code. In this chapter, we will explain how it can be used.

The first thing is adding assimp maven dependencies to the project pom.xml. We need to add compile time and runtime dependencies.

```xml
<dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-assimp</artifactId>
    <version>${lwjgl.version}</version>
</dependency>
<dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-assimp</artifactId>
    <version>${lwjgl.version}</version>
    <classifier>${native.target}</classifier>
    <scope>runtime</scope>
</dependency>
```

Once the dependencies has been set, we will cerate a new class named StaticMeshesLoader that will be used to load meshes with no animations. The class defines two static public methods:

```java
public static Mesh[] load(String resourcePath, String texturesDir) throws Exception {
    return load(resourcePath, texturesDir, aiProcess_JoinIdenticalVertices | aiProcess_Triangulate | aiProcess_FixInfacingNormals);
}

public static Mesh[] load(String resourcePath, String texturesDir, int flags) throws Exception {
    // ....
```

Both methods have the following arguments:

* `resourcePath`: The path to the file where the model file is located. This is an absolute path, because Assimp may need to load additional files and may use the same base path as the resource path \(For instance, material files for wavefront, OBJ, files\).

* `texturesDir`: The path to the directory that will hold the textures for this model. This a CLASSPATH relative path. For instance, a wavefront material file may define several texture files. The code, expect this files to be located in the `texturesDir`directory. If you find texture loading errors you may need to manually tweak these paths in the model file.

The second method has an extra argument named `flags`. This parameter allows to tune the loading process. The firstmethodsjust invokes the secondoneand passes some values that are useful in most of the situations:

* `aiProcess_JoinIdenticalVertices`: This flag reduces the number of vertices that are used, identifiying those that can be reused between faces.

* `aiProcess_Triangulate`: The model may use quads or other geometries to define their elements. Since we are only dealing with triangles, we must use this flag to split all he faces into triangles \(if needed\).

* `aiProcess_FixInfacingNormals`: This flags try to reverse normals that may point inwards.

There are many other flags that can be used, you chan check them in the LWJGL Javadoc documentation.

Let’s go back to the second constructor. The first thing we do is invoke the `aiImportFile`method to load the model with the selectedflags.

```java
AIScene aiScene = aiImportFile(resourcePath, flags);
if (aiScene == null) {
    throw new Exception("Error loading model");
}
```

The rest of the code for the constructor is a as follows:

```java
int numMaterials = aiScene.mNumMaterials();
PointerBuffer aiMaterials = aiScene.mMaterials();
List<Material> materials = new ArrayList<>();
for (int i = 0; i < numMaterials; i++) {
    AIMaterial aiMaterial = AIMaterial.create(aiMaterials.get(i));
    processMaterial(aiMaterial, materials, texturesDir);
}

int numMeshes = aiScene.mNumMeshes();
PointerBuffer aiMeshes = aiScene.mMeshes();
Mesh[] meshes = new Mesh[numMeshes];
for (int i = 0; i < numMeshes; i++) {
    AIMesh aiMesh = AIMesh.create(aiMeshes.get(i));
    Mesh mesh = processMesh(aiMesh, materials);
    meshes[i] = mesh;
}

return meshes;
```

We process the materials contained in the model. Materials define colour and textures to be used by the meshes that compose the model. Then we process the different meshes. A model can define several meshes and each of them can use one of the materials defined for the model.

If you examine the code above you may see that many of the calls to the Assimp library return `PointerBuffer`instances. You can think about them like C pointers, they just point to a memory region which contain data. You need to know in advance the type of data that they hold in order to process them. In the case of materials, we iterate over that buffer creating instances of theAIMaterialclass. In the second case, we iterate over the buffer that holds mesh data creating instance of the `AIMesh`class.

Let’s examine the `processMaterial`method.

```java
private static void processMaterial(AIMaterial aiMaterial, List<Material> materials, String texturesDir) throws Exception {
    AIColor4D colour = AIColor4D.create();

    AIString path = AIString.calloc();
    Assimp.aiGetMaterialTexture(aiMaterial, aiTextureType_DIFFUSE, 0, path, (IntBuffer) null, null, null, null, null, null);
    String textPath = path.dataString();
    Texture texture = null;
    if (textPath != null && textPath.length() > 0) {
        TextureCache textCache = TextureCache.getInstance();
        texture = textCache.getTexture(texturesDir + "/" + textPath);
    }

    Vector4f ambient = Material.DEFAULT_COLOUR;
    int result = aiGetMaterialColor(aiMaterial, AI_MATKEY_COLOR_AMBIENT, aiTextureType_NONE, 0, colour);
    if (result == 0) {
        ambient = new Vector4f(colour.r(), colour.g(), colour.b(), colour.a());
    }

    Vector4f diffuse = Material.DEFAULT_COLOUR;
    result = aiGetMaterialColor(aiMaterial, AI_MATKEY_COLOR_DIFFUSE, aiTextureType_NONE, 0, colour);
    if (result == 0) {
        diffuse = new Vector4f(colour.r(), colour.g(), colour.b(), colour.a());
    }

    Vector4f specular = Material.DEFAULT_COLOUR;
    result = aiGetMaterialColor(aiMaterial, AI_MATKEY_COLOR_SPECULAR, aiTextureType_NONE, 0, colour);
    if (result == 0) {
        specular = new Vector4f(colour.r(), colour.g(), colour.b(), colour.a());
    }

    Material material = new Material(ambient, diffuse, specular, 1.0f);
    material.setTexture(texture);
    materials.add(material);
}
```

We check if the material defines a texture or not. If so, we load the texture. We have created a new class named `TextureCache`which caches textures. This is due to the fact that several meshes may share the same texture and we do not want to waste space loading again and again the same data. Then we try to get the colours of the material for the ambient, diffuse and specular components. Fortunately, the definition that we had for a material already contained that information.

The `TextureCache`definition is very simple is just a map that indexes the different textures by the path to the texture file \(You can check directly in the source code\). Due to the fact, that now textures may use different image formats \(PNG, JPEG, etc.\), we have modified the way that textures are loaded. Instead of using the PNG library, we now use the STB library to be able to load more formats.

Let’s go back to the `StaticMeshesLoader`class. The `processMesh`is defined like this.

```java
private static Mesh processMesh(AIMesh aiMesh, List<Material> materials) {
    List<Float> vertices = new ArrayList<>();
    List<Float> textures = new ArrayList<>();
    List<Float> normals = new ArrayList<>();
    List<Integer> indices = new ArrayList();

    processVertices(aiMesh, vertices);
    processNormals(aiMesh, normals);
    processTextCoords(aiMesh, textures);
    processIndices(aiMesh, indices);

    Mesh mesh = new Mesh(Utils.listToArray(vertices),
        Utils.listToArray(textures),
        Utils.listToArray(normals),
        Utils.listIntToArray(indices)
    );
    Material material;
    int materialIdx = aiMesh.mMaterialIndex();
    if (materialIdx >= 0 && materialIdx < materials.size()) {
        material = materials.get(materialIdx);
    } else {
        material = new Material();
    }
    mesh.setMaterial(material);

    return mesh;
}
```

A `Mesh`is defined by a set of vertices position, normals directions, texture coordinates and indices. Each of these elements are processed in the `processVertices`, `processNormals`, `processTextCoords`and `processIndices`method. A Mesh also may point to a material, using its index. If the index corresponds to the previously processed materials we just simply associate them to the `Mesh`.

The `processXXX` methods are very simple, they just invoke the corresponding method over the `AIMesh`instance that returns the desired data. For instance, the process processVertices is defined like this:

```java
private static void processVertices(AIMesh aiMesh, List<Float> vertices) {
    AIVector3D.Buffer aiVertices = aiMesh.mVertices();
    while (aiVertices.remaining() > 0) {
        AIVector3D aiVertex = aiVertices.get();
        vertices.add(aiVertex.x());
        vertices.add(aiVertex.y());
        vertices.add(aiVertex.z());
    }
}
```

You can see that get get a buffer to the vertices by invoking the `mVertices`method. We just simply process them to create a `List`of floats that contain the vertices positions. Since, the method retyrns jusst a buffer you could pass that information directly to the OpenGL methods that create vertices. We do not do it that way for two reasons. The first one is try to reduce as much as possible the modifications over the code base. Second one is that by loading into an intermediate structure you may be able to perform some pros-processing tasks and even debug the loading process.

If you want a sample of the much more efficient approach, that is, directly passing the buffers to OpenGL, you can check this [sample](https://github.com/LWJGL/lwjgl3-demos/blob/master/src/org/lwjgl/demo/opengl/assimp/WavefrontObjDemo.java).

The `StaticMeshesLoader`makes the `OBJLoader`class obsolete, so it has been removed form the base source code. A more complex OBJ file is provided as a sample, if you run it you will see something like this:

![](/chapter27/model.png)

