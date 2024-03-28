# Explanation

This is a simple delay line built as a plugin for VCVRack using the DaisySP library

![DaisyDelay](https://github.com/fablabnk/VaseProtoPlugin/blob/delay_line_params/res/DelayProto.svg)

It's controls are:

- Two parameters:delay time and feedback)
- A switch that sets the delay range (short up to 0.5secs or long up to 5 secs)
- an LED (that currently does nothing)
- CV inputs for the two parameters
- Audio In and Out

See [here](https://github.com/fablabnk/VaseProtoPlugin/tree/delay_line) for version 1, with more notes and without parameters

- It is a first step towards prototyping a Daisy seed-based audio processing Eurorack module.
- The code is based on the Daisy example [here](https://github.com/electro-smith/DaisyExamples/blob/master/seed/DSP/delayline/delayline.cpp), but with the oscillator and envelope stripped out to create a simple fixed-length delay.

I added a convenience target to the Makefile `make plugin` which does `make`, `make dist` and `make install` together.

# Things I learned building this module...

## Plugin Element Categories

I learned more about the general structure of the plugin code in `DelayProto.cpp`, including how to declare the four categories of elements that make up a plugin:

- parameters
- input
- output
- lights

Note that the following example is for the `delay_line_simple` branch...

1. Module declaration

In `struct DelayProto : Module`, all four element categories must be provided, but we can leave some of them blank, by just ending them with NUM_xxx e.g.

```
	enum ParamIds {
		NUM_PARAMS
	};
	enum InputIds {
		AUDIO_INPUT,
		NUM_INPUTS
	};
	enum OutputIds {
		AUDIO_OUTPUT,
		NUM_OUTPUTS
	};
	enum LightIds {
		NUM_LIGHTS
	};
```

2. Constructor

In `DelayProto()`, we must always reference the four elements:
`config(NUM_PARAMS, NUM_INPUTS, NUM_OUTPUTS, NUM_LIGHTS);`

3. User Interface

The `struct DelayProtoWidget : ModuleWidget` represents the user interface. Here we just add the elements we actually use...

```
addInput(createInputCentered<PJ301MPort>(mm2px(Vec(15.24, 77.478)), module, DelayProto::AUDIO_INPUT));
addOutput(createOutputCentered<PJ301MPort>(mm2px(Vec(15.24, 108.713)), module, DelayProto::AUDIO_OUTPUT));
```

## Making the .svg panel graphic file

Elements in the .svg file should be separated into layers. I separated as follows:

- components (the colour circles representing VCVRack components i.e. input jack sockets, knobs, etc)
- text (as paths, .e. in Inkscape: Select the text, from Path menu choose `Object to Path`)
- panel (background representing the whole front panel)

Autogenerating the panel component positions didn't work for me
- the panel autogenerated Y values were wrong i.e. in the 200's not below 128. Why was this?

## Setting DelayTime correctly

This was more complex than it at first seemed. I misunderstood how to get the value of params and was treating them like inputs as follows:

```
inputs[PARAM_ONE_PARAM].getVoltage() 
```

When they should be treated like so:

```
params[PARAM_ONE_PARAM].getValue()
```

Inputs range between +-5v, whereas parameters range between the values you initialise them with i.e.

```
configParam(PARAM_ONE_PARAM, 0.f, 1.f, 1.f, "Delay Time (secs)");
```

## Setting short and long delay times with a switch (TODO)

I want to include a switch on the panel which allows switching between short and long delay times
- when set left/off = min delay time is 0.1 second, max delay time is 1 second
- when set right/on = min delay time is 0.5 seconds, max delay time is 5 seconds

## Smoothing Parameters

Parameters can be smoothed as follows i.e. to avoid jumps when turning the knob

```
fonepole(current_delay_, delay_target, .0002f); 
```

However this didn't help with avoid strange noises when moving the delay time knob. Odeally we should interpolate between delay time values over a larger amount of time. But I decided not to address this issue for now.

## Debugging using std::cout

I wanted to check if my delayTime was in the correct range (0.1 to 1.0) using a print statement. It's possible! I achieved this by:

1. Including the following header at top of .cpp
```
#include <iostream>
```

2. In the process function:
```
std::cout << delayTime << std::endl;
```

Output appears in terminal from where I'm running `./Rack`. Note: You might not get such output if you're running as an installed program on Windows or macOS.

## Components can be given 'hoverable' tooltip labels

In DelayProto(), we call various config functions, each of which ends with a string which is the tooltip text e.g.
```
configParam(PARAM_ONE_PARAM, 0.f, 1.f, 1.f, "Delay Time");
```

## How to use the values from the CV Inputs e.g. PARAM_ONE_CV_INPUT

I decided to combine the CV inputs with the value from their corresponding parameter knob. There is nothing under-the-hood to handle this as far as I could tell. Here was my approach:

1. I receive the parameter values as input voltage (-5v to +5v) and convert them to -1 to +1, as follows:

```
float paramOne = params[PARAM_ONE_PARAM].getValue() + (inputs[PARAM_ONE_CV_INPUT].getVoltage() / 5.0f);
```

2. I clamp them to within 0. - 1. bounds
```
float clampedParamOne = clamp(paramOne, 0.f, 1.f);
```

I rolled my own clamp function at the top of the .cpp file, which looks as follows:

```
template<typename T>
const T& clamp(const T& value, const T& low, const T& high) {
    return (value < low) ? low : (value > high) ? high : value;
}
```

3. After this we scale the resulting value as necessary, depending on the parameter time e.g. for delay time

```
float rangeDelayTimeSecs = maxDelayTimeSecs - minDelayTimeSecs;
float delayTime = (clampedParamOne * rangeDelayTimeSecs) + minDelayTimeSecs;
del.SetDelay(SAMPLE_RATE * delayTime);
```

Overall an attenuverter on each input would be nicer, so that the CV input doesn't cover the whole range. But this would make our hardware version more complicated

## Namespaces

We can reference our DaisySP code with or without a namespace.

1. Without a namespace (I chose this approach for now as we don't use many modules from DaisySP)

```
daisysp::DelayLine<float, MAX_DELAY> del;
```

2. With a namespace. For example by including the following line before our module declaration ```struct DelayProto : Module```
```
using namespace daisysp;
```

This could be a better approach when we are using many classes from the library together

## No 'size' loop is needed

Unlike in line 29 of the the [Daisy example](https://github.com/electro-smith/DaisyExamples/blob/master/seed/DSP/delayline/delayline.cpp#L29), we don't need to wrap our processing code in a for loop representing the audio block size.

```
for(size_t i = 0; i < size; i += 2)
{
// your DSP code here
}
```

Instead we simply output a single sample each time `void process` is called.

## Use of static

In [line 22 of the Daisy example] we can see the the DelayLine is declared as static. This applies to all variable, to ensure they take up a fixed size in memory of our embedded device. But when building VCVRack plugins, we don't and shouldn't use it. The plugin will build  but will not appear/load in VCVRack. Instead we simply do it like this:

```daisysp::DelayLine<float, MAX_DELAY> del;```

## Deriving the sample rate

We don't do that yet, our delay is hard coded to 48000Hz e.g.

```
#define MAX_DELAY static_cast<size_t>(48000 * 0.75f)
```

According to ChatGPT we could derive the sample rate by using `rack::engineGetSampleRate();` and by including "Rack.hpp" as a header, but we cannot use rack::engineGetSampleRate() directly in the above context because it's a runtime function and cannot be evaluated at compile time. So for now we live with the hardcoded value...

Also ChatGPT is wrong

```
src/DelayProto.cpp:26:23: error: ‘engineGetSampleRate’ is not a member of ‘rack’
   26 |                 rack::engineGetSampleRate();
```

I tried `rack::engine::getSampleRate();` and `rack::engine::Engine::getSampleRate();` and with the latter I get the error: `error: cannot call member function ‘float rack::engine::Engine::getSampleRate()’ without object`
