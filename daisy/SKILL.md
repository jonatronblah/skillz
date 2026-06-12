---
name: daisy
description: Program the Electrosmith Daisy platform using C++ with libDaisy (hardware support) and DaisySP (DSP library). Covers board classes, peripherals, audio callbacks, and all DSP modules.
version: 1.0.0
license: MIT
author: "@electro-smith-community"
tags:
  - daisy
  - electro-smith
  - eurorack
  - modular-synth
  - c++
  - dsp
  - embedded
  - libdaisy
  - daisysp
---

## Instructions

Use this skill when writing, debugging, or explaining C++ code for the **Electrosmith Daisy** embedded audio platform, using **libDaisy** (hardware support library) and **DaisySP** (DSP library).

### When to Use

- Writing or editing Daisy C++ programs (`.cpp`, `.h`)
- Explaining libDaisy API (board classes, GPIO, ADC, DAC, audio, MIDI, displays, etc.)
- Explaining DaisySP API (oscillators, filters, effects, envelopes, drums, etc.)
- Setting up audio callbacks, block processing, and hardware initialization
- Debugging Daisy firmware or build issues
- Designing DSP signal chains on Daisy hardware
- Questions about Daisy board variants (Seed, Patch, Pod, Petal, Field, etc.)

### Key Principles

- Daisy programs are written in **C++** and compiled with ARM GCC for STM32
- All DSP classes live in the `daisysp` namespace (`#include "daisysp.h"`)
- All hardware classes live in the `daisy` namespace (`#include "daisy.h"`)
- Audio is processed in **callback functions** called at the audio sample rate
- Use `Init(sample_rate)` on DSP objects before calling `Process()`
- Audio callbacks receive `float*` buffers (non-interleaved) or interleaved `float*`
- Block-based processing: `SetAudioBlockSize()` controls samples per callback
- Default sample rate is 48kHz; block size is typically 48 samples
- Build with `make` in the project directory; flash with `make program`
- Use `DaisySeed::GetPin(pin_idx)` for GPIO pin mapping (0-31, labeled 1-32)

---

## Build System

### Project Setup

```bash
# Clone libraries
git clone https://github.com/electro-smith/DaisySP
git clone https://github.com/electro-smith/libDaisy

# Create a new project (use the Daisy project template)
# Or manually create a Makefile referencing both libraries
```

### Typical Makefile Structure

```makefile
# Core paths
DAISYSP_DIR = ../../DaisySP
LIBDAISY_DIR = ../../libDaisy

# Source files
CXX_SOURCES = main.cpp

# Include libDaisy and DaisySP build infrastructure
include $(LIBDAISY_DIR)/Makefile
include $(DAISYSP_DIR)/Makefile
```

### Build & Flash

```bash
make        # build the firmware
make program  # flash via ST-Link or DFU bootloader
```

---

## libDaisy — Hardware Support Library

### Board Classes

All board classes provide `Init()`, `StartAudio()`, `StartAdc()`, `ProcessAnalogControls()`, and `ProcessDigitalControls()`.

#### DaisySeed

The base board class. All other board classes contain a `DaisySeed seed` member.

```cpp
#include "daisy_seed.h"

daisy::DaisySeed hw;
hw.Init(true);  // true = boost mode (480MHz)
hw.StartAudio(AudioCallback);
hw.StartLog(true);
hw.Print("Hello Daisy!\n");

// Pin access
daisy::Pin p = daisy::DaisySeed::GetPin(0);  // pin 0 (labeled 1)

// Audio config
hw.SetAudioBlockSize(48);       // 48 samples per block
hw.SetAudioSampleRate(daisy::SaiHandle::Config::SampleRate::SAI_48KHZ);
float sr = hw.AudioSampleRate();  // returns 48000.0f
```

**Key members:** `adc`, `dac`, `audio_handle`, `qspi`, `sdram_handle`, `usb_handle`, `led`, `testpoint`, `system`

**Audio callback types:**

- `AudioHandle::AudioCallback` — non-interleaved: `void(float* in, float* out, size_t size)`
- `AudioHandle::InterleavingAudioCallback` — interleaved: `void(float* in, float* out, size_t size)`

