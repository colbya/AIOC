# AIOC Feature Brainstorming: Potential Enhancements

This document explores potential new features for the AIOC firmware based on hardware capabilities, ham radio activities, and market analysis.

---

## Hardware Resources Available

| Resource | Available | Constraint Level |
|----------|-----------|------------------|
| Flash | ~50 KB free | Moderate |
| **RAM** | **~4 KB free** | **Tight - main limitation** |
| GPIO pins | 36+ unused | Abundant |
| Timers | 5+ free (TIM1, TIM5, TIM7, TIM8, etc.) | Abundant |
| I2C/SPI | 2 ports each, unused | Available |
| ADC channels | 14+ unused | Abundant |
| DAC | DAC2 unused | Available |
| CPU headroom | ~30-50% at 72MHz | Good |
| USB power | ~300mA available | Excellent |

**Key constraint**: RAM limits buffer-heavy features (no complex DSP or large packet buffers).

---

## Feature Ideas by Category

### Category 1: Beacon & Identification Modes (Fox Hunt Extensions)

**1.1 Multi-Mode Beacon**
- Add AFSK (1200 baud) beacon alongside morse
- Transmit callsign as both CW and APRS packet
- Alternating modes on configurable schedule
- *Complexity: Medium (AFSK generation similar to morse)*

**1.2 Contestable Fox Hunt**
- Multiple beacon messages (hunt stages)
- GPS coordinate encoding (if external GPS added via UART)
- Signal strength indicator tones
- *Complexity: Low (settings expansion)*

**1.3 Emergency Beacon Mode**
- SOS morse pattern
- Alternating frequencies (if radio supports)
- Maximum power/continuous duty cycle handling
- *Complexity: Low*

---

### Category 2: Audio Processing & VOX

**2.1 Intelligent VOX (Voice-Operated Switch)**
- Audio level threshold detection (already have VPTT/VCOS)
- Configurable attack/release times
- Anti-trip filtering (ignore brief noise spikes)
- *Complexity: Low (extend existing VPTT)*

**2.2 CTCSS/DCS Tone Injection**
- Generate sub-audible tones (67-254 Hz CTCSS)
- Mix with transmit audio
- Useful for repeater access without radio support
- *Complexity: Medium (requires tone generation + mixing)*

**2.3 Roger Beep / Courtesy Tone**
- Configurable end-of-transmission tone
- Multiple tone patterns
- Delay after PTT release
- *Complexity: Low*

**2.4 Audio Compressor/Limiter**
- Automatic gain control on TX audio
- Prevent over-deviation
- *Complexity: Medium (real-time processing, RAM-limited)*

---

### Category 3: Packet Radio & APRS

**3.1 Hardware KISS TNC**
- Implement KISS protocol framing in firmware
- Offload packet framing from host
- Compatible with Dire Wolf, APRS clients
- *Complexity: High (protocol stack, RAM for buffers)*

**3.2 APRS Digipeater Mode**
- Store-and-forward one packet at a time
- Simple path manipulation (WIDE1-1)
- Standalone operation without computer
- *Complexity: High (requires AX.25 decode, ~2-4KB RAM)*

**3.3 APRS Tracker Integration**
- Serial input for GPS NMEA sentences
- Generate position beacons automatically
- Configurable beacon rate, path, comment
- *Complexity: High (GPS parsing, AFSK generation)*

---

### Category 4: Radio Control & CAT

**4.1 CAT Command Passthrough**
- Second UART for radio CAT port
- USB serial multiplexing (multiple virtual ports)
- Transparent bridge mode
- *Complexity: Medium (requires USB CDC expansion)*

**4.2 Frequency Display/Logging**
- Parse CAT frequency data
- Log to settings/flash
- Report via HID
- *Complexity: Medium*

**4.3 Band-Specific Audio Profiles**
- Different EQ/gain per band
- Auto-switch based on CAT frequency
- *Complexity: Medium*

---

### Category 5: Measurement & Monitoring

**5.1 S-Meter / RSSI Input**
- Use spare ADC channel for signal strength
- Report via USB HID or serial
- Squelch threshold based on signal level
- *Complexity: Low (just ADC reading)*

**5.2 Power/SWR Meter Input**
- ADC inputs for forward/reflected power
- Calculate SWR
- Report via USB
- Alarm on high SWR
- *Complexity: Low-Medium*

**5.3 Battery/Voltage Monitor**
- Monitor supply voltage
- Low battery warning
- Report via USB
- *Complexity: Low*

