# TouchDesigner Deferred Lighting - Point Lights
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
![light attributes](https://github.com/raganmd/touchdesigner-deferred-lighting/blob/master/repo-assets/readme-screenshots/example-lights-cone-attributes.PNG?raw=true)

Next we're going to compute and pack data for the position, color, and falloff for our point lights.

For the sake of sanity / simplicity we'll use a piece of geometry to represent the position of our point lights - similar to the approach used for instancing. In our network we can see that this is represented by our Circle SOP `circle1`. We convert this CHOP data and use the attributes from this circle (number of points) to correctly ensure that the rest of our CHOP data matches the correct number of samples / lights we have in our scene.

When it comes to the color of our lights, we can use a noise or ramp TOP to get us started. These values are ultimately just CHOP data, but it's easier to think of them in a visual way - hence the use of a ramp or noise TOP. The attributes for our lights are packed into CHOPs where each sample represents the attributes for a different light. We'll use a `texelFetchBuffer()` call in our next stage to pull the information we need from these arrays. Just to be clear, our attributes are packed in the following CHOPs:
* position - `null_light_pos`
* color - `null_light_color`
* falloff - `null_light_falloff`

This means that sample 0 from each of these three CHOPs all relate to the same light. We pack them in sequences of three channels, since that easily translates to a `vec3` in our next fragment process. 

## Combining Buffers
![combining buffers](https://github.com/raganmd/touchdesigner-deferred-lighting/blob/master/repo-assets/readme-screenshots/example-lights-cone-combining-buffers.PNG?raw=true)

Next up we combine our color buffers along with our CHOPs that hold the information about our lights location and properties.

What does this mean exactly? It's here that we loop through each light to determine its contribution to the lighting in the scene, accumulate that value, and combine it with what's in our scene already. This assemblage of our stages and lights is "deferred" so we're only doing this set of calculations based on the actual output pixels, rather than on geometry that may or may not be visible to our camera. For loops are generally frowned on in openGL, but this is a case where we can use one to our advantage and with less overhead than if we were using light components for our scene.

Here's a look at the GLSL that's used to combine our various buffers:

```glsl
\\TouchDesigner glsl TOP

uniform int numLights;

uniform vec3 viewDirection;
uniform vec3 specularColor;

uniform float shininess;

uniform sampler2D samplerPos;
uniform sampler2D samplerNorm;
uniform sampler2D samplerColor;
uniform sampler2D samplerUv;

uniform samplerBuffer lightPos;
uniform samplerBuffer lightColor;
uniform samplerBuffer lightFalloff;

out vec4 fragColor;

void main()
{
    vec2 screenSpaceUV      = vUV.st;
    vec2 resolution         = uTD2DInfos[0].res.zw;

    // parse data from g-buffer
    vec3 position       = texture( sTD2DInputs[0], screenSpaceUV ).rgb;
    vec3 normal         = texture( sTD2DInputs[1], screenSpaceUV ).rgb;
    vec3 color          = texture( sTD2DInputs[2], screenSpaceUV ).rgb;
    vec2 uv             = texture( sTD2DInputs[3], screenSpaceUV ).rg;

    // set up placeholder for final color
    vec3 finalColor     = vec3(0.0);

    // loop through all lights
    for ( int light = 0; light < numLights; ++light ){

        // parse lighitng data based on the current light index
        vec3 currentLightPos        = texelFetchBuffer( lightPos, light ).xyz;
        vec3 currentLightColor      = texelFetchBuffer( lightColor, light ).xyz;
        vec3 currentLightFalloff    = texelFetchBuffer( lightFalloff, light ).xyz;

        // calculate the distance between the current fragment and the light source
        float lightDist             = length( currentLightPos - position );

        // diffuse contrabution
        vec3 toLight                = normalize( currentLightPos - position );
        vec3 diffuse                = max( dot( normal, toLight ), 0.0 ) * color * currentLightColor;

        // specular contrabution
        vec3 toViewer               = normalize( position - viewDirection );
        vec3 h                      = normalize( toLight - toViewer );
        float spec                  = pow( max( dot( normal, h ), 0.0 ), shininess );
        vec3 specular               = currentLightColor * spec * specularColor;

        // attenuation
        float attenuation           = 1.0 / ( 1.0, currentLightFalloff.y * lightDist + currentLightFalloff.z * lightDist * lightDist );
        diffuse                     *= attenuation;
        specular                    *= attenuation;

        // accumulate lighting
        finalColor                  += diffuse + specular;

    }

    // final color out 
    fragColor = TDOutputSwizzle( vec4( finalColor, 1.0 ) );
}

```

## Representing Lights
![representing lights](https://github.com/raganmd/touchdesigner-deferred-lighting/blob/master/repo-assets/readme-screenshots/example-lights-cone-represetning-lights.PNG?raw=true)

At this point we've successfully completed our lighting calculations, had them accumulate in our scene, and have a slick looking render. However, we probably want to see them represented in some way. In this case we might want to see them just so we can get a sense of if our calculations and data packing is working correctly. 

To this end, we can use instances and a render pass to represent our lights as spheres to help get a more accurate sense of where each light is located in our scene. If you've used instances before in TouchDesigner this should look very familiar. If that's new to you, check out: [Simple Instancing](https://matthewragan.com/2015/03/31/thp-494-598-simple-instancing-touchdesigner/)

## Post Processing for Final Output
![post process](https://github.com/raganmd/touchdesigner-deferred-lighting/blob/master/repo-assets/readme-screenshots/example-lights-cone-post-process.PNG?raw=true)

Finally we need to assemble our scene and do any final post process bits to get make things clean and tidy.

Up to this point we haven't done any anti-aliasing, and our instances are in another render pass. To combine all of our pieces, and do take off the sharp edges we need to do a few final pieces of work. First we'll composite our scene elements, then do an anti-aliasing pass. This is also where you might choose to do any other post process treatments like adding a glow or bloom to your render.

![final product](https://github.com/raganmd/touchdesigner-deferred-lighting/blob/master/repo-assets/readme-screenshots/piont-lights.gif?raw=true)