#### DaisyPatch

4-in / 4-out audio, 4 CV/knob controls, 2 gate inputs, encoder, OLED display, gate output, MIDI.

```cpp
#include "daisy_patch.h"

daisy::DaisyPatch hw;
hw.Init();

// Controls
hw.ProcessAnalogControls();
hw.ProcessDigitalControls();
float knob1 = hw.GetKnobValue(daisy::DaisyPatch::CTRL_1);  // 0.0 - 1.0
bool gate1 = hw.gate_input[daisy::DaisyPatch::GATE_IN_1].State();

// Display
hw.display.Fill(false);  // clear
hw.display.SetCursor(0, 0);
hw.display.WriteString("Hello", Font_7x10, true);
hw.display.Update();

// Audio
hw.StartAudio(AudioCallback);
```

**Members:** `seed`, `controls[4]`, `encoder`, `gate_input[2]`, `gate_output`, `display`, `midi`, `codec`

#### DaisyPod

2-in / 2-out audio, 2 knobs, 2 buttons, encoder, 2 RGB LEDs, MIDI.

```cpp
#include "daisy_pod.h"

daisy::DaisyPod hw;
hw.Init();

hw.ProcessAllControls();
float k1 = hw.GetKnobValue(daisy::DaisyPod::KNOB_1);
bool b1 = hw.button1.Pressed();

hw.led1.SetColor(daisy::Color(1.0f, 0.0f, 0.0f));  // red
hw.UpdateLeds();
```

**Members:** `seed`, `knob1`, `knob2`, `button1`, `button2`, `encoder`, `led1`, `led2`, `midi`

#### Other Boards

| Board               | Header           | Description                                                            |
| ------------------- | ---------------- | ---------------------------------------------------------------------- |
| `DaisyPetal`        | `daisy_petal.h`  | Guitar pedal: 4 knobs, 4 switches, encoder, footswitch, LED ring, OLED |
| `DaisyField`        | `daisy_field.h`  | Keyboard: 16 CV keys, 2 knobs, 2 switches, OLED, MIDI                  |
| `DaisyLegio`        | `daisy_legio.h`  | Virt Iter Legio: 2 knobs, 2 switches, 2 RGB LEDs                       |
| `DaisyVersio`       | `daisy_versio.h` | Desmodus Versio: 4 knobs, 4 switches, 2 LEDs                           |
| `patch_sm::PatchSM` | `patch_sm.h`     | Patch Sub Module: eurorack module with CV I/O                          |

---

### Peripherals

#### GPIO

```cpp
daisy::GPIO pin;
daisy::Pin my_pin = daisy::DaisySeed::GetPin(0);  // or construct: daisy::Pin(daisy::PORTA, 4)

pin.Init(my_pin, daisy::GPIO::Mode::OUTPUT);
pin.Write(true);   // high
bool state = pin.Read();
```

**Modes:** `INPUT`, `OUTPUT`, `ANALOG` (for ADC)

#### ADC (AnalogControl)

```cpp
// Configure ADC channels
daisy::AdcChannelConfig adc_config[2];
adc_config[0].InitSingle(daisy::DaisySeed::GetPin(21));  // pin 21
adc_config[1].InitSingle(daisy::DaisySeed::GetPin(22));

hw.adc.Init(adc_config, 2);
hw.adc.Start();

float val = hw.adc.GetFloat(0);  // 0.0 - 1.0
```

#### DAC

```cpp
daisy::DacHandle::Config dac_cfg;
dac_cfg.mode = daisy::DacHandle::Mode::DAC_OUTPUT;
dac_cfg.bitdepth = daisy::DacHandle::BitDepth::BITS_12;
dac_cfg.chn = daisy::DacHandle::Channel::BOTH;
hw.dac.Init(dac_cfg);
hw.dac.WriteValue(daisy::DacHandle::Channel::ONE, 2048);
```

#### Switch / Button