**5.4 Temperature Logging**
- Use STM32 internal temp sensor
- Log operating temperature
- Thermal protection
- *Complexity: Low*

---

### Category 6: External Hardware Integration

**6.1 I2C Sensor Hub**
- Temperature/humidity sensors
- Barometric pressure (weather station)
- Real-time clock with battery backup
- *Complexity: Low-Medium per sensor*

**6.2 External Display (I2C OLED)**
- Show PTT status, audio levels
- Frequency display (if CAT available)
- Settings menu
- *Complexity: Medium (display driver, RAM for framebuffer)*

**6.3 Rotary Encoder Input**
- Menu navigation
- Volume control
- Frequency tuning (if CAT)
- *Complexity: Low*

**6.4 SD Card Logging (SPI)**
- Log audio (low quality due to RAM)
- Log packets/events
- Store configuration backups
- *Complexity: High (filesystem, RAM buffers)*

---

### Category 7: Digital Mode Assistance

**7.1 Mode Detection**
- Detect FT8/FT4/WSPR/CW timing patterns
- Auto-adjust audio levels per mode
- LED indication of detected mode
- *Complexity: Medium-High (pattern recognition)*

**7.2 CW Decoder (Simple)**
- Detect CW timing, report via serial
- No full decode, just timing data for host
- *Complexity: Medium*

**7.3 PTT Sequencing**
- Configurable TX delay after PTT assert
- Amplifier/antenna relay timing
- Prevent hot-switching
- *Complexity: Low*

---

### Category 8: Network & Linking (AllStarLink/EchoLink)

**8.1 COS (Carrier Operated Switch) Enhancement**
- Better squelch tail handling
- Configurable hang time
- CTCSS decode for COS (if tone decoder added)
- *Complexity: Low-Medium*

**8.2 DTMF Decoder**
- Decode DTMF tones from received audio
- Report digits via USB serial
- Enable remote control of node
- *Complexity: Medium (Goertzel algorithm, ~1KB RAM)*

**8.3 Courtesy Tone Generator**
- Configurable multi-tone sequences
- Timing for repeater linking
- Different tones for local vs linked
- *Complexity: Low-Medium*

---

### Category 9: Novel/Experimental Ideas

**9.1 Bluetooth Audio Bridge**
- Add HC-05/HM-10 Bluetooth module via UART
- Wireless audio to phone/tablet
- Wireless PTT
- *Complexity: High (Bluetooth profile complexity)*

**9.2 LoRa/Meshtastic Gateway**
- Bridge LoRa APRS to traditional APRS
- Add SX1276/RFM95 module via SPI
- *Complexity: High*

**9.3 Remote Head Mode**
- Control radio over longer USB cable
- Buffer/repeat audio
- Remote PTT with status feedback
- *Complexity: Medium*

**9.4 Training/Practice Mode**
- Loopback audio without TX
- CW practice oscillator (already have morse!)
- Practice keying without transmitting
- *Complexity: Low*

**9.5 Contest Keyer**
- Programmable CW messages
- Paddle input via GPIO
- Speed adjustment
- *Complexity: Low-Medium*

---

## Recommended Starting Points

Based on feasibility (RAM constraints, implementation complexity, user value):

### Tier 1: Low Complexity, High Value
1. **Roger Beep** - Simple, immediately useful
2. **Intelligent VOX** - Extends existing VPTT
3. **S-Meter ADC Input** - Just add one ADC channel
4. **PTT Sequencing** - Important for amplifier users
5. **Temperature/Voltage Monitor** - Already have internal sensors

### Tier 2: Medium Complexity, High Value
1. **CTCSS Tone Injection** - Opens up repeater access
2. **DTMF Decoder** - Enables remote control
3. **Contest CW Keyer** - Leverages existing morse code
4. **External I2C Sensors** - Expandable platform

### Tier 3: Higher Complexity, Specialized Value
1. **KISS TNC Mode** - Major feature, RAM-limited
2. **CAT Passthrough** - Requires USB stack changes
3. **APRS Tracker** - Needs GPS + AFSK
4. **I2C OLED Display** - RAM for framebuffer

---

## Questions to Consider

1. **Target use cases**: Field portable? Fixed station? Emergency? Contest?
2. **Hardware modifications**: Willing to add external components (GPS, display, sensors)?
3. **Software compatibility**: Need to maintain compatibility with existing tools (Dire Wolf, CHIRP)?
4. **Standalone vs. computer-dependent**: Should features work without a connected computer?

---

*[Back to main guide](../new_dev_intro.md)*
