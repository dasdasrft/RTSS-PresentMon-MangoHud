# RTSS PresentMon MangoHud Overlay

A customized, low-saturation overlay layout for RivaTuner Statistics Server (RTSS), integrating Intel PresentMon 2.0 telemetry with no external software requirements.

## Features

- **Matte Color Palette**: Low-saturation, eye-friendly tones (Nord-inspired sage green, warm sand, and muted slate).
- **Standalone Telemetry**: Directly uses RTSS native HAL and built-in `LHMDataProvider` to read GPU VRAM clock and CPU Vcore voltage without third-party tools.
- **Intel PresentMon 2.0 Integration**:
  - Bottleneck detection: `GPU BOUND`, `CPU BOUND`, or `BALANCED`.
  - Presentation mode: `Direct Flip` or `Windowed`.
  - Animation error tracking.
- **Grid Layout**: Dual-column right alignment for 1% Low and 0.1% Low frame rates.
- **Frametime Graph**: Framed bottom graph with interior margins.

## Quick Start

1. Copy `presentmon_mangohud.ovl` to your RTSS overlays folder:
   `C:\Program Files (x86)\RivaTuner Statistics Server\Plugins\Client\Overlays\`
2. Open RTSS -> **Setup** -> **Plugins** -> enable `OverlayEditor.dll`.
3. Open OverlayEditor -> **Layout** -> **Load** -> select `presentmon_mangohud.ovl`.
4. Click **Layout** -> **Save**.

## License

MIT