```cpp
daisy::Switch btn;
btn.Init(daisy::DaisySeed::GetPin(27), daisy::Switch::Type::TYPE_MOMENTARY,
         daisy::Switch::Polarity::POLARITY_INVERTED,
         daisy::Switch::Pull::PULL_UP);

btn.Debounce();
if(btn.Pressed()) { /* held */ }
if(btn.RisingEdge()) { /* just pressed */ }
if(btn.FallingEdge()) { /* just released */ }
```

#### Encoder

```cpp
daisy::Encoder enc;
enc.Init(pin_a, pin_b, pin_click, daisy::Switch::Pull::PULL_UP);
enc.Debounce();
int32_t increment = enc.Increment();  // +1 or -1 per detent
bool clicked = enc.RisingEdge();
```

#### Gate Input

```cpp
daisy::GateIn gate;
gate.Init(daisy::DaisySeed::GetPin(0));
bool state = gate.State();      // current state
bool trig = gate.Trig();        // true on rising edge (one-shot)
```

#### LED / RgbLed

```cpp
daisy::Led led;
led.Init(daisy::DaisySeed::GetPin(0), daisy::Led::Polarity::INVERTED, 1000);  // 1kHz PWM
led.Set(0.5f);  // 50% brightness
led.Update();

daisy::RgbLed rgb;
rgb.Init(pin_r, pin_g, pin_b, daisy::Led::Polarity::INVERTED);
rgb.SetColor(daisy::Color(1.0f, 0.5f, 0.0f));  // orange
rgb.Update();
```

#### OLED Display

```cpp
// Typically accessed through board class
hw.display.Fill(false);
hw.display.SetCursor(0, 0);
hw.display.WriteString("Hello!", daisy::Font_7x10, true);
hw.display.DrawRect(0, 0, 127, 63, true);
hw.display.Update();
```

#### MIDI

```cpp
daisy::MidiUartHandler midi;
daisy::MidiUartHandler::Config midi_cfg;
midi.Init(midi_cfg);
midi.StartReceive();

// In main loop:
while(midi.HasEvents()) {
    auto msg = midi.PopEvent();
    if(msg.type == daisy::NoteOn) {
        uint8_t note = msg.data[0];
        uint8_t vel = msg.data[1];
    }
}
```

#### Persistent Storage

```cpp
daisy::PersistentStorage<uint32_t> storage;
daisy::PersistentStorage<uint32_t>::Config cfg;
cfg.file_name = "settings.bin";
storage.Init(cfg);
uint32_t val = storage.GetSettings().value;
storage.Save();
```

#### SDRAM

```cpp
// SDRAM is initialized by DaisySeed::Init()
// Allocate large buffers on SDRAM:
float* buffer = (float*)daisy::SDRAM_PERIPH;  // base address
// Or use the SdramHandle directly
```

---

## DaisySP — DSP Library

All classes are in the `daisysp` namespace. Every DSP object follows the pattern: `Init(sample_rate)` → `SetParam(value)` → `Process()` per sample.

### Synthesis

#### Oscillator

Bandlimited waveform oscillator with polyBLEP anti-aliasing.

```cpp
daisysp::Oscillator osc;
osc.Init(sample_rate);
osc.SetFreq(440.0f);
osc.SetAmp(0.5f);
osc.SetWaveform(daisysp::Oscillator::WAVE_SIN);
float out = osc.Process();
```

**Waveforms:** `WAVE_SIN`, `WAVE_TRI`, `WAVE_SAW`, `WAVE_RAMP`, `WAVE_SQUARE`, `WAVE_POLYBLEP_TRI`, `WAVE_POLYBLEP_SAW`, `WAVE_POLYBLEP_SQUARE`

**Methods:** `SetFreq(f)`, `SetAmp(a)`, `SetWaveform(wf)`, `SetPw(pw)`, `Process()`, `PhaseAdd(phase)`, `Reset(phase)`, `IsEOR()`, `IsEOC()`, `IsRising()`, `IsFalling()`

#### VariableSawOscillator

```cpp
daisysp::VariableSawOscillator osc;
osc.Init(sample_rate);
osc.SetFreq(220.0f);
osc.SetWaveshape(0.5f);  // 0=saw, 1=square
float out = osc.Process();
```

#### VariableShapeOscillator

