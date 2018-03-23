# TouchDesigner Deferred Lighting - Cone Lights
TouchDesigner networks are notoriously difficult to read, and this doc is intended to help shed some light on the ideas explored in this initial sample tox that's largely flat.

This approach is very similar to point lights, with the additional challenge of needing to think about lights as directional. We'll see that the first stage and last of this process - is consistent with our Point Light example, but in the middle we need to make some changes. We can get started by again with color buffers.

## Color Buffers
![color buffers]()

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
![light attributes]()

Here we'll begin to see a divergence from our previous approach.

We are still going to compute and pack data for the position, color, and falloff for our point lights like in our previous example. The difference now is that we also need to compute a look-at position for each of our lights. In addition to our falloff data we'll need to also consider the cone angle and delta of our lights. For the time being cone angle is working, but cone delta is broken - pardon my learning in public here. 

For the sake of sanity / simplicity we'll use a piece of geometry to represent the position of our point lights - similar to the approach used for instancing. In our network we can see that this is represented by our null SOP `null_lightpos`. We convert this to CHOP data and use the attributes from this null (number of points) to correctly ensure that the rest of our CHOP data matches the correct number of samples / lights we have in our scene. In this case we're using a null since we want to position the look-at points at some other position than our lights themselves. Notice that our circle has one transform SOP to describe light position, and another transform SOP to describe look-at position. In the next stage we'll use our `null_light_pos` CHOP and our `null_light_lookat` CHOP for the lighting calculations - we'll also end up using the results of our object CHOP `null_cone_rot` to be able to describe the rotation of our lights when rendering them as instances. 

When it comes to the color of our lights, we can use a noise or ramp TOP to get us started. These values are ultimately just CHOP data, but it's easier to think of them in a visual way - hence the use of a ramp or noise TOP. The attributes for our lights are packed into CHOPs where each sample represents the attributes for a different light. We'll use a `texelFetchBuffer()` call in our next stage to pull the information we need from these arrays. Just to be clear, our attributes are packed in the following CHOPs:
* position - `null_light_pos`
* color - `null_light_color`
* falloff - `null_light_falloff`
* light cone - `null_light_cone`

This means that sample 0 from each of these four CHOPs all relate to the same light. We pack them in sequences of three channels, since that easily translates to a `vec3` in our next fragment process. 

