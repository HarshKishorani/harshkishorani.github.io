---
title: "Explaination to Maxwell's Equation."
summary: "Coming Soon. Stay tuned"
math: true
---

### 1. **Gauss's Law for Electric Fields**  
_The electric flux through a closed surface is proportional to the total charge enclosed by that surface._

\[
\oint_{\partial V} \mathbf{E} \cdot d\mathbf{A} = \frac{Q_{\text{enc}}}{\varepsilon_0}
\]

---

### 2. **Gauss's Law for Magnetic Fields**  
_The magnetic flux through a closed surface is always zero, implying there are no magnetic monopoles._

\[
\oint_{\partial V} \mathbf{B} \cdot d\mathbf{A} = 0
\]

---

### 3. **Faraday's Law of Induction**  
_The electromotive force around a closed loop is equal to the negative rate of change of magnetic flux through the loop._

\[
\oint_{\partial S} \mathbf{E} \cdot d\mathbf{l} = -\frac{d}{dt} \int_S \mathbf{B} \cdot d\mathbf{A}
\]

---

### 4. **Ampère's Law with Maxwell's Correction**  
_The circulation of the magnetic field around a closed loop is equal to the sum of the conduction current and the displacement current through the loop._

\[
\oint_{\partial S} \mathbf{B} \cdot d\mathbf{l} = \mu_0 \int_S \mathbf{J} \cdot d\mathbf{A} + \mu_0 \varepsilon_0 \frac{d}{dt} \int_S \mathbf{E} \cdot d\mathbf{A}
\]