```cpp
daisysp::VariableShapeOscillator osc;
osc.Init(sample_rate);
osc.SetFreq(220.0f);
osc.SetWaveshape(0.5f);  // 0=saw, 0.5=triangle, 1=pulse
osc.SetPW(0.5f);
float out = osc.Process();
```

#### Fm2

2-operator FM synthesis.

```cpp
daisysp::Fm2 fm;
fm.Init(sample_rate);
fm.SetFrequency(220.0f);
fm.SetRatio(2.0f);     // modulator:carrier ratio
fm.SetIndex(1.0f);     // modulation index
float out = fm.Process();
```

#### FormantOscillator

```cpp
daisysp::FormantOscillator osc;
osc.Init(sample_rate);
osc.SetFormantFreq(800.0f);  // formant frequency
osc.SetCarrierFreq(150.0f);  // fundamental
osc.SetPhaseShift(0.0f);
float out = osc.Process();
```

#### GrainletOscillator

```cpp
daisysp::GrainletOscillator osc;
osc.Init(sample_rate);
osc.SetFreq(220.0f);
osc.SetGrainFreq(80.0f);
osc.SetShape(0.5f);
osc.SetBleed(0.0f);
float out = osc.Process();
```

#### HarmonicOscillator

Chebyshev polynomial-based harmonic oscillator.

```cpp
daisysp::HarmonicOscillator<8> osc;  // 8 harmonics
osc.Init(sample_rate);
osc.SetFreq(220.0f);
osc.SetFirstHarmIdx(1);
float amplitudes[8] = {1.0f, 0.5f, 0.25f, 0.0f, 0.0f, 0.0f, 0.0f, 0.0f};
osc.SetAmplitudes(amplitudes);
float out = osc.Process();
```

#### OscillatorBank

```cpp
daisysp::OscillatorBank osc;
osc.Init(sample_rate);
osc.SetFreq(220.0f);
osc.SetAmp(0.5f);
float out = osc.Process();
```

#### VosimOscillator

```cpp
daisysp::VosimOscillator osc;
osc.Init(sample_rate);
osc.SetFreq(220.0f);
osc.SetFormantFreq(800.0f);
osc.SetShape(0.5f);
float out = osc.Process();
```

#### ZOscillator

```cpp
daisysp::ZOscillator osc;
osc.Init(sample_rate);
osc.SetFreq(220.0f);
osc.SetFormantFreq(800.0f);
osc.SetShape(0.5f);
osc.SetMode(0.0f);
float out = osc.Process();
```

---

### Control Signal Generators

#### AdEnv

Attack-Decay envelope.

```cpp
daisysp::AdEnv env;
env.Init(sample_rate);
env.SetTime(daisysp::AdEnv::ATTACK, 0.01f);
env.SetTime(daisysp::AdEnv::DECAY, 0.5f);
env.SetMin(0.0f);
env.SetMax(1.0f);
env.Trigger();  // start the envelope
float out = env.Process();
```

#### Adsr

Attack-Decay-Sustain-Release envelope.

```cpp
daisysp::Adsr env;
env.Init(sample_rate);
env.SetTime(daisysp::Adsr::ATTACK, 0.01f);
env.SetTime(daisysp::Adsr::DECAY, 0.1f);
env.SetSustainLevel(0.7f);
env.SetTime(daisysp::Adsr::RELEASE, 0.5f);
bool gate = true;
float out = env.Process(gate);  // pass gate signal
```

#### Phasor

Simple ramp generator (0→1).

```cpp
daisysp::Phasor phasor;
phasor.Init(sample_rate);
phasor.SetFreq(1.0f);  // 1 Hz
float phase = phasor.Process();  // 0.0 to 1.0
```

---

### Filters

#### Svf

Double-sampled stable state variable filter with simultaneous LP, HP, BP, notch, and peak outputs.

```cpp
daisysp::Svf flt;
flt.Init(sample_rate);
flt.SetFreq(1000.0f);
flt.SetRes(0.5f);     // 0.0 - 1.0
flt.SetDrive(0.0f);

// Process input, then read desired output
flt.Process(input);
float lp = flt.Low();
float hp = flt.High();
float bp = flt.Band();
float notch = flt.Notch();
float peak = flt.Peak();
```

