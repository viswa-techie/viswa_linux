# Chapter 4: Audio Framework (ALSA / ASoC)

## Learning Goals
- Understand ALSA and ASoC architecture for embedded audio
- Know the machine/platform/codec driver model
- Grasp PCM streaming, mixer controls, and DAPM
- Trace audio data flow from user space to hardware

---

## 4.1 ALSA Architecture Overview

```
ALSA Architecture:

User Space:
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ PulseAud │  │ PipeWire │  │  aplay   │
  │ AudioFlgr│  │          │  │  arecord │
  └─────┬────┘  └─────┬────┘  └─────┬────┘
        │              │              │
        └──────────────┼──────────────┘
                       │  /dev/snd/*
Kernel:                │
  ┌────────────────────▼─────────────────────────────┐
  │              ALSA Core                            │
  │  ┌────────────┐ ┌────────────┐ ┌──────────────┐  │
  │  │ PCM Core   │ │ Control    │ │ Timer Core   │  │
  │  │ (capture/  │ │ (mixer     │ │              │  │
  │  │  playback) │ │  volume)   │ │              │  │
  │  └─────┬──────┘ └─────┬──────┘ └──────────────┘  │
  │        │              │                           │
  │  ┌─────▼──────────────▼──────────────────────┐    │
  │  │          ASoC Framework                    │    │
  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐   │    │
  │  │  │ Machine  │ │ Platform │ │  Codec   │   │    │
  │  │  │ Driver   │ │ Driver   │ │  Driver  │   │    │
  │  │  │ (board   │ │ (DMA/    │ │ (DAC/ADC │   │    │
  │  │  │  wiring) │ │  I2S)    │ │  chip)   │   │    │
  │  │  └──────────┘ └──────────┘ └──────────┘   │    │
  │  └────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────┘
                       │
Hardware:              ▼
  ┌──────────────────────────────────────────────┐
  │ SoC I2S/TDM ──► Audio Codec (e.g., WM8960) │
  │ DMA Engine        I2C control + I2S data     │
  └──────────────────────────────────────────────┘
```

---

## 4.2 ASoC Three-Driver Model

```
ASoC Model — Three drivers collaborate:

┌─────────────────────────┐
│    Machine Driver       │
│ (Board-specific)        │
│ ● Connects codec → SoC │
│ ● Defines DAI links     │
│ ● Clock routing         │
│ ● Amp/jack GPIOs        │
└────────┬────────────────┘
         │ snd_soc_dai_link
   ┌─────┴─────────────────────────┐
   │                               │
   ▼                               ▼
┌──────────────┐        ┌──────────────────┐
│ Platform Drv │        │   Codec Driver   │
│ (SoC-side)   │        │ (Codec chip)     │
│              │        │                  │
│ ● CPU DAI    │  I2S/  │ ● Codec DAI      │
│   (I2S/TDM)  │◄──────►│   (I2S/TDM)      │
│ ● DMA engine │  data  │ ● Mixer controls │
│ ● PCM ops    │        │ ● DAPM widgets   │
│              │        │ ● I2C/SPI regs   │
└──────────────┘        └──────────────────┘

DAI = Digital Audio Interface (I2S, TDM, PDM)
```

```c
/* Machine driver — connects SoC I2S to codec */
SND_SOC_DAILINK_DEFS(playback,
    DAILINK_COMP_ARRAY(COMP_CPU("i2s0")),
    DAILINK_COMP_ARRAY(COMP_CODEC("wm8960.1-001a", "wm8960-hifi")),
    DAILINK_COMP_ARRAY(COMP_PLATFORM("i2s0")));

static struct snd_soc_dai_link my_board_dai[] = {
    {
        .name           = "Primary",
        .stream_name    = "Playback",
        .dai_fmt        = SND_SOC_DAIFMT_I2S |
                          SND_SOC_DAIFMT_NB_NF |
                          SND_SOC_DAIFMT_CBS_CFS,
        SND_SOC_DAILINK_REG(playback),
    },
};

static struct snd_soc_card my_sound_card = {
    .name      = "My-Board-Audio",
    .owner     = THIS_MODULE,
    .dai_link  = my_board_dai,
    .num_links = ARRAY_SIZE(my_board_dai),
};

static int my_audio_probe(struct platform_device *pdev)
{
    my_sound_card.dev = &pdev->dev;
    return devm_snd_soc_register_card(&pdev->dev, &my_sound_card);
}
```

---

## 4.3 Platform (CPU DAI) Driver

