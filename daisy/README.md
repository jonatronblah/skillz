# daisy skill

A coding agent skill for programming the [Electrosmith Daisy](https://www.electro-smith.com/daisy) embedded audio platform using C++ with **libDaisy** (hardware support) and **DaisySP** (DSP library).

## What it covers

- **libDaisy**: Board classes (DaisySeed, DaisyPatch, DaisyPod, DaisyPetal, DaisyField, DaisyLegio, DaisyVersio, PatchSM), GPIO, ADC, DAC, audio callbacks, MIDI, OLED displays, encoders, switches, LEDs, persistent storage, SDRAM
- **DaisySP**: Oscillators (Oscillator, VariableSaw, VariableShape, Fm2, Formant, Grainlet, Harmonic, Vosim, ZOscillator), Filters (Svf, OnePole, Ladder, Soap, FIR), Effects (Chorus, Flanger, Phaser, Tremolo, Autowah, Decimator, Overdrive, Wavefolder, PitchShifter, SampleRateReducer), Envelopes (AdEnv, Adsr), Drums (AnalogBassDrum, AnalogSnareDrum, SyntheticBassDrum, SyntheticSnareDrum, HiHat), Physical Modeling (String, StringVoice, Resonator, ModalVoice, Drip), Noise (WhiteNoise, ClockedNoise, Dust, FractalRandom, Particle, SquareNoise, RingModNoise), Dynamics (Limiter, CrossFade, LinearVCA, SwingVCA), Sampling (GranularPlayer, Looper), Delay (DelayLine), Utility (Metro, Maytrig, SampleHold, DcBlock)
- **Common patterns**: Audio callbacks, FM synthesis, drum machines, delay with feedback, parameter smoothing

## Usage

This skill is automatically invoked when writing or debugging C++ code for the Daisy platform, or when explaining libDaisy/DaisySP APIs.

## References

- [DaisySP docs](https://electro-smith.github.io/DaisySP/index.html)
- [DaisySP source](https://github.com/electro-smith/DaisySP)
- [libDaisy docs](https://electro-smith.github.io/libDaisy/)
- [libDaisy source](https://github.com/electro-smith/libDaisy)
- [Daisy Wiki](https://github.com/electro-smith/DaisyWiki/wiki)
- [Daisy Examples](https://github.com/electro-smith/DaisyExamples)