#### OnePole

Simple one-pole lowpass or highpass.

```cpp
daisysp::OnePole flt;
flt.Init(sample_rate);
flt.SetFilter(daisysp::OnePole::FILTER_LP);  // or FILTER_HP
flt.SetFreq(1000.0f);
float out = flt.Process(input);
```

#### LadderFilter

Moog-style ladder filter.

```cpp
daisysp::LadderFilter flt;
flt.Init(sample_rate);
flt.SetRes(0.5f);
flt.SetFreq(1000.0f);
flt.SetDrive(0.0f);
float out = flt.Process(input);
```

#### Soap

Second-order all-pass filter.

```cpp
daisysp::Soap flt;
flt.Init(sample_rate);
flt.SetFreq(1000.0f);
flt.SetQ(0.5f);
flt.Process(input);
float out = flt.AllPass();  // or flt.BandPass(), flt.Notch()
```

#### FIR Filter

```cpp
// Generic FIR or ARM CMSIS DSP based
float coeffs[8] = {0.1f, 0.2f, 0.3f, 0.4f, 0.3f, 0.2f, 0.1f, 0.0f};
daisysp::FIRFilterImplGeneric<8> fir;
fir.Init(coeffs, 8);
float out = fir.Process(input);
```

---

### Effects

#### Chorus

```cpp
daisysp::Chorus ch;
ch.Init(sample_rate);
ch.SetLfoFreq(0.5f);   // LFO frequency
ch.SetLfoAmp(0.5f);    // LFO depth
ch.SetDelay(0.5f);     // base delay
ch.SetFeedback(0.5f);
float out = ch.Process(input);
```

#### Flanger

```cpp
daisysp::Flanger fl;
fl.Init(sample_rate);
fl.SetLfoFreq(0.5f);
fl.SetLfoAmp(0.5f);
fl.SetDelay(0.5f);
fl.SetFeedback(0.5f);
float out = fl.Process(input);
```

#### Phaser

```cpp
daisysp::Phaser ph;
ph.Init(sample_rate);
ph.SetLfoFreq(0.5f);
ph.SetLfoAmp(0.5f);
ph.SetFeedback(0.5f);
float out = ph.Process(input);
```

#### Tremolo

```cpp
daisysp::Tremolo tr;
tr.Init(sample_rate);
tr.SetDepth(0.5f);
tr.SetFreq(5.0f);
float out = tr.Process(input);
```

#### Autowah

```cpp
daisysp::Autowah aw;
aw.Init(sample_rate);
aw.SetWah(0.5f);
aw.SetDryWet(0.5f);
aw.SetLevel(0.5f);
float out = aw.Process(input);
```

#### Decimator

Sample rate and bit depth reduction.

```cpp
daisysp::Decimator dec;
dec.Init();
dec.SetDownsampleFactor(0.5f);  // 0.0-1.0, lower = more decimated
dec.SetBitsCrush(8);             // bit depth (1-32)
float out = dec.Process(input);
```

#### Overdrive

```cpp
daisysp::Overdrive od;
od.Init();
od.SetDrive(0.5f);  // 0.0-1.0
float out = od.Process(input);
```

#### Wavefolder

```cpp
daisysp::Wavefolder wf;
wf.Init();
wf.SetGain(1.0f);
wf.SetOffset(0.0f);
float out = wf.Process(input);
```

#### PitchShifter

```cpp
daisysp::PitchShifter ps;
ps.Init(sample_rate);
ps.SetTransposition(7.0f);  // semitones, can be fractional
float out = ps.Process(input);
```

#### SampleRateReducer

```cpp
daisysp::SampleRateReducer srr;
srr.Init(sample_rate);
srr.SetFreq(8000.0f);  // target sample rate
float out = srr.Process(input);
```

---

### Dynamics

#### Limiter

```cpp
daisysp::Limiter lim;
lim.Init(sample_rate);
lim.SetThreshold(-6.0f);  // dB
float out = lim.Process(input, 1.0f);  // input, gain
```

#### CrossFade