```c
/* Platform/CPU DAI driver — manages I2S controller + DMA */

static int my_i2s_hw_params(struct snd_pcm_substream *substream,
                            struct snd_pcm_hw_params *params,
                            struct snd_soc_dai *dai)
{
    unsigned int rate = params_rate(params);
    unsigned int channels = params_channels(params);
    unsigned int width = params_width(params);

    /* Configure I2S clock dividers for sample rate */
    my_i2s_set_clk(rate, channels, width);
    return 0;
}

static const struct snd_soc_dai_ops my_i2s_dai_ops = {
    .hw_params   = my_i2s_hw_params,
    .set_fmt     = my_i2s_set_fmt,
    .trigger     = my_i2s_trigger,  /* start/stop DMA */
};

static struct snd_soc_dai_driver my_i2s_dai = {
    .name = "i2s0",
    .playback = {
        .channels_min = 2,
        .channels_max = 8,
        .rates = SNDRV_PCM_RATE_8000_192000,
        .formats = SNDRV_PCM_FMTBIT_S16_LE |
                   SNDRV_PCM_FMTBIT_S24_LE |
                   SNDRV_PCM_FMTBIT_S32_LE,
    },
    .capture = { /* similar */ },
    .ops = &my_i2s_dai_ops,
};

static const struct snd_soc_component_driver my_i2s_component = {
    .name = "my-i2s",
    .pcm_construct = my_pcm_construct,  /* allocate DMA buffers */
};
```

---

## 4.4 Codec Driver

```c
/* Codec driver — DAC/ADC chip (e.g., WM8960) controlled via I2C */

/* Mixer controls — volume, mute */
static const struct snd_kcontrol_new wm8960_controls[] = {
    SOC_DOUBLE_R_TLV("Headphone Volume",
                     WM8960_LOUT1, WM8960_ROUT1,
                     0, 127, 0, hp_tlv),
    SOC_DOUBLE("Headphone Switch",
               WM8960_LOUT1, WM8960_ROUT1, 7, 1, 1),
    SOC_DOUBLE_R_TLV("Speaker Volume",
                     WM8960_LOUT2, WM8960_ROUT2,
                     0, 127, 0, spk_tlv),
};

/* DAPM widgets — audio path components */
static const struct snd_soc_dapm_widget wm8960_dapm_widgets[] = {
    SND_SOC_DAPM_INPUT("LINPUT1"),
    SND_SOC_DAPM_INPUT("RINPUT1"),
    SND_SOC_DAPM_OUTPUT("HP_L"),
    SND_SOC_DAPM_OUTPUT("HP_R"),
    SND_SOC_DAPM_DAC("Left DAC", "Playback",
                     WM8960_POWER2, 8, 0),
    SND_SOC_DAPM_DAC("Right DAC", "Playback",
                     WM8960_POWER2, 7, 0),
    SND_SOC_DAPM_PGA("Left Output Mixer", SND_SOC_NOPM, 0, 0, NULL, 0),
};

/* DAPM routes — signal flow */
static const struct snd_soc_dapm_route wm8960_routes[] = {
    { "Left Output Mixer", NULL, "Left DAC" },
    { "HP_L", NULL, "Left Output Mixer" },
    { "HP_R", NULL, "Right Output Mixer" },
};
```

---

## 4.5 DAPM — Dynamic Audio Power Management

```
DAPM Power Domains:

When user plays audio:
  ┌──────┐     ┌──────────┐     ┌──────────┐     ┌──────┐
  │ I2S  │────►│ Left DAC │────►│ Output   │────►│ HP_L │
  │ Input│     │          │     │ Mixer    │     │      │
  │      │     │ POWERED  │     │ POWERED  │     │POWERED
  └──────┘     └──────────┘     └──────────┘     └──────┘

When audio stops, ONLY active path powered down:
  ┌──────┐     ┌──────────┐     ┌──────────┐     ┌──────┐
  │ I2S  │  X  │ Left DAC │  X  │ Output   │  X  │ HP_L │
  │ Input│     │          │     │ Mixer    │     │      │
  │      │     │ OFF      │     │ OFF      │     │ OFF  │
  └──────┘     └──────────┘     └──────────┘     └──────┘

DAPM key principle:
  - Traces active audio paths from source to sink
  - Powers up only widgets in active paths
  - Powers down widgets when no path needs them
  - Automatic — no user intervention needed
  - Reduces power consumption dramatically on battery devices
```

---

## 4.6 PCM Data Flow