The additional light cone attribute here is used to describe the radius of the cone and the degree of softness at the edges (again pardon the fact that this isn't yet working).

## Combining Buffers
![combining buffers]()

Next up we combine our color buffers along with our CHOPs that hold the information about our lights location and properties.

What does this mean exactly? It's here that we loop through each light to determine its contribution to the lighting in the scene, accumulate that value, and combine it with what's in our scene already. This assemblage of our stages and lights is "deferred" so we're only doing this set of calculations based on the actual output pixels, rather than on geometry that may or may not be visible to our camera. For loops are generally frowned on in openGL, but this is a case where we can use one to our advantage and with less overhead than if we were using light components for our scene.

Here's a look at the GLSL that's used to combine our various buffers:

```glsl
\\TouchDesigner glsl TOP

uniform int numLights;

uniform vec3 viewDirection;             // camera location
uniform vec3 specularColor;

uniform float shininess;

uniform sampler2D samplerPos;
uniform sampler2D samplerNorm;
uniform sampler2D samplerColor;
uniform sampler2D samplerUv;

uniform samplerBuffer lightPos;         // position as xyz
uniform samplerBuffer lightColor;       // color as rgb
uniform samplerBuffer lightFalloff;     // falloff constant, linear, quadratic
uniform samplerBuffer lightLookat;      // lookat position xyz
uniform samplerBuffer lightCone;        // angle, delta, and falloff

out vec4 fragColor;

#define PI 3.14159265359

void main()
{
    vec2 screenSpaceUV      = vUV.st;
    vec2 resolution         = uTD2DInfos[0].res.zw;

    // parse data from g-buffer
    vec3 position       = texture( sTD2DInputs[0], screenSpaceUV ).xyz;
    vec3 normal         = texture( sTD2DInputs[1], screenSpaceUV ).xyz;
    vec4 color          = texture( sTD2DInputs[2], screenSpaceUV );
    vec2 uv             = texture( sTD2DInputs[3], screenSpaceUV ).rg;

    vec3 cameraVec      = normalize(viewDirection - position);

    // set up placeholder for final color
    vec3 finalColor     = vec3(0.0);

    // loop through all lights
    for ( int light = 0; light < numLights; ++light ){

        // parse lighitng data based on the current light index
        vec3 currentLightPos        = texelFetchBuffer( lightPos, light ).xyz;
        vec3 currentLightColor      = texelFetchBuffer( lightColor, light ).xyz;
        vec3 currentLightFalloff    = texelFetchBuffer( lightFalloff, light ).xyz;
        vec3 currentLightLookat     = texelFetchBuffer( lightLookat, light ).xyz;
        vec3 currentLightCone       = texelFetchBuffer( lightCone, light ).xyz;

        // cone attributes
        float uConeAngle            = currentLightCone.x;
        float uConeDelta            = currentLightCone.y;
        float uConeRolloff          = currentLightCone.z;   

        // calculate the distance between the current fragment and the light source
        float lightDist             = length( currentLightPos - position );
        vec3 lightVec               = normalize( currentLightPos - position );

        // spot
        float fullcos               = cos(radians((uConeAngle / 2.0) + uConeDelta));
        fullcos                     = (fullcos * 0.5) + 0.5;
        float scale                 = 0.5 / (1.0 - fullcos);
        float bias                  = (0.5 - fullcos) / (1.0 - fullcos);
        vec2 coneLookupScaleBias    = vec2(scale, bias);
        
        vec3 spot                   = normalize( currentLightLookat - currentLightPos );
        float spotEffect            = dot( spot, -lightVec );
        float coneAngle             = radians( uConeAngle / 2.0);
        float ang                   = acos(spotEffect);

        float dimmer;
        if( ang > coneAngle )
            dimmer                  = 0.0;
        else
            dimmer                  = texture( sTDSineLookup, 1.0 - ( ang / coneAngle ) ).r;


        vec3 toLight                = normalize( currentLightPos - position );
        vec3 diffuse                = max( dot( normal, toLight ), 0.0 ) * color.rgb * currentLightColor;

        float diffuseDot            = clamp(dot(lightVec, normal), 0.0, 1.0);
        vec3 colorSum               = diffuseDot * currentLightColor;
        vec3 halfAng                = normalize((cameraVec + lightVec).xyz);
        float specDot               = pow(clamp(dot(halfAng, normal), 0.0, 1.0), shininess);

        colorSum                    += specDot * currentLightColor * 0.3 * diffuse;
        colorSum                    *= dimmer;


        // accumulate lighting
        finalColor                  += colorSum;

    }

    // final color out 
    fragColor = TDOutputSwizzle( vec4( finalColor, color.a ) );
}
```
If you look at the final pieces of our for loop you'll find that much of this process is borrowed from the example Malcolm wrote (Thanks Malcolm!). This starting point serves as a baseline to help us get started from the position of how other lights are handled in Touch.

## Representing Lights
![representing lights]()

At this point we've successfully completed our lighting calculations, had them accumulate in our scene, and have a slick looking render. However, we probably want to see them represented in some way. In this case we might want to see them just so we can get a sense of if our calculations and data packing is working correctly. 

To this end, we can use instances and a render pass to represent our lights as spheres to help get a more accurate sense of where each light is located in our scene. If you've used instances before in TouchDesigner this should look very familiar. If that's new to you, check out: [Simple Instancing](https://matthewragan.com/2015/03/31/thp-494-598-simple-instancing-touchdesigner/)

Our divergence here is that rather than using spheres, we're instead using cones to represent our lights. In a future iteration the width of the cone base should scale along with our cone angle, but for now let's celebrate the fact that we have a way to see where our lights are coming from. You'll notice that the rotate attributes generated from the object CHOP are used to describe the rotation of the instances. Ultimately, we probably don't need these representations, but they sure are handy when we're trying to get a sense of what's happening inside of our shader.

## Post Processing for Final Output
![post process]()

Finally we need to assemble our scene and do any final post process bits to get make things clean and tidy.

Up to this point we haven't done any anti-aliasing, and our instances are in another render pass. To combine all of our pieces, and do take off the sharp edges we need to do a few final pieces of work. First we'll composite our scene elements, then do an anti-aliasing pass. This is also where you might choose to do any other post process treatments like adding a glow or bloom to your render.

![final product]()