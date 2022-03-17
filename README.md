# compushady
Python module for easily running Compute Shaders

![compushadyMandlebrot](compushady_mandlebrot.gif?raw=true "compushady Mandlebrot")

Join the Discord server for support: https://discord.gg/EFvWaaxdR4

Currently d3d12 (Windows), vulkan (Linux, Mac and Windows), metal (Mac), and d3d11 (Windows) are supported. 

You can write shaders in HLSL and they will be compiled into the appropriate format (DXIL, DXBC, SPIR-V, MSL ...) automatically (using the DXC compiler included in the module as well as SPIRV-Cross).

The Mandlebrot anim gif you are seeing above has been generated by compushady itself: https://github.com/rdeioris/compushady/blob/main/examples/mandlebrot_gif.py

If you are looking for something cool, try this naive pong implementation: https://github.com/rdeioris/compushady/blob/main/examples/pong.py

Python 3.6 is the minimal supported version and you obviously need a system with at least one GPU.

An OpenGL backend is expected to be included (sooner or later).

## Quickstart

```sh
pip install compushady
```

`Note for Linux`:
(if you are building from sources, be sure vulkan and x11 headers are installed, ```libvulkan-dev``` and ```libx11-dev``` on a debian based distribution)

`Note for Mac`:
(the vulkan headers will be searched in /usr/local or by reading the VULKAN_SDK environment variable, otherwise only the metal backend will be built)

`Note for Windows`:
(the vulkan headers will be searched by reading the VULKAN_SDK environment variable, automatically set by the LunarG installer. If no Vulkan SDK is found only the d3d12 and d3d11 backends will be built)

### Enumerate compute devices

```py
import compushady

for device in compushady.get_discovered_devices():
    name = device.name
    video_memory_in_mb = device.dedicated_video_memory // 1024 // 1024
    print('Name: {0} Dedicated Memory: {1} MB'.format(name, video_memory_in_mb))
```

### Upload data to a Texture

A Texture is an object available in the GPU memory. To upload data into it you need a so-called 'staging buffer'.
A staging buffer is a block of memory allocated in your system ram that is mappable from your GPU. Using it the GPU can copy data from that buffer to the texture memory.
To wrap up: if you want to upload data to the GPU, you first need to map a memory area in your ram (a buffer) and then ask the GPU to copy it in its memory (this assumes obviously a discrete GPU with on-board memory)

```py
from compushady import Buffer, Texture2D, HEAP_UPLOAD
from compushady.formats import R8G8B8A8_UINT

# creates a 8x8 texture in GPU with the classig RGBA 8 bit format
texture = Texture2D(8, 8, R8G8B8A8_UINT)  
# creates a staging buffer with the right size and in memory optimized for uploading data
staging_buffer = Buffer(texture.size, HEAP_UPLOAD)
# upload a bunch of pixels data into the staging_buffer
staging_buffer.upload(b'\xff\x00\x00\xff') # first pixel as red
# copy from the staging_buffer to the texture
staging_buffer.copy_to(texture)
```

### Reading back from GPU memory to system memory

Now that you have your data in GPU memory, you can manipulate them using a compute shader but, before seeing this, we need to learn how to copy back data from the texture memory to our system ram. We need a buffer again (this time a readback one):

```python
from compushady import HEAP_READBACK, Buffer, Texture2D, HEAP_UPLOAD
from compushady.formats import R8G8B8A8_UINT

# creates a 8x8 texture in GPU with the classig RGBA 8 bit format
texture = Texture2D(8, 8, R8G8B8A8_UINT)  
# creates a staging buffer with the right size and in memory optimized for uploading data
staging_buffer = Buffer(texture.size, HEAP_UPLOAD)
# upload a bunch of pixels data into the staging_buffer
staging_buffer.upload(b'\xff\x00\x00\xff') # first pixel as red
# copy from the staging_buffer to the texture
staging_buffer.copy_to(texture)

# do something with the texture...

# prepare the readback buffer
readback_buffer = Buffer(texture.size, HEAP_READBACK)
# copy from texture to the readback buffer
texture.copy_to(readback_buffer)

# get the data as a python bytes object (just the first 4 bytes)
print(readback_buffer.readback(4))
```

### Your first compute shader

We are going to run code in the GPU!
We will start with simple logic: we will just swap the red channel with the green one.
For doing this we need to write an HLSL shader that will take our texture as an input/output object:

```python
from compushady import HEAP_READBACK, Buffer, Texture2D, HEAP_UPLOAD, Compute
from compushady.formats import R8G8B8A8_UINT
from compushady.shaders import hlsl

# creates a 8x8 texture in GPU with the classig RGBA 8 bit format
texture = Texture2D(8, 8, R8G8B8A8_UINT)
# creates a staging buffer with the right size and in memory optimized for uploading data
staging_buffer = Buffer(texture.size, HEAP_UPLOAD)
# upload a bunch of pixels data into the staging_buffer
staging_buffer.upload(b'\xff\x00\x00\xff')  # first pixel as red
# copy from the staging_buffer to the texture
staging_buffer.copy_to(texture)

# do something with the texture...

shader = """
RWTexture2D<uint4> texture : register(u0);
[numthreads(2, 2, 1)]
void main(int3 tid : SV_DispatchThreadID)
{
    uint4 color = texture[tid.xy];
    uint red = color.r;
    color.r = color.g;
    color.g = red;
    texture[tid.xy] = color;
}
"""
compute = Compute(hlsl.compile(shader), uav=[texture])
compute.dispatch(texture.width // 2, texture.height // 2, 1)

# prepare the readback buffer
readback_buffer = Buffer(texture.size, HEAP_READBACK)
# copy from texture to the readback buffer
texture.copy_to(readback_buffer)

# get the data as a python bytes object (just the first 8 bytes)
print(readback_buffer.readback(8))
```

## API

This section covers the compushady API in detail (class by class)

### compushady.Device

This class represents a compute device (a GPU generally) on your system.

There can be multiple devices on a system, by default compushady will always choose the one with most dedicated memory (but you are free to specify a device whenever you create a resource)

As already seen you can get the list of devices using ```compushady.get_discovered_devices()``` or retrieve the current 'best' one with ```compushady.get_best_device()```

A compushady.Device object has the following fields:

* ```name```: a string with the device description
* ```dedicated_video_memory```: the amount (in bytes) of on-board (GPU) memory
* ```dedicated_system_memory```: the amount of system memory the OS has dedicated to the GPU (generally meaningful only on Windows)
* ```shared_system_memory```: the amount of system memory usable by the device (GPU)
* ```vendor_id```: an integer representing the vendor id code
* ```device_id```: an integer representing the device id code
* ```is_hardware```: True if it is a hardware devices (not an emulated one)
* ```is_discrete```: True if it is a discrete adapter (a dedicated GPU)

The ```compushady.get_current_device()``` function returns the currently set GPU device, you can override the current device using ```compushady.set_current_device(index)``` where 'index' is the index of one of the elements returned by ```compushady.get_discovered_devices()```.
You can change the current device even from the command line using the ```COMPUSHADY_DEVICE``` environment variable:

```sh
COMPUSHADY_DEVICE=2 python3 your_gpu_app.py
```

## compushady.Buffer

This class represents a resource accessible by the GPU that can be in system RAM or GPU dedicated memory.
Buffers are generic blobs of data that you can use as a plain storage for your compute shaders or staging/readback buffers when dealing with textures.

When you create a Buffer you need to specify its dimension (in bytes) and (optionally) the type of memory he needs to use: HEAP_DEFAULT (GPU memory), HEAP_UPLOAD (system memory optimized for writing) or HEAP_READBACK (system memory optimized for reading)

```python
import compushady

buffer_in_gpu = compushady.Buffer(64)
buffer_in_gpu2 = compushady.Buffer(128, compushady.HEAP_DEFAULT)
staging_buffer = compushady.Buffer(64, compushady.HEAP_UPLOAD)
readback_buffer = compushady.Buffer(256, compushady.HEAP_READBACK)
```

Buffers created with HEAP_UPLOAD exposes the ```upload(data, offset=0)``` and ```upload2d(data, row_pitch, height, bytes_per_pixel)``` methods

Bufers created with HEAP_READBACK exposes the ```readback(size=0, offset=0)```, ```readback2d(row_pitch, height, bytes_per_pixel)``` and ```readback_to_buffer(buffer, offset=0)``` methods

Buffers expose the ```size``` property returning the size in bytes.

Buffer can even be `structured` and `formatted`:

This is an HLSL shader using a StructuredBuffer object

```hlsl
struct Data
{
    uint field0;
    float field1;
};
RWStructuredBuffer<Data> buffer: register(u0);
[numthreads(1, 1, 1)]
void main()
{
    buffer[0].field0 = 1;
    buffer[0].field1 = 2.0;
    buffer[1].field0 = 4;
    buffer[1].field1 = 4.0;
}
```
You can create a 'structured' buffer by passing the `stride` option with the size of the structure:

```py
# will create a buffer of 16 bytes, divided in two structures of 8 bytes each (the struct Data)
structured_buffer = Buffer(size=16, stride=8)
```

Or you can use a typed buffer:

```hlsl
RWBuffer<float4> buffer: register(u0);
[numthreads(1, 1, 1)]
void main()
{
    buffer[0] = float4(1, 2, 3, 4);
    buffer[1] = float4(5, 6, 7, 8);
}
```

For which you can specify the format parameter:

```py
# will create a buffer of 32 bytes, divided in two 16 bytes blocks, each one representing 4 32bits float values)
typed_buffer = Buffer(size=32, format=compushady.formats.R32G32B32A32_FLOAT)
```

## compushady.Texture2D

A Texture2D object is a bidimensional (width and height) texture available in the GPU memory. You can read it from your Compute shader or blit it to a Swapchain.
For creating a Texture2D object you need to specify its width, height and the pixel format.

```python
from compushady import Texture2D
from compushady.formats import R8G8B8A8_UINT
texture = compushady.Texture2D(1024, 1024, R8G8B8A8_UINT)
```

Textures memory is always in GPU, so whenever you need to access (read or write) pixels/texels data of a texture from your main program, you need a staging or a readback buffer.

## compushady.Texture1D and compushady.Texture3D

They are exactly like Texture2D but monodimensional (height=1) for Texture1D and tridimensional for Texture3D (you can see it as a group of slices, each one containing a bidimensional texture). For Texture3D you need to specify an additional parameter (the depth) representing the number of 'slices':


```python
from compushady import Texture3D
from compushady.formats import R8G8B8A8_UINT
texture = compushady.Texture3D(1024, 1024, 4, R8G8B8A8_UINT) # you can see it as 4 1024x1024 textures
```

All of the textures types expose the following properties:

* width
* height
* depth
* row_pitch (bytes for each line)
* size (dimension of the texture in bytes)

## compushady.Compute

To build a Compute object (the one running the compute shader), you need (obviously) a shader blob (you can build it using the ```compushady.shaders.hlsl.compile``` function) and the resources (buffers and textures) you want to manage in the shader itself.

Note: while on DirectX based backends you need a DXIL/DXCB shader blob, on Vulkan any shader compiler able to generate SPIR-V blobs will be good for compushady (you can even precompile your shaders and store the SPIR-V blobs on files that you can load in compushady)

compushady uses the DirectX12 naming conventions: ```CBV``` (Constant Buffer View) for constant buffers (generally little ammount of data that do not change during the compute shader execution), ```SRV``` (Shader Resource View) for buffers and textures you need to read in the shader, and ```UAV``` (Unordered Access View) for buffers and textures that need to be written by the shader.

This is a quick Compute object implementing a copy from a texture (the SRV, filled with random data uploaded in a buffer) to another one (the UAV) doubling pixel values:

```py
from compushady import HEAP_UPLOAD, Buffer, Texture2D, Compute
from compushady.formats import R8G8B8A8_UINT
from compushady.shaders import hlsl
import random

source_texture = Texture2D(512, 512, R8G8B8A8_UINT)
destination_texture = Texture2D(512, 512, R8G8B8A8_UINT)

staging_buffer = Buffer(source_texture.size, HEAP_UPLOAD)
staging_buffer.upload(bytes([random.randint(0, 255) for i in range(source_texture.size)]))

staging_buffer.copy_to(source_texture)

shader = """
Texture2D<uint4> source : register(t0);
RWTexture2D<uint4> destination : register(u0);

[numthreads(8,8,1)]
void main(uint3 tid : SV_DispatchThreadID)
{
    destination[tid.xy] = source[tid.xy] * 2;
}
"""

compute = Compute(hlsl.compile(shader), srv=[source_texture], uav=[destination_texture])
compute.dispatch(source_texture.width // 8, source_texture.height // 8, 1)
````

What is that 8 value ?

A compute shader can be executed in parallel, with a level of parallelism (read: how many cores will run that code) specified by the numthreads attribute in the shader. That is a tridimensional value, so to get the final number of running threads you need to do x * y * z. In our case 64 threads can potentially run in parallel.

As we want to process 512 * 512 elements (the pixels in the texture) we want to run the compute shader only the amount of times required to fill the whole texture (the amount of executions are the arguments of the dispatch method, again as a tridimensional value)

If you set teh arguments of dispatch to (1,1,1) you will only copy the top left 8x8 quad of the texture.

Cool, but how can i check if everything worked well ?

We have obviously tons of ways, but let's use a popular one: Pillow (append that code after the dispatch() call)

```
from PIL import Image