```cpp
daisysp::CrossFade xf;
xf.Init();
xf.SetPos(0.5f);  // 0.0 = sig1, 1.0 = sig2
float out = xf.Process(sig1, sig2);
```

#### LinearVCA / SwingVCA

```cpp
daisysp::LinearVCA vca;
vca.Init();
vca.SetGain(0.5f);
float out = vca.Process(input, gain);

daisysp::SwingVCA svca;
svca.Init();
svca.SetStrength(0.5f);
float out = svca.Process(input, gain);
```

---

### Noise Generators

#### WhiteNoise

```cpp
daisysp::WhiteNoise noise;
noise.Init();
noise.SetAmp(0.5f);
float out = noise.Process();
```

#### ClockedNoise

```cpp
daisysp::ClockedNoise cn;
cn.Init(sample_rate);
cn.SetFreq(100.0f);  // clock frequency
cn.Sync();            // sync/reset
float out = cn.Process();
```

#### Dust

Random impulse generator.

```cpp
daisysp::Dust dust;
dust.Init(sample_rate);
dust.SetDensity(0.5f);  // 0.0-1.0
float out = dust.Process();
```

#### FractalRandomGenerator

Stacked octaves of noise.

```cpp
daisysp::FractalRandomGenerator frg;
frg.Init(sample_rate);
frg.SetFreq(100.0f);
frg.SetColor(0.5f);  // 0.0=bright, 1.0=dark
float out = frg.Process();
```

#### Particle

Random impulse train through resonant filter.

```cpp
daisysp::Particle pt;
pt.Init(sample_rate);
pt.SetFreq(440.0f);       // resonant filter frequency
pt.SetResonance(0.5f);
pt.SetRandomFreq(1000.0f);
pt.SetDensity(0.5f);
pt.SetSpread(0.5f);
float out = pt.Process();
```

#### SquareNoise / RingModNoise

808-style metallic noise sources.

```cpp
daisysp::SquareNoise sqn;
sqn.Init(sample_rate);
sqn.SetFreq(1000.0f);
float out = sqn.Process();

daisysp::RingModNoise rmn;
rmn.Init(sample_rate);
rmn.SetFreq(1000.0f);
float out = rmn.Process();
```

---

### Drum Synthesis

#### AnalogBassDrum

808 bass drum model.

```cpp
daisysp::AnalogBassDrum bd;
bd.Init(sample_rate);
bd.SetFreq(60.0f);
bd.SetSustain(false);
bd.SetAccent(0.5f);
bd.SetTone(0.5f);
bd.SetDecay(0.5f);
bd.SetAttackFm(0.5f);
bd.SetSelfFm(0.5f);
float out = bd.Trig();  // returns one sample, call per-sample after trigger
```

#### AnalogSnareDrum

808 snare drum model.

```cpp
daisysp::AnalogSnareDrum sd;
sd.Init(sample_rate);
sd.SetFreq(200.0f);
sd.SetSustain(false);
sd.SetAccent(0.5f);
sd.SetTone(0.5f);
sd.SetDecay(0.5f);
sd.SetSnappy(0.5f);
float out = sd.Trig();
```

#### SyntheticBassDrum

Naive bass drum (modulated oscillator with FM + envelope).

```cpp
daisysp::SyntheticBassDrum sbd;
sbd.Init(sample_rate);
sbd.SetFreq(60.0f);
sbd.SetSustain(false);
sbd.SetAccent(0.5f);
sbd.SetTone(0.5f);
sbd.SetDecay(0.5f);
sbd.SetAttack(0.01f);
float out = sbd.Trig();
```

#### SyntheticSnareDrum

Naive snare (two modulated oscillators + filtered noise).

```cpp
daisysp::SyntheticSnareDrum ssd;
ssd.Init(sample_rate);
ssd.SetFreq(200.0f);
ssd.SetSustain(false);
ssd.SetAccent(0.5f);
ssd.SetTone(0.5f);
ssd.SetDecay(0.5f);
ssd.SetSnappy(0.5f);
float out = ssd.Trig();
```

#### HiHat

808 hi-hat with extended parameters.

