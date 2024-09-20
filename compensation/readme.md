## NAME 

compensation.py 

## SYNOPSIS 

**loadusr -Wn compensation** python compensation.py probe-results.txt *cubic*

## DEPENDENCIES 

The component has Python 2.7 or Python 3 dependencies of NumPy, SciPy and Enum34.

External Offsets are supported in LinuxCNC 2.8.0 and newer.

## DESCRIPTION 

**compensation.py** is a bed levelling / distortion correction user space compensation component for 3D Printing or engraving.

The component utilises the [external offsets](https://linuxcnc.org/docs/stable/html/motion/external-offsets.html) feature of LinuxCNC, which allows an axis to be offset by an external data source (in this case a Z compensation map). External Offsets (and hence Z compensation) is active during both world jogs and coordinated motion, offsetting the Z axis by the value of the map at that location.

To generate a Z compensation map, a probe data input file is specified in the *loadusr* string. An optional argument for the interpolation method can be either *nearest*, *linear* or *cubic*. If not specified the default method is cubic *(bicubic)* interpolation. Interpolation resolution and fade height are set according to HAL pins created by the component.

The probe data input file must have data on a regularly spaced grid. The file can be modified / updated at anytime while compensation is disabled (Don't probe whilst compensation is enabled). When compensation is next enabled, the compensation map will be recalculated.

A suitable probing program to create such a grid is [gridprobe.ngc](https://github.com/LinuxCNC/linuxcnc/blob/master/nc_files/gridprobe.ngc) (or [smartprobe.ngc](https://github.com/LinuxCNC/linuxcnc/blob/master/nc_files/smartprobe.ngc) if modified slightly to output data in the same format as *gridprobe.ngc*). A modified version, *smartprobe_compensation.ngc*, with these (and some further modifications) is in this repository.

The component uses the SciPy [griddata](https://docs.scipy.org/doc/scipy/reference/generated/scipy.interpolate.griddata.html) function to interpolate between the probe data points at the specified resolution in X and Y.

Moves outside of the probe data region receive the compensation value at the edge of the compensation map so as not to have a jump / discontinuity, and hence the compensated value extends to the machine coordinate limits if the compensation map is non-zero at its edge.

The ability of the machine to maintain the compensated Z value or to conform to a probed surface depends on the proportion of the Z axis acceleration/velocity allocated to External Offsets vs. the acceleration/velocity of the other axes, the steepness of the surface gradient to follow, the probe grid resolution, interpolation resolution, and potentially the update rate of the component (which is every 0.05 seconds by default).

![Compensation map](compensationMap.png)

## NAMING

The names for pins, parameters, and functions are prefixed as:  
**compensation**.name


## FUNCTIONS 

There are no discrete functions to call. Once loaded the component runs in user space in parallel with LinuxCNC.

## PINS 

| Pin | Type | Direction | Funtion|
| :--- | :--- | :---: | :--- |
| compensation.enable-in | bit | in | enables Z compensation |
| compensation.enable-out | bit | out | enables External offset |
| compensation.scale | float | out | scale value for External offset |
| compensation.count | s32 | out | count value for External offset |
| compensation.clear | bit | out | clear External offset |
| compensation.x-pos | float | in | X position command |
| compensation.y-pos | float | in | Y position command |
| compensation.z-pos | float | in | Z position command |
| compensation.fade-height  | float | in | compensation will be faded out up to this value |
| compensation.resolution  | float | in | the resolution of the interpolation |

eoffset == scale * count

## INSTALLATION

For the component to function, External Offsets must be enabled for the Z axis and the component linked to the External Offsets pins:


### CONFIG MODIFICATIONS

Place the compensation.py in your configuration ini directory.


### INI FILE MODIFICATIONS

Add the following lines to enable [HALUI](https://linuxcnc.org/docs/stable/html/gui/halui.html) to make the relative xyz position HAL pins available:

	[HAL]  
	HALUI = halui

And add the following line to add an OFFSET_AV_RATIO parameter:

	[AXIS_Z]  
	OFFSET_AV_RATIO = 0.2

This may be a value between 0 (External Offsets disabled) and 0.9, where the value is the proportion of the axis max velocity and acceleration allocated to External Offsets. For more information, see the [documentation](https://linuxcnc.org/docs/stable/html/motion/external-offsets.html#_ini_file_settings).


### HAL FILE MODIFICATIONS

Add the following lines:

	loadusr -Wn compensation python compensation.py probe-results.txt cubic

	net eoffset-xpos 	<= halui.axis.x.pos-relative	=> compensation.x-pos
	net eoffset-ypos	<= halui.axis.y.pos-relative	=> compensation.y-pos
	net eoffset-zpos	<= halui.axis.z.pos-relative	=> compensation.z-pos
	net eoffset-enable	<= compensation.enable-out	=> axis.z.eoffset-enable
	net eoffset-scale	<= compensation.scale		=> axis.z.eoffset-scale
	net eoffset-counts	<= compensation.counts 		=> axis.z.eoffset-counts
	net eoffset-clear	<= compensation.clear 		=> axis.z.eoffset-clear
	net compensation-on	<= compensation.enable-in

To enable/disable compensation by setting the signal directly: 

	sets compensation-on 1
 	sets compensation-on 0
 Or connect the signal to an appropriate GUI button or M code so it can be easily toggled.
