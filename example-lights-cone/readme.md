# TouchDesigner Deferred Lighting - Cone Lights
TouchDesigner networks are notoriously difficult to read, and this doc is intended to help shed some light on the ideas explored in this initial sample tox that's largely flat. 

## Color Buffers
![color buffers](https://github.com/raganmd/touchdesigner-deferred-lighting/blob/master/repo-assets/readme-screenshots/example-lights-cone-color-buffers.PNG?raw=true)

These four color buffers represent all of that information that we need in order to do our lighting calculations further down the line. At this point we haven't done the lighting calculations yet - just set up all of the requisite data so we can compute our lighting in another pass.

Our four buffers represent:
* position - `renderselect_postition`
* normals - `renderselect_normal`
* color - `renderselect_color`
* uvs - `renderselect_uv`

If we look at our GLSL material we can get a better sense of how that's accomplished.

```glsl
// TouchDesigner vertex shader

// struct and data to fragment shader
out VS_OUT{
    vec4 position;
    vec3 normal;
    vec4 color;
    vec2 uv;
    } vs_out;

void main(){
    
    // packing data for passthrough to fragment shader
    vs_out.position     = TDDeform(P);
    vs_out.normal       = TDDeformNorm(N);
    vs_out.color        = Cd;
    vs_out.uv           = uv[0].st;

    gl_Position         = TDWorldToProj(vs_out.position); 
}
```

```glsl
// TouchDesigner frag shader

// struct and data from our vertex shader
in VS_OUT{
    vec4 position;
    vec3 normal;
    vec4 color;
    vec2 uv;
    } fs_in;

// color buffer assignments
layout (location = 0) out vec4 o_position;
layout (location = 1) out vec4 o_normal;
layout (location = 2) out vec4 o_color;
layout (location = 3) out vec4 o_uv;

void main(){
    o_position  = fs_in.position;
    o_normal    = vec4( fs_in.normal, 1.0 );
    o_color     = fs_in.color;
    o_uv        = vec4( fs_in.uv, 0.0, 1.0 );
}
```

Essentially, the idea here is that we're encoding information about our scene in color buffers for later combination. In order to properly do this in our scene we need to know point position, normal, color, and uv. This is normally handled without any additional intervention by the programmer, but in the case of working with lots of lights we need to organize our data a little differently. 

## Light Attributes
Next we're going to compute and pack data for the position, color, and falloff for our point lights.

For the sake of sanity / simplicity we'll use a piece of geometry to represent the position of our point lights - similar to the approach used for instancing.

## Combining Buffers
Next up we combine our color buffers along with our CHOPs that hold the information about our lights location and properites.

## Represeting Lights
We can use instances and a render pass to represent our lights as spheres to help get a more accurate sense of where each light is located in our scene.

## Post Processing for Final Output
Finally we need to asemble our scene and do any final post process bits to get make things clean and tidy.