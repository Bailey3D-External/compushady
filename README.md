# compushady
Python module for easily running Compute Shaders

![compushadyMandlebrot](compushady_mandlebrot.gif?raw=true "compushady Mandlebrot")

Currently d3d12 (Windows), vulkan (Linux x86-64 and Windows) and d3d11 (Windows) are supported. 

You can write shaders in HLSL, they will be compiled in the appropriate format (DXIL, DXBC, SPIR-V, ...) automatically.

The Mandlebrot anim gif you are seeing above has been generated by compushady itself: https://github.com/rdeioris/compushady/blob/main/examples/mandlebrot_gif.py

OpenGL, Metal and GLSL backends are expected to be released in the future.

## Quickstart

```sh
pip install compushady
```

(if you are building from sources, be sure vulkan and x11 headers are installed, ```libvulkan-dev``` and ```libx11-dev``` on a debian based distribution)

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

## compushady.Texture2D

## compushady.Texture1D and compushady.Texture3D

## compushady.Compute

To build a Compute object (the one running the compute shader), you need (obviously) a shader blob (you can build it using the ```compushady.shaders.hlsl.compile``` function) and the resources (buffers and textures) you want to manage in the shader itself.

compushady uses the DirectX12 naming: CBV (Constant Buffer View) for constant buffers (generally little ammount of data that do not change during the compute shader execution), SRV (Shader Resource View) for buffer and textures you need to read in the shader, and UAV (Unordered Access View) for buffers and textures that need to be written by the shader.

## compushady.Swapchain

While very probably you are going to run compushady in a headless environment, the module exposes a Swapchain object for blitting your textures on a window.
For creating a swapchain you need to specify a window handle to attach to (this is operating system dependent), a format (generally B8G8R8A8_UNORM) and the number of buffers (generally 2).

This is an example of doing it using the glfw module (with a swpachain with 3 buffers):

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
