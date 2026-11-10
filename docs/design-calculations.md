# Design Calculations

Component sizing, loss estimates and small-signal models for the bidirectional buck-boost converter.

Several values — the 470 µH inductor, the bulk capacitors, the MOSFETs — were fixed before the analysis began, so this works in the opposite direction to a clean-sheet design: given the parts, what switching frequency and operating point keep the converter inside its ripple and thermal budgets? Losses that are often dropped at this stage are kept in, because the numbers feed a real board.

---

## 1. Design parameters

### Known

| Parameter | Value |
|---|---|
| Bus voltage V<sub>in</sub> | 48 V |
| Battery voltage V<sub>out</sub> | 24 V ±2 % |
| Inductor L | 470 µH |
| Input capacitance C<sub>in</sub> | 990 µF |
| Output capacitance C<sub>out</sub> | 992 µF |
| MOSFET R<sub>DS(on)</sub> | 9 mΩ at V<sub>GS</sub> = 10 V |
| Inductor DCR | 11.5 mΩ |
| Capacitor ESR | 70 mΩ × 3 parallel = 23.3 mΩ |
| Rated current | 3.5 A nominal |
| MOSFET t<sub>r</sub>, t<sub>f</sub> | 90 ns, 5 ns |

### Targets

| Target | Value |
|---|---|
| Output ripple ΔV<sub>o</sub> | ≤ 1 % peak-to-peak |
| Input ripple ΔV<sub>in</sub> | ≤ 1 % peak-to-peak |
| Inductor ripple Δi<sub>L</sub> | ≈ 20 % of I<sub>o</sub> |
| Efficiency | ≥ 90 % |

### Unknown, to be solved

Duty cycle · switching frequency

> **A note on the inductor.** The 470 µH part cannot carry the 10 A the converter was originally scoped for. At a 30 % ripple allowance its ceiling is about 3.8 A, so the whole analysis is pinned at 3.5 A — roughly 200 W rather than the 240 W originally discussed. The design is sound at that rating; it simply isn't the rating first intended.

---

## 2. Buck mode (battery charging)

### 2.1 Inductor and switching frequency

Over one steady-state cycle the inductor current must return to where it started, so the ripple built during the on-time equals the ripple shed during the off-time:

$$\Delta i_{L(on)} = \Delta i_{L(off)} \qquad (1.1)$$

From the inductor relation $V_L = L\,di/dt$ with $dt = t_{on}$:

$$L = \frac{V_{L(on)} \times T_{on}}{\Delta i_L} \qquad (1.4)$$

Around the loop, including the conduction drops across each switch:

$$V_{L(on)} = V_{in} - V_{DS(q1)} - V_{out} \qquad (1.5)$$

$$V_{L(off)} = V_{out} + V_{DS(q2)} \qquad (1.6)$$

With $D = T_{on} \cdot f_{sw}$, equations (1.4), (1.5) and (1.7) combine to

$$L = \frac{\left(V_{in} - V_{DS(q1)} - V_{out}\right) \times D}{\Delta i_L \times f_{sw}} \qquad (1.8)$$

and setting $\Delta i_{L(on)} = \Delta i_{L(off)}$ gives the duty cycle including switch drops:

$$D = \frac{V_{out} + V_{DS(q1)}}{V_{in} - V_{DS(q1)} + V_{DS(q2)}} \qquad (1.12)$$

Since the inductance is already fixed, rearrange for the one remaining unknown:

$$f_{sw} = \frac{\left(V_{in} - V_{DS(q1)} - V_{out}\right)\left(V_{out} + V_{DS(q1)}\right)}{L \times ICR \times I_{out} \times \left(V_{in} - V_{DS(q1)} + V_{DS(q2)}\right)} \qquad (1.14)$$

**Numerically**

$$V_{DS(q1,q2)} = 9\ \text{m}\Omega \times 3.5\ \text{A} = 0.0315\ \text{V}$$

$$D = \frac{24 + 0.0315}{48 - 0.0315 + 0.0315} = 0.50065 \quad (\geq 50\%)$$

$$\Delta i_L = 0.2 \times 3.5 = 0.7\ \text{A}$$

$$f_{sw} \geq \frac{(48 - 0.0315 - 24)(24 + 0.0315)}{(470\,\mu\text{H} \times 0.7)(48)} = 36{,}474\ \text{Hz}$$

**40 kHz** was chosen, comfortably above the floor and a convenient timer division.

### 2.2 Output capacitor

The output capacitor buffers the difference between inductor current and load current. When the inductor runs ahead of the load the surplus charges it; when it falls behind, the capacitor supplies the shortfall. Over many cycles these balance, average capacitor current is zero, and the DC output holds steady with only ripple superimposed.

**Transient sag.** On a load step the inductor cannot ramp instantly. The catch-up time is

$$t_{catch} = \frac{L \times \Delta I_{out}}{V_{in} \times D_{max} - V_{out}} \qquad (2.4)$$

During that interval the capacitor supplies, on average, half the step. The resulting droop is