```
PCM Data Flow (Playback):

User Space:                     Kernel:
┌─────────────┐                ┌──────────────────────────┐
│ Application │                │                          │
│ writes PCM  │───write()─────►│ ALSA PCM Core            │
│ data to     │   or mmap      │ ├── Ring buffer (DMA)    │
│ /dev/snd/   │                │ │   hw_ptr (HW position) │
│  pcmC0D0p   │                │ │   appl_ptr (app pos)   │
│             │                │ ├── Period interrupt      │
│             │◄──poll()───────│ │   (wake app for more)  │
│             │   when space   │ └── DMA to I2S FIFO      │
└─────────────┘   available    └──────────┬───────────────┘
                                          │ DMA
                                ┌─────────▼──────────┐
                                │ I2S Controller     │
                                │ Serializes to      │
                                │ BCLK + LRCLK + DATA│
                                └─────────┬──────────┘
                                          │ I2S bus
                                ┌─────────▼──────────┐
                                │ Audio Codec (DAC)  │
                                │ Digital → Analog   │
                                │ → Headphone/Speaker│
                                └────────────────────┘

Ring Buffer:
  ┌───────┬───────┬───────┬───────┬───────┐
  │Period0│Period1│Period2│Period3│Period4│
  └───────┴───────┴───────┴───────┴───────┘
     ▲                       ▲
     └── hw_ptr              └── appl_ptr
     (DMA reading)           (app writing)
```

---

## 4.7 User-Space ALSA Commands

```bash
# List sound cards
$ cat /proc/asound/cards
$ aplay -l

# Playback
$ aplay -D hw:0,0 -f S16_LE -r 48000 -c 2 audio.wav

# Record
$ arecord -D hw:0,0 -f S16_LE -r 48000 -c 2 -d 5 recording.wav

# Mixer controls
$ amixer -c 0 contents
$ amixer -c 0 set 'Headphone' 80%
$ amixer -c 0 set 'Headphone' mute

# Detailed info
$ cat /proc/asound/card0/pcm0p/info
$ cat /proc/asound/card0/codec#0
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| ALSA core | sound/core/ | PCM, control, timer |
| ASoC core | sound/soc/soc-core.c | Card/DAI/component |
| ASoC DAPM | sound/soc/soc-dapm.c | Dynamic power management |
| ASoC PCM | sound/soc/soc-pcm.c | PCM operations |
| Codec drivers | sound/soc/codecs/ | WM8960, TLV320, etc. |
| Platform drivers | sound/soc/`<vendor>`/ | I2S/DMA for each SoC |

---

## Interview Questions

**Q1: Explain the ASoC three-driver model.**
A: ASoC separates audio into three drivers: (1) **Machine driver** — board-specific, defines which codec connects to which CPU DAI, clock routing, and GPIO amp controls. (2) **Platform/CPU DAI driver** — SoC-specific, handles I2S/TDM controller and DMA engine for PCM data transfer. (3) **Codec driver** — codec chip-specific, controls DAC/ADC registers, mixer controls, DAPM power widgets. The machine driver binds them together via `snd_soc_dai_link`. This separation allows reuse: same codec driver works on different SoCs, same platform driver works with different codecs.

**Q2: What is DAPM and why is it important?**
A: DAPM (Dynamic Audio Power Management) automatically manages power states of audio path components. It traces active audio routes from source (I2S input) to sink (headphone/speaker output). When a stream starts, DAPM powers up only the widgets in the active path. When streaming stops, it powers them down. This is done without user intervention by tracking widget connections (routes) and stream state. It's critical for battery-powered devices — a codec may have 20+ power domains but only 5 need to be on for headphone playback.

**Q3: How does DMA-based PCM streaming work in ALSA?**
A: The PCM buffer is a ring buffer in DMA-capable memory divided into periods. The DMA engine reads data from the buffer and feeds it to the I2S FIFO. Two pointers track state: `hw_ptr` (where DMA is reading) and `appl_ptr` (where the application has written). When DMA finishes a period, a period-elapsed interrupt fires, ALSA updates `hw_ptr` and wakes user space to provide more data. The application writes PCM samples to the ring buffer and advances `appl_ptr`. If `appl_ptr` meets `hw_ptr`, an underrun (XRUN) occurs.

---

## Summary

- ALSA provides the Linux audio subsystem: PCM, mixer controls, timer
- ASoC splits audio into machine/platform/codec drivers for embedded reuse
- The machine driver defines board wiring via `snd_soc_dai_link`
- DAPM automatically powers audio widgets on active paths only
- PCM data flows through DMA ring buffers with period interrupts
- Codec drivers expose mixer controls and DAPM widgets for power optimization

---

*Next: [Chapter 5 — Graphics Framework (DRM/KMS)](Chapter_05_DRM_KMS.md)*
