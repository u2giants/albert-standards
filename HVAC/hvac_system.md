# HVAC System Report - Brooklyn Single-Family Residence

Last updated: June 13, 2026

This file is intended to replace `HVAC/hvac_system.md` in `https://github.com/u2giants/albert-standards/HVAC`. It documents the current HVAC design direction, the decisions made so far, the reasons behind those decisions, the constraints that must not be forgotten, and the open engineering questions that still require licensed engineer verification.

> Important: this is an owner/design-direction report, not a signed engineering document. Final loads, equipment selections, drawings, controls, and code compliance must be completed by the licensed mechanical/HVAC engineer using Elite RHVAC, Daikin selection software, current manufacturer submittals, and current NYC code.

---

## 1. Project Summary

The project is a major HVAC redesign for a roughly 5,400 sq. ft., four-floor single-family house in Brooklyn, New York. The earlier design assumed natural gas would not be available and therefore centered around an air-to-water heat pump, hydronic fan coils, chilled water, and a buffer tank. Natural gas is now available, so the design direction has changed.

The new preferred design is a hybrid system:

- Daikin VRV-S R32 and/or Daikin VRV-S Aurora R32 for cooling and selected forced-air heating.
- A high-efficiency condensing natural-gas boiler for radiant floor heating and hydronic snow melt.
- Separate domestic hot water, likely with cascaded gas tankless water heaters.
- ERV-based ventilation with distributed small ductwork.
- Carefully engineered kitchen makeup air for a 600 CFM induction hood.

The design goal is high comfort, high efficiency, good serviceability in NYC, low noise, and realistic construction within severe space constraints.

---

## 2. House Details

| Item | Current Understanding |
| --- | --- |
| Location | Brooklyn, NY |
| Climate zone | Climate Zone 4A |
| Approximate size | About 5,400 sq. ft. |
| Floors | Basement, 1st floor, 2nd floor, 3rd floor |
| Occupants | 4 adults + 3 children |
| Basement construction | ICF remains in basement |
| Above-grade exterior walls | 2x8 cold-formed steel studs, 2 in. exterior Comfortboard/mineral wool, 5 in. closed-cell spray foam in cavity |
| Joists | Cold-formed steel, 16 in. O.C. |
| Joist openings | Max. 8 in. diameter web openings |
| Interior partitions | 2x4 dimensional lumber; vertical risers/line sets must fit 3.5 in. wall depth unless otherwise coordinated |
| Register rule | High sidewall or ceiling only. No floor, low sidewall, or toe-kick registers. |
| Drop ceilings | Limited; equipment must fit in planned AC closets, mechanical rooms, soffits, or approved ceiling pockets |
| Snow melt | Approx. 900 sq. ft. hydronic exterior snow melt, occasional use |
| DHW occupancy basis | 7 occupants |

### Envelope Modeling Notes

The HVAC engineer still needs the final wall, roof, window, slab, and basement assemblies for Manual J. The structural engineer may own wall design, but the HVAC engineer must not use generic/default walls. The above-grade CFS walls must be modeled with steel-stud thermal bridging corrections, not simple nominal cavity R-values.

The basement should be modeled as ICF, including thermal mass assumptions if Elite RHVAC supports that accurately.

---

## 3. Background: What Changed

### Old design direction

The original engineer scope assumed:

- Air-to-water heat pump generation.
- Chilled water in summer.
- Hot water in winter.
- Central buffer tank.
- Hydronic fan coils / FCUs for cooling.
- Basement fan coil for forced-air heat.
- Possible SpacePak Solstice R32 AWHP.
- SANCO2/DHW AWHP boost concept for snow melt.
- Rack/shelf AC and exhaust fan for IT room.

### Why the old direction is being abandoned

The old direction was designed around the assumption that natural gas was unavailable. Now that natural gas is available, the old design is unnecessarily complex and creates avoidable risk:

- Chilled-water fan coils add complexity compared with VRV cooling.
- AWHP capacity and water temperature concerns complicate snow melt.
- Buffer-tank and hydronic-cooling design increases pumping, controls, condensation, insulation, and service complexity.
- Using DHW heat-pump equipment to assist snow melt is overly complex.
- Rack AC and IT exhaust fan approach is less elegant than dedicated VRV cooling.
- Gas boiler is a better fit for radiant heat and occasional snow melt.

### New direction

Use VRV for cooling and forced-air heating where needed, and a gas condensing boiler for hydronic heating/snow melt.

---

## 4. What We Want to Accomplish

The HVAC system should accomplish the following:

1. Provide reliable cooling to all four floors.
2. Provide excellent radiant comfort on Floors 1, 2, and 3.
3. Provide forced-air heat to the basement without hydronic air handlers.
4. Keep IT Room 008 and Gym cooled 12 months per year.
5. Avoid simultaneous heating/cooling complexity by assigning IT/gym to the same outdoor unit as radiant-heated upper floors.
6. Avoid floor, low-wall, and toe-kick registers.
7. Respect tight joist, wall, and equipment-room limitations.
8. Use equipment with strong NYC support, parts availability, and contractor familiarity.
9. Avoid overly complex one-off mechanical concepts that are hard to service.
10. Keep kitchen makeup air comfortable without defaulting to a large electric resistance heater unless necessary.
11. Provide a boiler strategy that can handle radiant heat and at least occasional snow-melt usage.
12. Produce clear Elite RHVAC, Manual J, Manual S, Manual D, hydronic, and plan-overlay deliverables.

---

## 5. Preferred System Architecture

### 5.1 Cooling and forced-air heating

Basis of design:

- Daikin VRV-S R32 and/or Daikin VRV-S Aurora R32.
- Two outdoor systems.
- Ducted VRV indoor units for main comfort zones.
- Dedicated year-round cooling for IT room and gym.

### 5.2 Outdoor unit split

| Outdoor Unit | Serves | Operating Role |
| --- | --- | --- |
| Outdoor Unit #1 | 2nd floor, 3rd floor, IT room, Gym | Cooling-capable year-round. Floors 2 and 3 are heated by radiant floor, so the outdoor unit does not need to provide winter heat to those floors. IT/gym must still be able to cool in winter. |
| Outdoor Unit #2 | Basement and 1st floor | Cooling in summer. Heating in winter, primarily for basement because 1st floor has radiant heat. May have spare winter heat-pump capacity that could potentially help temper kitchen MUA if engineered correctly. |

### 5.3 Why this split avoids heat recovery

A heat-recovery VRV system is usually used when one part of a building needs heating while another simultaneously needs cooling on the same refrigerant network. This project can avoid that because:

- The 2nd and 3rd floors have radiant heat.
- IT room and gym can be on Outdoor Unit #1, which can remain cooling-capable in winter.
- Basement is the only main floor expected to need VRV heating in winter.
- 1st floor has radiant heat, so Outdoor Unit #2 winter heating demand is mostly basement.

Therefore, the project does not need simultaneous heating/cooling if the zones are assigned correctly.

---

## 6. Main Comfort Zones

The expected main comfort zoning is eight zones, plus dedicated IT/gym treatment if not included in the main eight.