readback_buffer = Buffer(source_texture.size, HEAP_READBACK)
destination_texture.copy_to(readback_buffer)

image = Image.frombuffer('RGBA', (destination_texture.width,
                         destination_texture.height), readback_buffer.readback())
image.show()
```

If everything goes well a window should open with a 512x512 image with random pixel colors.

Try experimenting with different dispatch() arguments to see how the behaviour changes.

## compushady.Swapchain

While very probably you are going to run compushady in a headless environment, the module exposes a Swapchain object for blitting your textures on a window.
For creating a swapchain you need to specify a window handle to attach to (this is operating system dependent), a format (generally B8G8R8A8_UNORM) and the number of buffers (generally 3). When you want to 'blit' a texture to the swapchain you just call the ```compushady.Swapchain.present(texture, x=0, y=0)``` method (with x and y you can specify where to blit the texture). The Swapchain always waits for the VSYNC.

This is an example of doing it using the glfw module (with a swapchain with 3 buffers):

```python
import glfw
from compushady import HEAP_UPLOAD, Buffer, Swapchain, Texture2D
from compushady.formats import B8G8R8A8_UNORM
import platform
import random

glfw.init()
# we do not want implicit OpenGL!
glfw.window_hint(glfw.CLIENT_API, glfw.NO_API)

target = Texture2D(256, 256, B8G8R8A8_UNORM)
random_buffer = Buffer(target.size, HEAP_UPLOAD)

window = glfw.create_window(
    target.width, target.height, 'Random', None, None)

if platform.system() == 'Windows':
    swapchain = Swapchain(glfw.get_win32_window(
        window), B8G8R8A8_UNORM, 3)
else:
    swapchain = Swapchain((glfw.get_x11_display(), glfw.get_x11_window(
        window)), B8G8R8A8_UNORM, 3)

while not glfw.window_should_close(window):
    glfw.poll_events()
    random_buffer.upload(bytes([random.randint(0, 255), random.randint(
        0, 255), random.randint(0, 255), 255]) * (target.size // 4))
    random_buffer.copy_to(target)
    swapchain.present(target)

swapchain = None  # this ensures the swapchain is destroyed before the window

glfw.terminate()
```

## Accessing native GPU resources (advanced usage)

TODO

## Multithreading

TODO

## Backends

There are currently 4 backends for GPU access: vulkan, metal, d3d12 and d3d11 (this last one is way slower than the others and included just for backward compatibility)

There is one shader backend for HLSL (GLSL support is in progress) based on Microsoft DXC, but (on vulkan) you can use any SPIR-V blob by passing it as the first argument of the ```compushady.Compute``` initializer.

## Dealing with backends differences

### Bindings

In HLSL you assign resources to registers/slot using 3 main concepts: cbv, srv, uav and their related registers (b0, t0, t1, u0,...)

When converting HLSL to SPIR-V an offset is added to SRVs (1024) and UAVs (2048), so b0 will be mapped to SPIR-V binding 0, b1 to 1, t0 to 1024, t2 to 1026 and u0 to 2048. Remember this when using SPIR-V directly.

When converting HLSL to MSL the code is first translated to SPIR-V and finally remapped to MSL. Here the mapping is again different to overcome metal approach to resource binding. In metal we have only two (from the compushady point of view) kind of resources: buffers and textures. When doing the conversion the SPIR-V bindings are just trashed and instead a sequential model is applied: from the lowest binding id just assign the next index based on the type of SPIR-V resource (Uniform is a buffer, ConstantUniform is a texture).
Example: 2048 uniform will be buffer(1), 2049 contant uniform will be texture(0), 1025 uniform will be buffer(0). Remember the numerical order is always relevant!

## Known Issues

* The Metal backend does not support x and y for the Swapchain (so you will always blit the texture at 0, 0)
* There is some alignment issues on older Metal GPUs, generally the alignment of structured/formatted buffer should be handled in a more robust way (but Apple Silicon GPUs generally work without issues)
* Avoid using the d3d11 backend, it is slow in lot of operations and not thread safe
