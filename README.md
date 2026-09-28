# Hydrogen Atom Orbital Visualizer

This is a little project of mine which creates a 3D visualization of a particular electron orbital of the hydrogen atom. This was mainly inspired by Kavan's atom simulation over at his [GitHub page](https://github.com/kavan010/Atoms).

## Build

Here's the handy-dandy command I use to build this with dynamic linking for GLFW.

```cmd
g++ main.cpp glad/src/glad.c -o main.exe -Iglad/include -IGLFW -L. -lglfw3dll -lopengl32 -lgdi32
```

If you build with this command, the built application file must be in the same directory as glfw3.dll. If you don't want dynamic linking and want to statically link instead, here's the command. The built application file produced by this command can be run standalone.

```cmd
g++ main.cpp glad/src/glad.c -o main.exe -Iglad/include -IGLFW -L. -lglfw3 -lopengl32 -lgdi32
```

NOTE: Both of these commands require the 64-bit version of g++. Without it, libglfw3.a and libglfw3dll.a will refuse to link during compilation.

## How it works

It's a C++ program which uses OpenGL to interface with the graphics card. Two libraries are used: GLFW and GLAD. GLFW does the window management while GLAD is an extension loader.

I've used four shaders, a generate shader, a raymarch shader, a vertex shader and a fragment shader. The vertex and fragment shader simply draw a texture onto a fullscreen triangle.

The generate shader calculates the probability density values and stores them in a 3D texture. This is accessed by the raymarch shader, which performs a raymarching algorithm I made and renders onto a 2D texture using the Inferno colormap. This is then displayed over the screen by the vertex and fragment shaders.

## Density drawing algorithm

Ideally, a density drawing algorithm should be bright where the density is high and dim where the density is low. If we imagine a cloud of gas with the same density as our probability density, then light passing through it would work oppositely to how we want it to, dimmer (more opaque) in denser regions and brighter (more transparent) in less dense regions. So one way we could implement density drawing is by figuring out an algorithm for drawing gases in front of a white backdrop and then flip it somehow.

When light passes through a gas, it hits many gas particles on the way. Every time it hits a gas particle, the light gets absorbed and re-emitted in all directions. Only a fraction of the light gets re-emitted in the same direction, so if we just imagine the part of the light's trajectory that ends up going in a straight line, every gas particle acts as a multiplier on the intensity of the light by some value between 0 and 1. However, this is just for the sake of modeling; real light rays behave in much more complicated ways.

We can approximate this behavior by simply running a ray through the gas and multiplying all the multipliers throughout. What's nice about this is that multiplication can be done in any order: coming towards the camera or going away from the camera. Since we only care about light rays ending up at the camera, we can simply shoot out rays from the camera and figure out the final intensity and display that.

Instead of multiplying by a factor less than 1 along the whole ray, we can simply subtract a decrement as we walk along the path and at the end, we can exponentiate, so the subtraction becomes multiplication by a factor less than 1. That means we simply have to integrate a quantity over the ray and then multiply by -1 and exponentiate. The result of this integral is what I call `intensityLog` in my code, however it's scaled differently due to optimization.

One quantity that we can integrate over to get nice results is the density function itself. If the ray passes through a region of high density, then the integral accumulates a large amount, so when we multiply by -1 and exponentiate, the intensity is close to 0. However, if the ray passes through a region of density close to 0, then the integral stays close to 0 and multiplying by -1 and exponentiate, the intensity is close to 1. And crucially, densities are non-negative, so the integral is always non-negative and the intensity is always between 0 and 1.

Now that we have our intensity for light passing through a gas, to get the intensity of light to display high density as bright and low density as dim, we simply have to subtract this intensity from 1. That's how the intensity is calculated in the raymarch shader.

However, we can also multiply our integral by a positive constant before multiplying by -1 and exponentiating, and this leaves our relationship of dim and bright unchanged, but changes how bright the density looks.

If we consider our probability density function to be $P(x,y,z)$ then we can define

$$ I(R) = \int_R P(x,y,z) ds $$

$I(R)$ is our accumulated integral, where $R$ is the ray, and the notation $\int_R ds$ denotes the line integral over the ray $R$. We can write down our intensity (or brightness) as

$$ B(R) = 1 - e^{-\lambda I(R)} $$

where $\lambda > 0$ is our constant multiplier, which controls how bright it looks.

However, just seeing a grayscale image isn't all that fun, and I wanted to capture that distinctive look of the pictures of the hydrogen atom orbitals I'd seen on the internet, so I used a colormap called Inferno to turn that bland, dry intensity into a colorful visualization.

## Technical details

This project uses OpenGL along with GLFW and GLAD (although I haven't used any extensions). A standard raymarching algorithm would take place in the fragment shader, but instead I created a texture and filled it with a compute shader. The fragment shader simply draws that texture over the screen.

Earlier, my compute shader was making a lot of lag, mostly due to the Inferno colormap being in an array as opposed to a texture, but also partially due to the orbital probability density being recalculated for every pixel, every frame, even though it was constant throughout. So I split my compute shader into two compute shaders, one for rendering onto a texture, which runs every frame, and another for calculating the probability density values and storing them in a 3D texture, which runs only once at the beginning.

Since the raymarch shader actually only draws to a fixed size texture, there can be some pixelation or visual artifacts when resizing the window. Setting a sufficiently high resolution for the texture should resolve the issue, although it could cause some lag.

## Mathematical details

The actual details of solving the Schrödinger equation for the hydrogen atom and obtaining the wavefunction and subsequently the probability density are immensely complicated, and are included in PDF files for better reading.

## Dependencies

This project uses OpenGL 4.6, so your graphics driver and graphics card have to support that for this to run. The program actually outputs your OpenGL version when it runs, in the console output, so you can check that there. If you do not have OpenGL 4.6, you can still rebuild the project with OpenGL version 4.3 or later, for compute shaders to work, but you will have to reconfigure GLAD to use that version instead, and in the code, change the preprocessor macros `OPENGL_VERSION_MAJOR` and `OPENGL_VERSION_MINOR` at the top of the code to your particular version.

This project also depends on GLFW 3.5.1, GLAD 1.0 and the Inferno colormap provided by [zaman13](https://github.com/zaman13) on GitHub. These are distributed with the project source code though, so there's no need to install them manually.