| Zone | Area Served | Likely Equipment Location | Design Concerns |
| --- | --- | --- | --- |
| B1 | Basement front | Basement front mechanical room | VRV ducted unit; likely easiest place for larger/higher-static equipment. |
| B2 | Basement rear | Basement rear AC room, approx. 5 ft x 3 ft 9 in x 8 ft | Tight; compact/high-static unit choice must be proven with service access. |
| 1F-front | First-floor front | Basement front mechanical room | Unit pushes air up from basement to high sidewall/ceiling registers. Needs enough static. No low wall/toe-kick/floor registers. |
| 1F-rear | First-floor rear | Basement rear AC room or pantry/mudroom ceiling pocket | Prefer basement if supply/return/static can work. Pantry/mudroom ceiling pocket is possible alternate. |
| 2F-front | Second-floor front | Third-floor front AC room, approx. 4 ft x 3 ft | Long run/drop from 3rd to 2nd floor. Static and service access are concerns. |
| 2F-rear | Second-floor rear | Third-floor rear AC room, approx. 6 ft x 4 ft | Long run/drop from 3rd to 2nd floor, but rear AC room has more space. |
| 3F-front | Third-floor front | Third-floor front AC room | Shorter duct runs; compact unit likely if service access works. |
| 3F-rear | Third-floor rear | Third-floor rear AC room | Shorter duct runs; compact unit likely if service access works. |
| IT-008 | IT room | Dedicated indoor unit on Outdoor Unit #1 or dedicated alternative | Year-round cooling. Old rack AC/exhaust concept deleted. |
| Gym | Gym | Dedicated indoor unit on Outdoor Unit #1 or dedicated alternative | Year-round cooling. |

Approximate zone areas:

- Most zones are about 800 sq. ft.
- Third-floor zones are about 500 sq. ft. each.
- Final equipment cannot be sized by area; it must be sized by room-by-room Manual J and Daikin corrected capacity.

---

## 7. Equipment Location Constraints

### Basement rear AC room

Approximate size: 5 ft x 3 ft 9 in x 8 ft high.

Potential use:

- Basement rear zone.
- Possibly rear 1st-floor zone if duct risers, return path, and static are acceptable.

Concerns:

- Very limited width/depth.
- Service clearance and filter access must be shown on plan.
- Stacking two compact ducted units may be possible, but must be proven.
- Stacking a multi-position air handler with a concealed ceiling unit above it is likely unrealistic.

### Basement front mechanical room

Likely the best place for:

- Basement front zone.
- First-floor front zone.
- Larger or higher-static ducted VRV indoor units.
- More complicated duct transitions.

### First-floor pantry/mudroom

Potential alternate location for rear first-floor ceiling-mounted concealed unit if basement-to-first-floor routing is not practical.

### Third-floor rear AC room

Approximate size: 6 ft x 4 ft.

Likely best upper-floor AC room. Potentially serves:

- 3rd-floor rear zone.
- 2nd-floor rear zone.

### Third-floor front AC room

Approximate size: 4 ft x 3 ft.

Potentially serves:

- 3rd-floor front zone.
- 2nd-floor front zone.

Concerns:

- Very tight.
- Service clearance must be proven.
- Longer drop to 2nd floor may require more static.

---

## 8. Duct and Register Rules

### Absolute register restrictions

Do not use:

- Floor registers.
- Toe-kick registers.
- Low sidewall registers.

Use only:

- High sidewall registers.
- Ceiling registers.

### Why this matters

Some first-floor zones may be served by units located in the basement. Without low sidewall, floor, or toe-kick registers, those units must have enough static pressure and duct routing ability to push conditioned air up to high sidewall or ceiling delivery points.

Therefore, first-floor zones served from the basement likely need high-static ducted VRV indoor units or another Daikin R32-compatible ducted unit proven by Manual D.

### Joist restrictions

- Joists are 16 in. O.C.
- Unit bodies such as a Daikin FXSA-style concealed ducted unit do not fit between joists because the body width is far larger than the joist spacing.
- Ducts, line sets, condensate, and piping through joists must fit max. 8 in. web openings.

### Static pressure implications

The engineer must calculate total external static pressure for every ducted unit, including:

- Supply duct.
- Return duct.
- Filter.
- Grilles/registers.
- Balancing dampers.
- Elbows.
- Vertical risers.
- Firestopping transitions.
- Register boots.
- Long drops from 3rd floor to 2nd floor.