```cpp
daisysp::HiHat hh;
hh.Init(sample_rate);
hh.SetSustain(false);
hh.SetAccent(0.5f);
hh.SetDecay(0.5f);
hh.SetTone(0.5f);
hh.SetNoisiness(0.5f);
hh.SetStick(0.5f);
float out = hh.Trig();
```

---

### Physical Modeling

#### String

Comb filter / Karplus-Strong string.

```cpp
daisysp::String str;
str.Init(sample_rate);
str.SetFreq(220.0f);
str.SetBrightness(0.5f);
str.SetDamping(0.5f);
str.SetNonlinearity(0.0f);
str.Trig();  // excite the string
float out = str.Process();
```

#### StringVoice

Extended Karplus-Strong with Rings-style features.

```cpp
daisysp::StringVoice sv;
sv.Init(sample_rate);
sv.SetFreq(220.0f);
sv.SetBrightness(0.5f);
sv.SetDamping(0.5f);
sv.SetStructure(0.5f);
sv.SetAccent(0.5f);
sv.Trig();
float out = sv.Process();
```

#### Resonator

Resonant body simulation.

```cpp
daisysp::Resonator res;
res.Init(sample_rate);
res.SetFreq(220.0f);
res.SetStructure(0.5f);
res.SetBrightness(0.5f);
res.SetDamping(0.5f);
float out = res.Process(excitation_signal);
```

#### ModalVoice

Modal synthesis with mallet exciter.

```cpp
daisysp::ModalVoice mv;
mv.Init(sample_rate);
mv.SetFreq(220.0f);
mv.SetBrightness(0.5f);
mv.SetDamping(0.5f);
mv.SetStructure(0.5f);
mv.SetAccent(0.5f);
mv.Trig();
float out = mv.Process();
```

#### Drip

Water drop model.

```cpp
daisysp::Drip drip;
drip.Init(sample_rate);
drip.SetDamping(0.5f);
drip.SetEnergy(0.5f);
drip.Trig();
float out = drip.Process();
```

---

### Sampling

#### GranularPlayer

```cpp
daisysp::GranularPlayer gp;
gp.Init(sample_rate, sample_buffer, buffer_size);
gp.SetFreq(1.0f);       // grain frequency
gp.SetGrainSize(0.1f);  // grain size in seconds
gp.SetOverlap(0.5f);
gp.SetSpeed(1.0f);
float out = gp.Process();
```

#### Looper

```cpp
daisysp::Looper lp;
lp.Init(sample_buffer, buffer_size);
lp.SetMode(daisysp::Looper::Mode::NORMAL);  // NORMAL, FREEZE, PLAY
lp.SetSpeed(1.0f);
lp.Record(true);
float out = lp.Process(input);
```

---

### Delay

#### DelayLine

Template-based delay line.

```cpp
daisysp::DelayLine<float, 48000> delay;  // max 1 second at 48kHz
delay.Init();
delay.SetDelay(24000);  // 0.5 seconds in samples
delay.Write(input);
float out = delay.Read();
float out_fb = delay.Read();  // read again for feedback
delay.Write(input + out_fb * 0.5f);  // feedback
```

---

### Utility

#### Metro

Metronome / clock generator.

```cpp
daisysp::Metro metro;
metro.Init(sample_rate);
metro.SetFreq(2.0f);  // 2 Hz
bool tick = metro.Process();  // true on each tick
```

#### Maytrig

Probability-based trigger.

```cpp
daisysp::Maytrig mt;
mt.Init();
bool triggered = mt.Process(0.5f);  // 50% chance
```

#### SampleHold

```cpp
daisysp::SampleHold sh;
sh.Init();
float out = sh.Process(trigger, input);  // holds input on trigger
```

#### DcBlock

DC blocker filter.

```cpp
daisysp::DcBlock dc;
dc.Init(sample_rate);
float out = dc.Process(input);
```

#### Looper

See Sampling section above.

---

## Common Patterns

### Basic Audio Callback (DaisySeed)

