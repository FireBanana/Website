---
title: "Loading Gltf files in-Engine"
date: 2023-05-15T10:19:24+05:00
draft: false
---

For my engine, I had originally planned to go with Assimp, but ultimately decided on gltf support only as it would be a fun learning experience. Perhaps later I can add support for more formats. Since I did not want to write the entire parser from scratch, I decided to use TinyGLTF to load data from the `.glb` file, which presented it in a clean node hierarchy. Here is an example of how a simple triangle would look in a gltf file:

```json
// https://github.com/KhronosGroup/
{
  "scene": 0,
  "scenes" : [
    {
      "nodes" : [ 0 ]
    }
  ],
  
  "nodes" : [
    {
      "mesh" : 0
    }
  ],
  
  "meshes" : [
    {
      "primitives" : [ {
        "attributes" : {
          "POSITION" : 1
        },
        "indices" : 0
      } ]
    }
  ],

  "buffers" : [
    {
      "uri" : "data:application/octet-stream;base64,AAABAAIAAAAAAAAAAAAAAAAAAAAAAIA/AAAAAAAAAAAAAAAAAACAPwAAAAA=",
      "byteLength" : 44
    }
  ],
  "bufferViews" : [
    {
      "buffer" : 0,
      "byteOffset" : 0,
      "byteLength" : 6,
      "target" : 34963
    },
    {
      "buffer" : 0,
      "byteOffset" : 8,
      "byteLength" : 36,
      "target" : 34962
    }
  ],
  "accessors" : [
    {
      "bufferView" : 0,
      "byteOffset" : 0,
      "componentType" : 5123,
      "count" : 3,
      "type" : "SCALAR",
      "max" : [ 2 ],
      "min" : [ 0 ]
    },
    {
      "bufferView" : 1,
      "byteOffset" : 0,
      "componentType" : 5126,
      "count" : 3,
      "type" : "VEC3",
      "max" : [ 1.0, 1.0, 0.0 ],
      "min" : [ 0.0, 0.0, 0.0 ]
    }
  ],
  
  "asset" : {
    "version" : "2.0"
  }
}

```

There are a couple of members here that are the most essential for any model, namely, `meshes`, `buffers`, `bufferViews`, and `accessors`. To loosely summarize, the 'flow' looks something like `meshes`->`accessor`->`bufferView`->`buffer`. The `meshes` property contains a list of `primitives` which, in turn, consist of a list of `attributes` and a single value `indices`. These properties don't contain the actual data, but point to the index of the respective `accessor`.

The `accessor` contains an index for a `bufferView` that it points to, as well as the size, offset, and type of value that it refers to. In the same fashion, the `bufferView` holds an index for a `buffer`, as well as offset, length, and stride properties. All these values provide info of the span of data inside the buffer for this attribute.

For my current use case, I'll only be needing the vertex positions and the indices so I'll need to extract only those two things. Fortunately, it wasn't too hard, let's take a look of the code in detail.

```c++

	tinygltf::TinyGLTF loader;
	tinygltf::Model    model;
	std::string        err, wrn;

	loader.LoadBinaryFromFile(&model, &err, &wrn, path);

	std::vector<float>        bufferData;
	std::vector<unsigned int> indices;

```

At the start we use TinyGLTF to load the file from disk and set up two vectors which will contain the vertex positions and indices respectively.


```c++

	auto primitive = model.meshes[0].primitives[0];

	for (auto& attrib : primitive.attributes)
	{
		if (attrib.first != "POSITION") continue; //Position only currently
		auto& accessor = model.accessors[attrib.second];
		auto& bufferView = model.bufferViews[accessor.bufferView];

		auto byteStride = accessor.ByteStride(bufferView);
		auto size = accessor.count;

		auto& positionBuffer = model.buffers[bufferView.buffer];

		for (auto i = 0; i < size; ++i)
		{
			bufferData.push_back(
				*(reinterpret_cast<float*>(&(positionBuffer.data[(i * byteStride + bufferView.byteOffset)])))
			);

			bufferData.push_back(
				*(reinterpret_cast<float*>(&(positionBuffer.data[(i * byteStride + bufferView.byteOffset) + sizeof(float)]))));

			bufferData.push_back(
				*(reinterpret_cast<float*>(&(positionBuffer.data[(i * byteStride + bufferView.byteOffset) + sizeof(float) * 2])))
			);
		}
	}

```

In this case, we assume that there is only one `mesh` containing only one `primitive`. For the sake of clarity, we loop through the attributes inside the `primitive`, discarding everything except `POSITION`. This is where UVs, normals, and similar attributes can be processed. Then we extract the size and strides, as well as cache the reference to the bufferViews and buffers. The inner-loop is where most of the magic happens. Although I have not used `accessor.type` here, I know that the data for `POSITION` is laid out as a 3-component vector of floats. Since the data inside the buffer is stored as `unsigned char`, we'll need to convert it to `float`. Therefore, the values are retrieved `reinterpret_cast`ing the three component offset values for each of the attribute entries.

```c++

	auto& indexAccessor = model.accessors[primitive.indices];
	auto& indexBufferView = model.bufferViews[indexAccessor.bufferView];
	auto indexStride = indexAccessor.ByteStride(indexBufferView);
	auto& indexBuffer = model.buffers[indexBufferView.buffer];

	for (auto i = 0; i < indexAccessor.count; ++i)
	{
		auto val = *(reinterpret_cast<unsigned int*>(&(indexBuffer.data[i * indexStride + indexBufferView.byteOffset]))) & ((~0u) >> 16);

		indices.push_back(val);
	}

```

Indices share a similar approach; the required data is stored from the accessor and the `reinterpret_cast`ed. Something different to note here is the mask `((~0u) >> 16)`. Since unsigned char is 2 bytes long and unsigned int is 4 bytes long (on the MSVC compiler as of writing this article), the top 2 bytes will cause the value to be incorrect since thats data for another entry. Masking the 2 bytes out causes the correct value to show.

Note: this is a very crude and simple example, since I wanted to have very simple models that I process myself. Utilizing the other data that TinyGLTF provides, such as the data type, can help decode a larger quantity of models with lesser code. I will expand on this in the future when the need arises. Meanwhile, enjoy this picture of an unlit Suzanne that the above code loads:

![Suzanne](/images/gltf/gltf-1.png)