Medium-static concealed units may work for compact local zones. Basement-to-first-floor zones and third-to-second-floor zones may need high-static units.

---

## 9. Daikin VRV Selection Direction

### Daikin VRV-S R32

Daikin VRV-S R32 is the baseline small-scale VRV/VRF option. Current Daikin/VRV Drive information indicates:

- 2, 3, 4, and 5 ton models.
- Cooling operation range around 23 F to 122 F.
- Heating operation range around -4 F to 60 F.
- Low-ambient cooling options.
- Selection must be verified in Daikin WebXpress with all derates.

### Daikin VRV-S Aurora R32

Daikin VRV-S Aurora R32 is more attractive where winter heating matters. Published Daikin Aurora information indicates:

- Heating performance down to lower outdoor temperatures than standard VRV-S.
- Cooling operation range still around 23 F to 122 F in published Aurora literature.

### Recommended use

| Application | Recommended Direction |
| --- | --- |
| Outdoor Unit #1 serving 2nd/3rd + IT/gym | VRV-S R32 may be sufficient, but engineer must prove winter cooling for IT/gym. |
| Outdoor Unit #2 serving basement/1st | Strongly evaluate VRV-S Aurora R32 because it handles winter heating better and basement relies on VRV heat. |

### Indoor unit warning

Do not casually name older Daikin model families unless the engineer verifies R32 compatibility. Final indoor units must be selected from current Daikin R32-compatible submittals and Daikin selection software.

---

## 10. Hydronic Radiant Heating

Radiant floor heat is the primary heating system for:

- 1st floor.
- 2nd floor.
- 3rd floor.

Current radiant floor build-up:

- 1.5 in. Gypcrete overpour.
- 7/8 in. Weyerhaeuser Edge Gold T&G subfloor.
- PEX tubing.

Engineer must define:

- Tubing size.
- Tubing spacing.
- Loop lengths.
- Manifold locations.
- Flow per loop.
- GPM by zone/floor.
- Head loss.
- Mixing strategy.
- Supply/return water temperatures.
- Max floor surface temperatures.
- Floor covering R-values.
- Controls and thermostats.

---

## 11. Boiler Direction

### Basis-of-design boiler

Call out:

**Navien NFB-200H high-efficiency condensing gas boiler.**

Current product facts to verify in final submittal:

- 199,900 BTU/hr max heating input.
- Approximately 183,000 BTU/hr heating capacity.
- 95% AFUE.
- 15:1 turndown.

### Why this boiler is being called out

Navien is preferred because:

- The project now has natural gas.
- Radiant heat and snow melt are better served by a condensing gas boiler than by the old AWHP/chilled-water concept.
- The NFB-200H gives more capacity than Navien's smaller residential-only boiler options and gives margin for occasional snow-melt usage.
- 15:1 turndown helps reduce short cycling when only small radiant zones are calling.

### Important caution

The NFB-200H should be the basis of design, not a guarantee. The engineer must verify:

- Total radiant load.
- Snow-melt load.
- Smallest active zone load.
- Boiler minimum firing rate versus smallest zone.
- Required supply water temperatures.
- Venting.
- Gas meter/service capacity.
- Whether snow melt needs priority/staging/cascade.

The 900 sq. ft. snow-melt area can dominate the boiler size if designed for aggressive active melting. If the NFB-200H is insufficient for the selected snow-melt performance target, the engineer should recommend staging, priority, slower melt rate, cascaded boilers, or a larger commercial boiler strategy.

---

## 12. Snow Melt

Current concept:

- Approx. 900 sq. ft. exterior hydronic snow-melt area.
- Occasional use.
- Served by the space-heating boiler through a brazed plate heat exchanger.
- Dedicated glycol loop.

Engineer must define:

- Design intent: keep clear during snowfall vs. melt after storm vs. occasional assisted melting.
- BTU/sq. ft. assumption.
- Glycol percentage.
- Supply/return water temperatures.
- Heat exchanger size.
- GPM.
- Pump head.
- Slab/driveway insulation assumptions.
- Controls.
- Priority/staging relative to radiant heating.
- Whether snow melt can temporarily reduce space-heating output.

---

## 13. Domestic Hot Water

Current owner direction:

- DHW is separate from the space-heating boiler.
- Likely two cascaded 199,000 BTU/hr condensing gas tankless water heaters, such as Navien NPE-240A2 or equivalent.
- Occupancy basis: 7 people.

Engineer/plumber must verify:

- Fixture count.
- Simultaneous use assumptions.
- Tub/shower flow requirements.
- Recirculation strategy.
- Gas capacity.
- Venting.
- Condensate neutralization.
- Water quality and maintenance requirements.

---

## 14. Kitchen Ventilation and Makeup Air

### Hood

Current concept:

- 5-burner induction range.
- Pro-style deep-sump hood.
- Remote exterior blower to minimize kitchen noise.
- Target airflow: 600 CFM variable speed.
- Exhaust should vent directly through exterior wall/roof if possible and should not route through floor joists.

### Makeup air location

Target location:

- West Wall of Dinette Room 111.

This is intended to provide mixing and separation from the hood, but final location depends on comfort, duct routing, code separation, and architecture.

### Makeup air options discussed

#### Option A - Electric resistance duct heater

Pros:

- Simple.
- Likely lowest upfront equipment cost.
- Packaged residential MUA products exist.
- Easy controls relative to hydronic.

Cons:

- 10 kW is a large electric load.
- Expensive to operate.
- Adds electrical service burden.
- Owner does not want this as the default.

Potential products to evaluate:

- Fantech MUAS 10 / MUAS 1200 type makeup-air fan system.
- Fantech MUAH 10/10 type 10 kW duct heater.
- Broan MD10TU type automatic makeup-air damper where appropriate.

#### Option B - Gas-fired MUA unit

Decision: **Do not pursue.**

Why:

- Overkill for this residence.
- Adds combustion, venting, and code complexity.
- More equipment complexity than desired.

#### Option C - Hydronic hot-water MUA coil from boiler

Pros:

- Uses gas heat instead of electric resistance.
- Potentially lower operating cost.
- Fits the project now that a boiler exists.

Cons:

- Not necessarily same upfront cost as electric.
- Requires coil, pump or valve, controls, freeze protection, aquastat/sensors, airflow proving, motorized damper, and service access.
- Needs careful freeze-safe design.

Decision:

- Evaluate this after VRV-assisted tempering.

#### Option D - Use Outdoor Unit #2 / VRV system to temper MUA

This is now worth serious evaluation because Outdoor Unit #2 is sized for basement + 1st floor cooling, but in winter it primarily heats the basement because the 1st floor has radiant heat.

Potential benefit:

- The system may have spare winter heating capacity that could temper MUA without electric resistance.

Key concerns:

- Code compliance.
- Control sequence.
- Whether the VRV indoor unit can accept the outside-air volume.
- Mixed-air temperature limits.
- Whether airflow is guaranteed when the hood runs even if thermostat is satisfied.
- Whether MUA load steals comfort capacity from basement/1st floor.
- Noise and drafts.
- Filter and freeze-protection strategy.

Preferred evaluation order:

1. Evaluate VRV-assisted MUA tempering using Outdoor Unit #2 / basement-first-floor system.
2. If not practical, evaluate hydronic hot-water coil from Navien boiler.
3. Use electric duct heater only as fallback after reviewing electrical service and operating cost.
4. Do not use gas-fired MUA.

---

## 15. Ventilation / ERV

Required:

- ASHRAE 62.2 ventilation design.
- ERV with automatic summer bypass / bypass core if appropriate.
- Dedicated rigid small-duct distribution because the old central fan-coil distribution is gone.
- Multiple supply registers per floor; do not dump all ventilation at one point.
- At least 10 ft separation between outdoor air intake and exhaust/contaminant sources where required.
- Bathroom/laundry/local exhaust code verification.

