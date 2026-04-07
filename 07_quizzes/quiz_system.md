# Quiz: System

Test your knowledge of package design, PCB routing, and system-level LPDDR considerations.

---

**1.** What does PoP stand for?
- A) Point-of-Presence
- B) Package-on-Package
- C) Power-over-Pin
- D) Protocol-over-Physical

**2.** What is the typical TMV (Through-Mold Via) inductance?
- A) 1-5 pH
- B) 50-150 pH
- C) 1-5 nH
- D) 50-150 nH

**3.** Why is the PoP signal path shorter than discrete PCB routing?
- A) PoP uses faster materials
- B) The DRAM is stacked directly on the SoC package (2-5 mm vs 20-50 mm)
- C) PoP uses wider traces
- D) PoP eliminates the need for impedance matching

**4.** What is the main reliability concern for PoP solder joints?
- A) Electromigration
- B) Thermal cycling fatigue from CTE mismatch
- C) Corrosion
- D) Radiation damage

**5.** What is back-drilling used for in PCB via design?
- A) Creating blind vias
- B) Removing the unused via stub to reduce resonance
- C) Increasing via diameter
- D) Adding thermal vias

**6.** What is the typical ball pitch for LPDDR PoP applications?
- A) 0.1 mm
- B) 0.4-0.5 mm
- C) 1.0 mm
- D) 2.0 mm

**7.** What is the RDL (Redistribution Layer)?
- A) A software routing algorithm
- B) Metal layers on the die or package that reroute bump positions
- C) A PCB design standard
- D) A DRAM memory array structure

**8.** What is the primary SI concern with DRAM wire bonds in PoP?
- A) High resistance
- B) High inductance (1-2 nH) creating impedance discontinuity
- C) High capacitance
- D) Low reliability

**9.** For PCB LPDDR routing, what is the target single-ended impedance?
- A) 25 ohm
- B) 50 ohm
- C) 75 ohm
- D) 100 ohm

**10.** What is the purpose of via fencing in package substrates?
- A) Mechanical support
- B) EMI containment by acting as a Faraday cage
- C) Thermal dissipation
- D) Power distribution

**11.** What drives the need for controlled impedance on PCB LPDDR traces?
- A) Manufacturing cost reduction
- B) Minimising reflections and signal degradation at high data rates
- C) Aesthetic requirements
- D) Regulatory compliance only

**12.** What is package warpage?
- A) Signal distortion in the package
- B) Bending of the package substrate from CTE mismatch between materials
- C) Power supply variation in the package
- D) Clock skew in the package routing

**13.** Which PAM signaling level is being considered for LPDDR6?
- A) PAM-2 (NRZ)
- B) PAM-3
- C) PAM-4
- D) PAM-8

**14.** What is the expected maximum data rate for LPDDR6?
- A) 6400 MT/s
- B) 8533 MT/s
- C) 10000-14400 MT/s
- D) 25600 MT/s

**15.** What advanced packaging technology could eliminate PoP for future LPDDR?
- A) QFP (Quad Flat Package)
- B) 3D integration with hybrid bonding or TSVs
- C) DIP (Dual Inline Package)
- D) SOT (Small Outline Transistor)

**16.** What is the typical total PoP stack height for mobile applications?
- A) 0.3 mm
- B) 1.0-2.0 mm
- C) 5.0 mm
- D) 10.0 mm

---

## Answer Key

1. **B** -- PoP = Package-on-Package, a 3D stacking technology.
2. **B** -- TMV inductance is typically 50-150 pH, a significant parasitic for PI and SI.
3. **B** -- PoP stacks DRAM directly on SoC, reducing the signal path from ~20-50 mm to ~2-5 mm.
4. **B** -- Thermal cycling causes solder fatigue from CTE mismatch between the two packages.
5. **B** -- Back-drilling removes the unused portion of a through-hole via to eliminate the stub.
6. **B** -- 0.4-0.5 mm ball pitch is typical for LPDDR PoP applications.
7. **B** -- RDL is a set of thin metal layers that reroute connections between die pads and package bumps.
8. **B** -- Wire bonds have 1-2 nH inductance, creating a major impedance discontinuity and bandwidth limit.
9. **B** -- 50 ohm single-ended (100 ohm differential) is the standard target.
10. **B** -- Via fencing creates a Faraday cage effect, containing EM fields within the substrate.
11. **B** -- Controlled impedance minimises reflections that degrade signal quality at high data rates.
12. **B** -- Warpage is bending from CTE mismatch, affecting solder joint reliability in PoP assembly.
13. **C** -- PAM-4 is being considered for LPDDR6 to double the data rate per symbol.
14. **C** -- LPDDR6 targets 10000-14400 MT/s per pin.
15. **B** -- 3D integration (hybrid bonding, TSVs) could replace PoP with direct die stacking.
16. **B** -- Typical PoP stack height is 1.0-2.0 mm for mobile applications.
