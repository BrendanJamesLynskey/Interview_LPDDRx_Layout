# Quiz: Signal Integrity

Test your knowledge of signal integrity and power integrity for LPDDR interfaces.

---

**1.** What is the Nyquist frequency for LPDDR5 at 6400 MT/s?
- A) 1600 MHz
- B) 3200 MHz
- C) 6400 MHz
- D) 12800 MHz

**2.** What causes inter-symbol interference (ISI)?
- A) Power supply noise only
- B) Frequency-dependent loss and reflections from impedance discontinuities
- C) Temperature variation
- D) Clock jitter only

**3.** What is the typical VDDQ ripple budget for LPDDR5?
- A) 1%
- B) 3-5%
- C) 10-15%
- D) 20%

**4.** What is the primary benefit of MIM capacitors for decoupling?
- A) Low cost
- B) High capacitance density with low ESR, effective at high frequencies
- C) No area consumption
- D) Process-independent characteristics

**5.** What does TDR stand for, and what does it measure?
- A) Time-Domain Reflectometry; impedance profile along a signal path
- B) Thermal Design Rating; power dissipation capability
- C) Total Data Rate; bandwidth measurement
- D) Timing Delay Report; setup/hold analysis

**6.** What is ground bounce?
- A) Mechanical vibration of the ground plane
- B) Voltage spike on VSS from inductive di/dt during simultaneous switching
- C) Thermal expansion of ground connections
- D) Electrostatic charging of the ground plane

**7.** What is the target PDN impedance for LPDDR5 at 200 mA peak current and 25 mV noise budget?
- A) 12.5 mohm
- B) 125 mohm
- C) 1.25 ohm
- D) 12.5 ohm

**8.** At what frequency does the on-die capacitance typically become effective for decoupling?
- A) 1 kHz - 1 MHz
- B) 1 MHz - 100 MHz
- C) 500 MHz - 5+ GHz
- D) Above 100 GHz

**9.** What is SSO noise?
- A) Single-Sided Operation noise
- B) Simultaneous Switching Output noise from multiple drivers switching at once
- C) System Signal Oscillation noise
- D) Sub-Surface Oxide noise

**10.** How does supply noise convert to timing uncertainty?
- A) It does not affect timing
- B) Through driver delay modulation and clock jitter from PLL supply sensitivity
- C) Only through thermal effects
- D) Only at DC conditions

**11.** What is the reflection coefficient at a discontinuity where Z changes from 50 to 60 ohm?
- A) 0.091
- B) 0.167
- C) 0.200
- D) 0.500

**12.** What BER target is typical for LPDDR5?
- A) 10^-6
- B) 10^-9
- C) 10^-12
- D) 10^-16

**13.** What tool is used for 3D EM simulation of package structures?
- A) Cadence Innovus
- B) Synopsys ICC2
- C) ANSYS HFSS
- D) Cadence Genus

**14.** What is the primary purpose of spread-spectrum clocking?
- A) Increase data rate
- B) Reduce peak EMI by spreading spectral energy
- C) Improve signal integrity
- D) Reduce power consumption

**15.** How does crosstalk affect the eye diagram?
- A) Only reduces voltage margin
- B) Only reduces timing margin
- C) Reduces both voltage and timing margins
- D) Has no effect on the eye diagram

**16.** What is PDN resonance, and why is it problematic?
- A) Resonance in the data path causing bit errors
- B) Impedance peaking where package inductance resonates with on-die capacitance, amplifying noise
- C) Mechanical resonance causing package failure
- D) Clock resonance causing PLL unlock

---

## Answer Key

1. **B** -- Nyquist frequency = Data Rate / 2 = 6400/2 = 3200 MHz.
2. **B** -- ISI is caused by frequency-dependent channel loss and reflections from impedance mismatches.
3. **B** -- 3-5% (15-25 mV at 0.5V VDDQ) is the typical ripple budget.
4. **B** -- MIM caps offer 10-20 fF/um^2 density with ~20 mohm ESR, effective at GHz frequencies.
5. **A** -- Time-Domain Reflectometry measures the impedance profile by sending a step and observing reflections.
6. **B** -- Ground bounce is the VSS voltage spike from Ldi/dt during simultaneous NMOS switching.
7. **B** -- Z_target = V_noise / I_peak = 25 mV / 200 mA = 125 mohm.
8. **C** -- On-die decoupling (MIM/MOM) is effective from ~500 MHz to 5+ GHz.
9. **B** -- SSO = Simultaneous Switching Output, noise from multiple drivers switching simultaneously.
10. **B** -- Supply noise modulates driver delay and PLL/VCO frequency, creating timing uncertainty.
11. **A** -- Gamma = (60-50)/(60+50) = 10/110 = 0.091.
12. **D** -- LPDDR5 targets BER of 10^-16 (less than 1 error per ~24 hours of operation).
13. **C** -- ANSYS HFSS is the standard tool for 3D EM simulation of package structures.
14. **B** -- SSC reduces peak EMI by modulating clock frequency to spread spectral energy.
15. **C** -- Crosstalk adds both voltage noise and timing jitter, reducing both margins.
16. **B** -- PDN resonance is impedance peaking from package L and die C resonating, amplifying noise at that frequency.