---

## 16. IT Room 008 and Gym

### Old idea deleted

Do not use:

- Shelf/rack AC unit like SRCOOL7KRME as the basis.
- Dedicated exhaust fan from IT room to laundry as the cooling strategy.

### New direction

- IT Room 008 and Gym need cooling 12 months per year.
- They should be assigned to Outdoor Unit #1, which also serves the 2nd and 3rd floors.
- Because the 2nd and 3rd floors have radiant heat, Outdoor Unit #1 can remain cooling-capable in winter.
- If Daikin VRV-S / VRV-S Aurora cannot reliably satisfy winter cooling requirements, engineer must propose a dedicated alternative.

Known IT load note:

- Prior assumption included approximately 3,800 BTU/hr of IT equipment heat. Engineer should verify current equipment load.

---

## 17. What We Decided Not To Do

| Rejected / Deprioritized Option | Why |
| --- | --- |
| Air-to-water heat pump as main generation | No longer needed now that gas is available; more complex for this project. |
| Chilled-water fan coils | More piping, condensation, pumping, insulation, and controls complexity than VRV. |
| Hydronic air handlers | Basement/1st floor need cooling anyway, so VRV ducted units are simpler. |
| DHW heat pump assisting snow melt | Overly complex and not needed with gas boiler. |
| Rack AC + exhaust fan for IT room | Dedicated VRV cooling is cleaner and more integrated. |
| Heat-recovery VRV | Not necessary if IT/gym and radiant-heated upper floors are on Outdoor Unit #1. |
| Low sidewall / toe-kick / floor registers | Owner does not allow them; high sidewall or ceiling only. |
| Gas-fired makeup air unit | Too much combustion/venting/code complexity for this residence. |
| Default 10 kW electric MUA heater | Simple but high electric load; evaluate VRV-assisted or hydronic tempering first. |
| Assuming units fit between joists | Joists are 16 in. O.C.; unit bodies are too wide. |
| Stacking multi-position air handler with concealed unit above | Likely unrealistic in small AC rooms unless engineer proves service and clearance. |

---

## 18. Honorable Mentions / Alternates

### High-velocity small-duct systems

Could solve some routing issues, but not preferred because:

- Noise risk.
- Outlet count requirements.
- Static pressure/sound attenuation sensitivity.
- Less ideal if standard/high-static VRV ducted units can work.

Keep as emergency fallback only.

### Dedicated mini-split for IT/gym

Could be used if VRV-S winter cooling is not acceptable, but it adds another outdoor unit or system. Prefer keeping IT/gym on Outdoor Unit #1 if Daikin confirms reliable low-ambient cooling.

### Cascaded boilers

Could be needed if snow-melt performance target exceeds NFB-200H capability. Not preferred unless calculations justify it.

### Larger commercial boiler

Could be needed for aggressive snow melt. Not preferred unless calculations justify it.

### Electric MUA heater

Probably simplest and cheapest upfront, but not preferred due to electric load and operating cost. Keep as fallback.

---

## 19. Open Engineering Questions

These must be resolved by the engineer:

1. Which exact Daikin R32 outdoor units are selected for Outdoor Unit #1 and #2?
2. Should Outdoor Unit #2 be VRV-S Aurora R32 because it provides winter basement heating?
3. Can Outdoor Unit #1 reliably cool IT/gym in winter at Brooklyn low ambient conditions?
4. What Daikin field settings/accessories are required for low-ambient cooling?
5. Which exact R32-compatible Daikin indoor units fit each zone?
6. Which zones require high-static ducted units?
7. Can 1st-floor high sidewall/ceiling delivery be achieved from basement units without excessive static/noise?
8. Can 2nd-floor zones be served from 3rd-floor AC rooms without excessive static/noise?
9. Do the small AC rooms provide sufficient service access and filter access?
10. What is the final room-by-room Manual J load?
11. What is the total radiant load?
12. What is the basement VRV heating load?
13. What is the IT/gym process load?
14. What is the snow-melt design load?
15. Is the Navien NFB-200H sufficient with snow-melt priority/staging?
16. Is gas service/meter capacity adequate for boiler + tankless DHW + other gas loads?
17. Can Outdoor Unit #2 temper kitchen makeup air in a code-compliant, controllable way?
18. If not, is hydronic MUA coil practical and freeze-safe?
19. If not, what is the minimum acceptable electric MUA heater size and control sequence?
20. Which NYCECC version applies based on filing date?

---

## 20. Engineer Deliverables Checklist

The engineer should provide:

- Elite RHVAC native files.
- Preserved DWG/DXF background layers if imported.
- Custom assemblies embedded in the Elite file.
- Manual J room-by-room loads.
- Manual S VRV equipment selections with Daikin WebXpress reports.
- Manual D duct sizing and static pressure reports.
- Hydronic radiant schedule with GPM and head by zone.
- Boiler sizing report.
- Snow-melt load and heat exchanger report.
- Pump schedule.
- ERV/ventilation calculation.
- Kitchen hood/MUA design.
- Controls sequence.
- Condensate routing plan.
- PDF and DWG plan overlays.
- One-page design criteria summary.

---

## 21. Source References for Current Product / Code Facts

These sources were used for current product/code facts and should be rechecked by the engineer at final design:

- [Daikin VRV-S R-32 Engineering Manual - RXTA-AAVJU](https://daikincomfort.com/docs/default-source/vrv-s-series-r-32/engineering-manual---rxta.pdf)
- [Daikin VRV-S Single Phase Heat Pump product page / VRV Drive](https://myvrvdrive.com/category/vrv/products/rxta_a/)
- [Daikin VRV Aurora air-cooled systems product bulletin](https://daikincomfort.com/docs/default-source/vrv-aurora-heat-recovery/pb-cb-vrvaurora.pdf)
- [Daikin FXSA R32 concealed ducted product data / Daikin Comfort](https://my.daikincomfort.com/product/1ton-r32-msp-concealed-ducted-unit/01tRn0000091WxKIAU)
- [Daikin high-static concealed ducted VRV indoor unit product family](https://daikincomfort.com/products/commercial-systems/variable-refrigerant-volume/vrv-indoor-units/hsp-dc-concealed-ducted-unit)
- [Navien NFB-200H product page](https://www.navieninc.com/products/nfb-200h)
- [NYC DOB Energy Conservation Code page](https://www.nyc.gov/site/buildings/codes/energy-conservation-code.page)
- [Fantech MUAS makeup-air system](https://www.fantech.net/en-us/products/fans-and-accessories/makeup-air-system/muas)
- [Broan MD10TU automatic makeup-air damper](https://www.broan-nutone.com/en-us/accessory/md10tu)

---

## 22. Current Final Direction in One Paragraph

Use two Daikin VRV-S R32 / VRV-S Aurora R32 outdoor systems: Outdoor Unit #1 serves the 2nd floor, 3rd floor, IT room, and Gym and must be capable of year-round cooling; Outdoor Unit #2 serves the basement and 1st floor and provides summer cooling plus winter forced-air heat primarily for the basement. Use hydronic radiant floor heat on Floors 1, 2, and 3, served by a Navien NFB-200H basis-of-design condensing gas boiler that also supports occasional 900 sq. ft. snow melt through a dedicated glycol heat exchanger loop. Do not use hydronic air handlers, chilled-water fan coils, rack AC for IT, low sidewall/toe-kick/floor registers, or gas-fired makeup air. Engineer must prove all loads, static pressures, Daikin compatibility, low-ambient operation, service clearances, MUA strategy, and NYC code compliance.

