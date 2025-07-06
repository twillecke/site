---
title: "An introduction to fragment shaders"
description: "What are they and how to implement them"
date: "Jul 07 2025"
draft: false
---
    "Colors are the deeds and sufferings of light - J.W. Goethe"

A common approach in game development is to implement spine animations, which, despite being hand-made and nice looking, can really increase the memory footprint. If the distribution platform is the web and there are specific performance requirements, we can optimize the build by replacing simpler spine animations with shaders. 

## What are shaders?
Shaders are programs that run on the GPU, modifying attributes of pixels - a pixel's properties are its coordinates and color expressed as a `vec4(r, g, b, a)` -  effectively changing how images are rendered on the screen:

- Turn a colored sprite to grayscale in a disabled state button? Shader!
- Color Gradient over text label? Shader!
- Light sweep over end-game winning values? Shader!
- Distort a texture as if looked through a magnifier? You know already...

Shaders are a whole universe by themselves. For that reason, the scope of this article will be **fragment shaders**, which compute the final color of each pixel, generating effects such as gradients, color manipulation, and post-processing.

In the modern era of game development, many engines offer graphical interfaces to create and manipulate shader nodes - Unity 3D and Unreal Engine have their own Shader Graph modules.

Alternatively, frameworks such as Three.js allow shaders to be developed for the web in the C-like **[OpenGL Shading Language](https://www.khronos.org/opengl/wiki/OpenGL_Shading_Language)** (GLSL). 

For the sake of simplicity, the examples provided in this article will use **[ShaderToy](https://www.shadertoy.com/)** as a playground.

## Color-to-grayscale shader
Let’s see what a color-to-grayscale shader looks like

```glsl
// in shadertoy mainImage() is the main function of a GLSL shader
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
	vec2 uv = (fragCoord.xy / iResolution.xy);
	vec3 gammaColor = texture(iChannel0, uv).xyz;
	vec3 color = pow(gammaColor, vec3(2.0));
	float gray = dot(color, vec3(0.2126, 0.7152, 0.0722));
	float gammaGray = sqrt(gray);
	fragColor = vec4(gammaGray, gammaGray, gammaGray, 1.0);
}
```
Shader by *aureliendrouet* available at https://www.shadertoy.com/view/4tlyDN

In the simplest terms this fragment shader processes every pixel on the screen in **parallel**. For each pixel, it uses its coordinates (`uv`) to find the original color from an input image (`iChannel0`). It then runs a specific calculation to convert that color into its **perceived brightness**, creating the correct shade of gray. This final gray value is sent to the output (`fragColor`), rendering the complete grayscale image.

### How pixels coordinates are mapped
At this point, it's important to understand how each pixel coordinate is mapped in the expression  `vec2 uv = (fragCoord.xy / iResolution.xy);` 

Breakdown:

- 1 `fragCoord.xy`
This variable holds the specific coordinates of the current pixel being processed. In a 1920x1080 resolution viewport, `fragCoord.xy` ranges from approximately `(0, 0)` for the bottom-left pixel to `(1920, 1080)` for the top-right pixel. It's a **`vec2`** containing the pixel's `x` and `y` position.

- 2 `iResolution.xy`
This variable holds the total resolution of the viewport. `iResolution.xy` is a **`vec2`** with the value `(1920.0, 1080.0)`.

- 3 `fragCoord.xy / iResolution.xy`
This expression divides the pixel's x-coordinate by the total width and the pixel's y-coordinate by the total height, effectively normalizing the pixel coordinates in a range from 0 to 1. 

- 4 `vec2 uv = (fragCoord.xy / iResolution.xy);`
The type of a pixel normalized coordinate is of a two-vector structure containing its `x` and `y` coordinates as `floats`, which can be accessed via dot notation (e.g. `uv.y` returns the `y` coordinate).  

## Two-color gradient shader
Let's now look at a basic color gradient shader:
```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord.xy / iResolution.xy;
    vec3 top = vec3(1.,1.,1.);
    vec3 bottom = vec3(0.,0.,0.);
    fragColor = vec4(mix(bottom, top, uv.y), 1.);
}
```

This shader renders a linear color gradient from white to black. As usual, it starts by normalizing pixel coordinates. Then with `mix(bottom, top, uv.y )` it blends the white and black colors. The third argument `uv.y` sets the blending factor according to pixel `y` coordinate. 

In this case, at `uv.y = 0.0` (bottom of the screen) color will be 100% black as our blending factor is 0. In the other way around, at `uv.y = 1.0` color will be 100% white. Anywhere in between, the color is a proportional mix of the two, creating a smooth gray gradient.

We know already that we can change the properties of a pixel according to its coordinate and original color. One cool thing about shaders is that we can also use `time` as a variable to create animations.

## Animating shaders as a function of time
Let's animate the previous gradient shader as an infinite scroll: 
```glsl
void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 uv = fragCoord.xy / iResolution.xy;
    vec3 top = vec3(1.,1.,1.);
    vec3 bottom = vec3(0.,0.,0.);
    float speed = 0.1; 
    float offset = mod(iTime * speed, 1.0);
    float gradientPos = mod(uv.y + offset, 1.0);
    vec3 color = mix(vec3(bottom), vec3(top), gradientPos);
    fragColor = vec4(color, 1.0);
}
```
Shader by *twillecke* (myself) available at https://www.shadertoy.com/view/WXt3WX

In ShaderToy, the global variable `iTime` holds the total elapsed time since the shader started running. Now, besides changing pixel colors according to their `y` position, we also add an offset factor that changes with time. Since our `offset` is wrapped in a range from [0, 1] by the modulo function `mod()`, the resulting animation is an infinite scroll.

From this simple example, you can see how we could use time to generate all sorts of procedural animations, color transitions, oscillations...

### Light sweep shader
Now let's code a light sweep shader with the following requirements: 
1. It must scroll infinitely over a texture:
	- Animate sweep position according to elapsed time. Make it loop infinitely with the modulo operator.
2. Have a semi-transparent color:
	- Add transparency by decreasing the alpha channel value in the sweep color.
3. Have a width:
	- In the same way a shader maps a normalized pixel coordinate by dividing its position by the screen resolution, we'll set a custom sweep width by dividing the desired width in pixels by the screen `x` resolution
4. Smoothly transition sweep from colored to fully transparent at its edges:
	- To generate a smooth transition at its edges, we'll calculate the sweep alpha value with a `smoothstep()` easing curve.
```glsl
void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 uv = fragCoord.xy / iResolution.xy;
    vec4 backgroundTexture = texture(iChannel0, uv);
    float sweepWidth = 200.0 / iResolution.x;
    float speed = 0.5;
    float totalSpan = 1.0 + sweepWidth * 2.0;
    float position = mod(iTime * speed, totalSpan) - sweepWidth;
    float alpha = smoothstep(position - sweepWidth, position, uv.x) *
                  (1.0 - smoothstep(position, position + sweepWidth, uv.x));
    alpha *= 0.5;
    vec3 sweepColor = vec3(1.0);
    vec3 finalColor = mix(backgroundTexture.rgb, sweepColor, alpha);  
    fragColor = vec4(finalColor, 1.0);
} 
```
Shader by *twillecke* available at https://www.shadertoy.com/view/tXd3Djs

Notice that we must subtract `sweepWidth` from the `position` to sweep start outside of the leftmost border. Also, take note of how `sweepWidth` is used to calculate the feathered sweep edges inside its width boundaries.

## Wrap-up
- Why use shaders?
	- They are lighter and much more performant than pre-rendered animations and effects.
	- They are not bound to specific sprites or spines, allowing for general use.
	- They can be dynamically controlled by developers through exposed parameters.
- What are the downsides?
	- Not suitable for complex, character-like animations.
	- GLSL is its programming language, requiring a kind of reasoning that is not common.
	- In web development, it requires attention to supported browsers.
	- Dynamic shaders can be tricky to control on sprite frames from atlases.

## Other references
- https://www.youtube.com/watch?v=f4s1h2YETNY
- https://thebookofshaders.com/04/
- https://nmattia.com/posts/2025-01-29-shader-css-properties/