```cpp
#include "daisy_seed.h"
#include "daisysp.h"

using namespace daisy;
using namespace daisysp;

DaisySeed hw;
Oscillator osc;
AdEnv env;

void AudioCallback(float* in, float* out, size_t size) {
    for(size_t i = 0; i < size; i++) {
        float sig = osc.Process() * env.Process();
        out[i] = sig;           // left
        out[size + i] = sig;    // right (non-interleaved)
    }
}

int main() {
    hw.Init();
    float sr = hw.AudioSampleRate();

    osc.Init(sr);
    osc.SetFreq(440.0f);
    osc.SetWaveform(Oscillator::WAVE_SIN);

    env.Init(sr);
    env.SetTime(AdEnv::ATTACK, 0.01f);
    env.SetTime(AdEnv::DECAY, 0.5f);

    hw.StartAudio(AudioCallback);
    while(1) {}
}
```

### DaisyPatch with Controls

```cpp
#include "daisy_patch.h"
#include "daisysp.h"

using namespace daisy;
using namespace daisysp;

DaisyPatch hw;
Oscillator osc;
Svf flt;

void AudioCallback(float* in, float* out, size_t size) {
    hw.ProcessAnalogControls();
    float freq = hw.GetKnobValue(DaisyPatch::CTRL_1) * 2000.0f + 50.0f;
    float cutoff = hw.GetKnobValue(DaisyPatch::CTRL_2) * 8000.0f + 100.0f;
    float res = hw.GetKnobValue(DaisyPatch::CTRL_3);

    osc.SetFreq(freq);
    flt.SetFreq(cutoff);
    flt.SetRes(res);

    for(size_t i = 0; i < size; i++) {
        float sig = osc.Process();
        flt.Process(sig);
        out[i] = flt.Low();
        out[size + i] = out[i];
    }
}

int main() {
    hw.Init();
    float sr = hw.AudioSampleRate();
    osc.Init(sr);
    flt.Init(sr);
    hw.StartAudio(AudioCallback);
    while(1) {}
}
```

### FM Synthesis

```cpp
Oscillator carrier, modulator;
float mod_index = 100.0f;
float ratio = 2.0f;
float base_freq = 220.0f;

// In callback:
modulator.SetFreq(base_freq * ratio);
float mod = modulator.Process() * mod_index;
carrier.SetFreq(base_freq + mod);
float out = carrier.Process();
```

### Drum Machine Pattern

```cpp
AnalogBassDrum bd;
AnalogSnareDrum sd;
HiHat hh;
Metro metro;
int step = 0;

// In callback:
if(metro.Process()) {
    step = (step + 1) % 16;
    if(step % 4 == 0) bd.Trig();
    if(step % 8 == 4) sd.Trig();
    if(step % 2 == 0) hh.Trig();
}
float out = bd.Process() + sd.Process() + hh.Process();
```

### Delay with Feedback

```cpp
DelayLine<float, 48000> delay;
float feedback = 0.5f;

// In init:
delay.Init();
delay.SetDelay(24000);  // 0.5s

// In callback:
float del_out = delay.Read();
delay.Write(input + del_out * feedback);
float out = input + del_out * 0.7f;
```

### Parameter Smoothing

```cpp
// Use OnePole as a smoother
OnePole smoother;
smoother.Init(sr);
smoother.SetFilter(OnePole::FILTER_LP);
smoother.SetFreq(10.0f);  // 10Hz smoothing

// In callback:
float target = hw.GetKnobValue(DaisyPatch::CTRL_1) * 2000.0f;
float smooth_freq = smoother.Process(target);
osc.SetFreq(smooth_freq);
```

---

## References

- **DaisySP docs**: https://electro-smith.github.io/DaisySP/index.html
- **DaisySP source**: https://github.com/electro-smith/DaisySP
- **libDaisy docs**: https://electro-smith.github.io/libDaisy/
- **libDaisy source**: https://github.com/electro-smith/libDaisy
- **Daisy Wiki**: https://github.com/electro-smith/DaisyWiki/wiki
- **Daisy Examples**: https://github.com/electro-smith/DaisyExamples
- **Daisy Forum**: https://forum.electro-smith.com/
- **Daisy Discord**: https://discord.gg/ByHBnMtQTR
