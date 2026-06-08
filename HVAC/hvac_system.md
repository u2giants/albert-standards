# HVAC System Architecture: Daikin R-32 (VRV 5 vs. MXM)

## System Goal
Establish a high-efficiency, R-32 cooling system for an 8-zone, 6,000 sq ft custom home in Brooklyn with staggered-stud, high-performance exterior walls. Heating is managed independently via gas-boiler radiant floors, making heat recovery VRF unnecessary. 

This document evaluates the two finalized options: the **Daikin VRV 5 (Single-Phase)** and the **Daikin MXM (Aurora Series)**. 

---

## 1. Daikin VRV 5 (Single-Phase)
*A commercial-grade mini-VRF system scaled for luxury residential. The superior choice for complex ducted layouts.*

### Piping: Trunk-and-Branch (Refnet)
- **Architecture:** A single pair of main copper lines enters the building and branches off at engineered Y-joints (Refnets) only near the specific zone. 
- **Structural Advantage:** Drastically reduces total copper volume and the size of mechanical chases required. Minimizes mechanical flare connections at the outdoor unit to just two, substantially lowering the statistical risk of long-term refrigerant leaks.
- **Maintenance Note:** If a leak does occur, the interconnected pressure tree makes it harder to isolate the specific zone compared to home-run piping, requiring a full-system leak search.

### Ductwork & Framing (FXMQ / FXSQ Series Air Handlers)
- **High Static Pressure:** Blower units can push up to 1.1" of static pressure. 
- **Design Impact:** This allows the units to be grouped in dedicated mechanical closets rather than dropped into the ceiling joists above individual rooms. It supports complex duct layouts, restrictive linear slot diffusers, and deep MERV-13 filter housings.
- **Acoustic Integrity:** By keeping the blowers out of the room envelopes, living spaces remain whisper-quiet, and the acoustic barrier of the 7/8" Weyerhaeuser Edge Gold T&G subfloors is preserved (no need to carve massive return-air cavities between floors).

### Home Assistant Integration
- **Protocol:** Encrypted P1/P2 commercial communication bus.
- **Requirements:** Bypassing cloud dependencies requires a closed-source **Daikin DKN Plus Interface (AZAI6WSPDKC)** hardwired to each indoor unit (~$250–$300 per zone).
- **Control:** Provides a local REST API or Modbus connection to Home Assistant, allowing digital setpoint control while preserving the native variable-speed inverter logic.

---

## 2. Daikin MXM (Aurora Series)
*A traditional residential multi-split system. Excellent for straightforward retrofits, but architecturally restrictive for a luxury new build.*

### Piping: Home-Run 
- **Architecture:** Every indoor unit requires its own dedicated pair of copper lines running continuously from the room back to the outdoor condenser. 
- **Structural Disadvantage:** For 8 zones split across two condensers, this requires routing 16 individual copper lines through the framing to the exterior. It creates a massive "copper spaghetti" bottleneck in mechanical chases and introduces 16 mechanical flare connections exposed to weather/vibration, increasing leak probability.
- **Maintenance Advantage:** If a line is pierced by a contractor, an HVAC technician can easily pressure-test individual lines at the condenser to isolate the exact zone.

### Ductwork & Framing (FDMQ / FDXS Series Air Handlers)
- **Low-to-Medium Static Pressure:** Blowers max out at ~0.6" static pressure.
- **Design Impact:** These units cannot push air through long, complex duct runs, heavily restrictive linear diffusers, or dense air filtration.
- **Acoustic Integrity:** Must be installed in a soffit, bulkhead, or ceiling joist directly adjacent to the room they serve, introducing ambient blower motor noise directly into the living envelope. 

### Home Assistant Integration
- **Protocol:** Standard S21 residential communication port.
- **Requirements:** Highly open-source friendly. Requires a ~$15 custom ESP32 board (like Faikin or Daikin2MQTT) plugged directly into the S21 header.
- **Control:** Provides 100% local, cloud-free control over MQTT directly into Home Assistant, preserving inverter efficiency without the massive hardware tax of the VRV enterprise modules. Easily drops into an existing Docker/networking homelab.

---

## Systems Ruled Out
* **Mitsubishi SMART MULTI:** Excluded due to heavy reliance on R-410A refrigerant in the single-phase tier, bulky intermediate Branch Box distribution, and the notorious instability of Kumo Cloud Wi-Fi nodes.
* **3-Phase Commercial VRF:** Full commercial 10-ton capacity is unnecessary. Splitting the home across two single-phase condensers provides built-in redundancy, and the tight staggered-stud thermal envelope requires significantly fewer BTUs.
* **Daikin VRV Life:** Engineered to natively integrate with a ducted gas furnace; unnecessary due to the radiant floor system.
