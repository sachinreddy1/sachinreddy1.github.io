## Background
After reading through [Zuofu Cheng’s Dissertation](https://www.ideals.illinois.edu/bitstream/handle/2142/50364/Zuofu_Cheng.pdf?sequence=1) on Real-Time Geometric Acoustics in Interactive Applications, I was inspired to apply some of his concepts into a Minecraft Mod. Seeing that Zuofu has had incredible success in implementing Geometric Acoustics into [Quake 3](https://www.youtube.com/watch?v=TXUTgEmnD6U), I felt Minecraft would be a great application to apply these sound concepts to. 

## Goals/Functionalities
- Simple Reverberation
- Raytracing Environment
- Real-time Reverberation
- Real-time Occlusion
- Visualizing Raytraces in GUI

### Simple Reverberation
To begin, I needed to access Minecraft's OpenAL sound engine and add filters to all the sound sources to give a reverberation effect. I used OpenAL's [Effects Extension Guide](https://kcat.strangesoft.net/misc-downloads/Effects%20Extension%20Guide.pdf), which served in-valuable in implementing the various featured needed in the mod. 

The first major step was to create auxiliary effect slots which are  containers for DSP Effects. Four effect slots are created with the following:
```java
// Creating Effect Slot
int auxFXSlot0 = EFX10.alGenAuxiliaryEffectSlots();
EFX10.alAuxiliaryEffectSloti(auxFXSlot0, EFX10.AL_EFFECTSLOT_AUXILIARY_SEND_AUTO, AL10.AL_TRUE);
```

After this, effect objects need to be created in order to bind to these effect slots. This is done with the following:
```java
// Creating Effect Object
int reverb0 = EFX10.alGenEffects();
EFX10.alEffecti(reverb0, EFX10.AL_EFFECT_TYPE, EFX10.AL_EFFECT_EAXREVERB);
```
There are two reverb effect types in this SDK: AL_EFFECT_REVERB and AL_EFFECT_EAXREVERB. I decided to use AL_EFFECT_EAXREVERB since it has a more advanced parameter set. 

After adding the code, the next step was to create filters 
```java
// Creating Filter
int sendFilter0 = EFX10.alGenFilters();
EFX10.alFilteri(sendFilter0, EFX10.AL_FILTER_TYPE, EFX10.AL_FILTER_LOWPASS);
```
After generating a Filter object the application can set the values for each of the parameters using the alFilter(i,iv,f,fv) function call, which will be used when calculating the environment.

To then set the reverberation parameters, I can simply do:
```java
// Set reverberation parameters
EFX10.alEffectf(reverbSlot, EFX10.AL_EAXREVERB_DENSITY, r.density);		
EFX10.alEffectf(reverbSlot, EFX10.AL_EAXREVERB_DIFFUSION, r.diffusion);
EFX10.alEffectf(reverbSlot, EFX10.AL_EAXREVERB_GAIN, r.gain);
EFX10.alEffectf(reverbSlot, EFX10.AL_EAXREVERB_GAINHF, r.gainHF);
...
```
I created a reverberation class to easily add these values for density, diffusion, etc. Each of which ranges from 0 to 10. 

And lastly, I had to configure the filter for each slot that was created with:
```java
EFX10.alFilterf(sendFilter0, EFX10.AL_LOWPASS_GAIN, sendGain0);
EFX10.alFilterf(sendFilter0, EFX10.AL_LOWPASS_GAINHF, sendCutoff0);
AL11.alSource3i(sourceID, EFX10.AL_AUXILIARY_SEND_FILTER, auxFXSlot0, 0, sendFilter0);
```
To test the reverberation, I simply set sendGain0 to 0.0 and sendCutoff0 to 1.0. This gave a large and pronounced reverberation for each sound source in the game. 

### Raytracing Environment

The next hard step was to use raytracing to calculate the environment, so that accurate values could be used for the Gain and Cutoff values. This would allow for more accurate reverberation. For example, if a player is in an open field, they should not hear large reverberation due to a lack of reflected sound waves. On the other hand, if a player is in a cave, reverberation should be pronounced considering the possibility of echoing.

To keep things simple in the explanation, the way this is done is by having a set number of rays that are shot out of each sound source at various angles. These rays are then checked to see if they hit any blocks. If so, a secondary bounce is calculated at the normal angle the ray hit the initial block. This can then be used to calculate the total distance the ray traveled, and different blocks the ray hit along it path. To give a better idea, here is simplied version of the code:

```java
// Getting player and sound positions
Vec3d soundPos = new Vec3d(posX, posY, posZ);
Vec3d playerPos = minecraft.thePlayer.getPositionVector();

final float phi = 1.618033988f;
final float gAngle = phi * (float)Math.PI * 2.0f;
final float maxDistance = 256.0f;

final int numRays = 30;
final int rayBounces = 4;

for (int i = 0; i < numRays; i++)
{
	float fi = (float)i;
	float fiN = (float)fi / (float)numRays;
	float longitude = gAngle * fi * 1.0f;
	float latitude = (float)Math.asin(fiN * 2.0f - 1.0f);
	
	Vec3d rayDir = new Vec3d(0.0, 0.0, 0.0);
	{
		double x = Math.cos(latitude) * Math.cos(longitude);
		double y = Math.cos(latitude) * Math.sin(longitude);
		double z = Math.sin(latitude);
		rayDir = new Vec3d(x, y, z);
	}
	
	Vec3d rayStart = new Vec3d(soundPos.xCoord, soundPos.yCoord, soundPos.zCoord);
	Vec3d rayEnd = new Vec3d(rayStart.xCoord + rayDir.xCoord * maxDistance, rayStart.yCoord + rayDir.yCoord * maxDistance, rayStart.zCoord + rayDir.zCoord * maxDistance);
	RayTraceResult rayHit = minecraft.theWorld.rayTraceBlocks(rayStart, rayEnd, true);
	
	if (rayHit != null)
	{
	    double rayLength = soundPos.distanceTo(rayHit.hitVec);
				
		// Additional bounces
		Int3 lastHitBlock = Int3.create(rayHit.getBlockPos().getX(), rayHit.getBlockPos().getY(), rayHit.getBlockPos().getZ());
		Vec3d lastHitPos = rayHit.hitVec;
		// For reflecting
		Vec3d lastHitNormal = getNormalFromFacing(rayHit.sideHit);
		Vec3d lastRayDir = rayDir;
				
		float totalRayDistance = (float)rayLength;
				
		//Secondary ray bounces
		for (int j = 0; j < rayBounces; j++)
		{
			Vec3d newRayDir = reflect(lastRayDir, lastHitNormal);
			Vec3d newRayStart = new Vec3d(lastHitPos.xCoord + lastHitNormal.xCoord * 0.01, lastHitPos.yCoord + lastHitNormal.yCoord * 0.01, lastHitPos.zCoord + lastHitNormal.zCoord * 0.01);
			Vec3d newRayEnd = new Vec3d(newRayStart.xCoord + newRayDir.xCoord * maxDistance, newRayStart.yCoord + newRayDir.yCoord * maxDistance, newRayStart.zCoord + newRayDir.zCoord * maxDistance);					
			RayTraceResult newRayHit = minecraft.theWorld.rayTraceBlocks(newRayStart, newRayEnd, true);
												
			if (newRayHit != null)
			{	
				double newRayLength = lastHitPos.distanceTo(newRayHit.hitVec);
				totalRayDistance += newRayLength;
				
				lastHitPos = newRayHit.hitVec;
				lastHitNormal = getNormalFromFacing(newRayHit.sideHit);
			}
			else
				totalRayDistance += lastHitPos.distanceTo(playerPos);
					
			if (newRayHit == null)
				break;
		}
	}
}
```

And with this code, I can then add various calculations to modify the Gain and Cutoff values that will be used for modifying the filters!

### Real-time Reverberation
Now that I am able to calculate the environment using raytracing. I can then use the calculated values to modify the Gain and Cutoff values. With reverberation, I mainly need to modify the Gain.

In the raytracing of the environement, I calculate the energy towards the player with the block reflectivity. Block reflectivity is a preset value that I estimated at the start. For example:

```java
if (soundType == SoundType.STONE)
	reflectivity = 1.0;
else if (soundType == SoundType.WOOD)
	reflectivity = 0.5;
...
```

Various calculation were added to ensure a good sound for reverberation. If you want to look into all the calculations done, check out the code on my [Github](https://github.com/sachinreddy1/Geometric_Acoustics/blob/master/src/main/java/com/sachinreddy/GeometricAcoustics/GeometricAcoustics.java).

### Visualizing Raytraces in GUI
Aside from getting good sounding reverberation in the mod. I felt it necessary to add an overlay GUI to the game so that users could visualize the sound source's raytraces in their given environment. 

Simple rendering of textures was done by adding sprites to the resources folder and calling the following code:

```java
ResourceLocation histogramBlock = new ResourceLocation(GeometricAcousticsCore.modid, "textures/gui/histogram.png");

GL11.glPushMatrix();
{
    GL11.glColor3f(r, g, b);
    mc.renderEngine.bindTexture(histogramBlock);
    drawTexturedModalRect(0, 0, 0, 0, 10, 10);
}
GL11.glPopMatrix();
```
GL11 can be used to transform the texture in many different ways. In the above example, I am able to modify the color of the texture any given RGB value.

Users will press F6 see the GUI. An example of the finished overlay is pictured here:

![GUI Overlay](https://s3.amazonaws.com/sachinreddy.com/GeometricAcoustics/GuiOverlay.png "GUI Overlay")

### Conclusion
Overall I really enjoyed this project, since it applied various mathematical modeling and game engine techniques to get everything to work. I plan on updating the code considering that new version of Minecraft continue to be released and my code only supports version 1.11.

[Download (Version 1.11)](GeometricAcoustics-1.11.jar)
[Link to Github Repo](https://github.com/sachinreddy1/Geometric_Acoustics)