$$V_{sag} = \frac{L \left(\Delta I_{out}\right)^2}{2\,C_{out}\left(V_{in} \times D_{max} - V_{out}\right)} \qquad (2.8)$$

and the mirror case when load current drops away:

$$V_{soar} = \frac{L\left(\Delta I_{out}\right)^2}{2\,V_{out}\,C_{out}} \qquad (2.9)$$

In practice $V_{in}D_{max} - V_{out} \to 0$, which would predict infinite sag, so the working form drops the $D_{max}$ term:

$$V_{sag} = \frac{L\left(\Delta I_{out}\right)^2}{2\,C_{out}\left(V_{in} - V_{out}\right)} \qquad (2.11)$$

For a 7S Li-ion pack at a 24 V CV setpoint with ±2 % allowed, V<sub>soar</sub> must stay under 24.48 V.

**Switching ripple.** Two components add: the ESR term and the charge/discharge term.

$$\Delta V_{out(esr)} = ESR \times \frac{\Delta i_L}{2}$$

$$\Delta V_{out} = \Delta V_{esr} + \Delta V_{C}$$

The capacitor current over a cycle is triangular, so the charge transferred is the area of that triangle. With $\Delta t = \tfrac{1}{2}t_{on} + \tfrac{1}{2}t_{off} = \tfrac{1}{2f_{sw}}$:

$$\Delta Q_C = \frac{1}{2} \times \frac{\Delta i_L}{2} \times \frac{1}{2 f_{sw}} = \frac{\Delta i_L}{8 f_{sw}}$$

$$\boxed{\;C_{min} \geq \frac{\Delta i_L}{8 \, f_{sw} \, \Delta V_C}\;}$$

Three 330 µF electrolytics in parallel give 23.3 mΩ equivalent ESR. The 1 µF ceramics alongside them are ignored here — their ESR is negligible by comparison.

### 2.3 Input capacitor

Input current in a buck converter is not smooth. The high-side MOSFET chops the bus at the switching frequency, and drawing those pulses straight from the source would ring against wiring inductance. The input capacitor provides a local low-impedance loop so the source only ever delivers the average DC current.

RMS ripple current the capacitor must tolerate:

$$I_{Cin(rms)} = \sqrt{D\left(I_{out}^2(1-D) + \frac{\Delta i_L^2}{12}\right)} \qquad (3.1)$$

Peak-to-peak input ripple, worst case at D = 50 % where the $D(1-D)$ product peaks:

$$\Delta V_{Cin(pp)} = \frac{I_{out} \times D(1-D)}{f_{sw} \times C_{in}} \;\;\longrightarrow\;\; \frac{I_{out}}{4 f_{sw} C_{in}} \qquad (3.3)$$

**Numerically**, limiting input ripple to 1 % of 48 V = 480 mV:

$$C_{in} \geq \frac{3.5}{4 \times 40{,}000 \times 0.48} = 45.6\ \mu\text{F}$$

The board already carries 990 µF, giving

$$\Delta V_{in} = \frac{3.5}{4 \times 40{,}000 \times 990\,\mu} = 22\ \text{mV}$$

— roughly twenty times inside the budget.

### 2.4 Losses

Conduction, gate-drive and capacitor losses are omitted to keep the estimate tractable; switching and inductor losses dominate at this operating point.

**Switching loss**

$$P_{sw(q1)} = \tfrac{1}{2} V_{in} \, i_L \, (t_r + t_f) \, f_{sw} = 0.5 \times 48 \times 3.5 \times 95\,\text{ns} \times 40\,\text{kHz} = 319.2\ \text{mW}$$

**Inductor loss**

$$P_L = DCR \times \sqrt{i_L^2 + \frac{\Delta i_L^2}{12}} = \sqrt{3.5^2 + \frac{0.7^2}{12}} \times 11.5\ \text{m}\Omega = 142\ \text{mW}$$

**Efficiency**

$$\eta = \frac{84}{84 + 0.142 + 0.319} = 99.5\%$$

This is an optimistic figure by construction — it counts only two loss mechanisms. Treat it as an upper bound on the power stage, not a system efficiency.

### 2.5 Duty-to-output transfer function

Averaging over a switching period and applying inductor volt-second balance:

$$L\frac{di}{dt} = d\left(V_{in} - V_{out}\right) + (1-d)\left(-V_{out}\right) = dV_{in} - V_{out}$$

and capacitor charge balance:

$$C\frac{dv}{dt} = i_L - \frac{v}{R}$$

The measured output includes the ESR drop:

$$V_{out} = V_C + r_c\left(i_L - \frac{V_{out}}{R}\right)$$

Perturbing around the operating point and solving for $\hat{v}_{out}/\hat{d}$:

$$\boxed{\;G_{vd}(s) = \frac{V_{in}\left(1 + s\,r_c C\right)}{LCs^2 + s\left(\frac{L}{R} + r_c C\right) + 1}\;}$$

Ignoring ESR:

$$G_{vd}(s) = \frac{V_{in}}{LCs^2 + \frac{L}{R}s + 1}$$

Control-to-inductor-current:

$$G_{id}(s) = \frac{V_{in} C s}{LCs^2 + \frac{L}{R}s + 1}$$

**Resonance**

$$f_0 = \frac{1}{2\pi\sqrt{LC}} = \frac{1}{2\pi\sqrt{470\,\mu \times 992\,\mu}} = 233\ \text{Hz}$$

The LC pole pair sits at 233 Hz, two decades below the 40 kHz switching frequency — ample separation for a current loop crossing well above the resonance and well below f<sub>sw</sub>/5.

---

## 3. Boost mode (battery discharging)

### 3.1 Inductor

With `S1` off and `S2` on: $V_{L(on)} = V_{in} - V_{DS(q2)}$

With `S1` on and `S2` off: $V_{L(off)} = V_{in} - \left(V_{out} + V_{DS(q2)}\right)$

Volt-second balance over a cycle:

$$D\left(V_{in} - V_{DS(q1)}\right) + (1-D)\left(V_{in} - V_{out} - V_{DS(q2)}\right) = 0$$

$$V_{out} = \frac{V_{in}}{1-D} - V_{DS(q1)} - \frac{D}{1-D}V_{DS(q2)}$$

$$\boxed{\;D = \frac{V_{out} + V_{DS(q1)} - V_{in}}{V_{out} + V_{DS(q1)} - V_{DS(q2)}}\;}$$

Average inductor current is higher than output current by the conversion ratio:

$$i_{L(avg)} = \frac{I_{out}}{1-D}$$

### 3.2 Capacitors

While `S2` conducts, the output capacitor alone supplies the load:

$$\Delta V_{C(out)} = \frac{I_{out} \times D}{C_{out} \times f_{sw}} + \Delta i_L \times ESR$$

$$\Delta V_{C(in)} = \frac{\Delta i_L \times D}{8 \, C_{in} \, f_{sw}}$$

### 3.3 Numerical results

$$D = \frac{48 + 0.0315 - 24}{48 + 0.0315 - 0.0315} = 0.5 \quad (\leq 50\%)$$

$$i_{L(avg)} = \frac{3.5}{1 - 0.5} = 7\ \text{A}$$

At a 10 % ripple ratio, $\Delta i_L = 0.7$ A, giving

$$f_{sw} \geq \frac{(48 - 24 + 0.0315)(1 - 0.5)}{470\,\mu \times 0.7} = 36{,}426\ \text{Hz} \;\approx\; 40\ \text{kHz}$$

Note that both modes land on essentially the same frequency floor — one 40 kHz timer configuration serves the whole converter.

$$\Delta V_{C(out)} = \frac{3.5 \times 0.5}{990\mu \times 40\text{k}} + 0.7 \times 0.023 = 44\ \text{mV}$$

against a 2 % budget of 960 mV.

$$\Delta V_{C(in)} = \frac{0.7 \times 0.5}{8 \times 992\mu \times 40\text{k}} = 1.1\ \text{mV}$$

### 3.4 Losses

$$P_{sw(q2)} = \tfrac{1}{2} V_{out} \, i_L \, (t_r + t_f) \, f_{sw} = 0.5 \times 48 \times 7 \times 95\,\text{ns} \times 40\,\text{kHz} = 638.4\ \text{mW}$$

$$P_L = \sqrt{7^2 + \frac{0.7^2}{12}} \times 11.5\ \text{m}\Omega = 80.5\ \text{mW}$$

$$\eta = \frac{168}{168 + 0.638 + 0.081} = 99.5\%$$

### 3.5 Transfer function

$$\boxed{\;G_{vd}(s) = \frac{(1-D)V - I_L s}{LCs^2 + \frac{L}{R}s + (1-D)^2}\;}$$

The negative $s$ term in the numerator is the **right-half-plane zero** characteristic of boost operation. Physically, raising the duty cycle initially *drops* output voltage — more energy is being stored in the inductor before any of it reaches the output. It adds phase lag without attenuation, so no amount of loop gain can design around it. The crossover frequency must sit below the RHP zero, and below roughly f<sub>sw</sub>/5.

---

## 4. Summary

| Quantity | Buck (charge) | Boost (discharge) |
|---|---|---|
| Duty cycle | 0.5007 | 0.500 |
| Minimum f<sub>sw</sub> | 36.5 kHz | 36.4 kHz |
| Chosen f<sub>sw</sub> | 40 kHz | 40 kHz |
| Δi<sub>L</sub> | 0.7 A (20 %) | 0.7 A (10 %) |
| i<sub>L(avg)</sub> | 3.5 A | 7 A |
| Output ripple | 22 mV | 44 mV |
| Switching loss | 319 mW | 638 mW |
| Inductor loss | 142 mW | 81 mW |
| Estimated efficiency | ~99.5 % | ~99.5 % |
| LC resonance | 233 Hz | 233 Hz |
