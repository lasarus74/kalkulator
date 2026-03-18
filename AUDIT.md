# Calculator Logic Audit

> Generated: 2026-03-18
> Scope: Full business logic and calculation flow analysis
> Source: `kalkulator.html` (lines 645–1051), `data/diets.json`, `data/eur.json`

---

## 1. All Input Fields

| # | ID | Type | Default | Label (PL) | Meaning |
|---|---|---|---|---|---|
| 1 | `stawkaBruttoEuro` | number (step 0.01) | 21 | Stawka NETTO (do wypłaty) w EURO za godzinę | Desired **net** hourly rate in EUR (misleading variable name — it's net, not brutto) |
| 2 | `kursEuro` | number (step 0.01) | 4.25 | Kurs EURO (PLN) | EUR → PLN exchange rate |
| 3 | `iloscGodzin` | number (step 1) | 176 | Ilość godzin w miesiącu | Working hours per month |
| 4 | `dniZaGranica` | number (step 1) | 20 | Ilość dni za granicą (w miesiącu) | Days spent abroad in the month |
| 5 | `pitNarastajaco` | number (step 0.01) | 0 | Podstawa PIT narastająco przed tym miesiącem (G, PLN) | Year-to-date PIT base before current month |
| 6 | `dietCountry` | select | — | (dropdown) | Country selector — populates diet value from JSON |
| 7 | `wysokoscDiety` | number (step 0.01) | 55 | Wysokość diety w EURO | Daily diet allowance in EUR |
| 8 | `srednia` | number (step 0.01) | 8673 | Średnia krajowa (PLN) | National average salary in PLN (ZUS floor) |
| 9 | `imie` | text | — | Imię | First name (display only, not used in calculations) |
| 10 | `nazwisko` | text | — | Nazwisko | Last name (display only, not used in calculations) |

**Note:** `imie` and `nazwisko` have zero impact on calculations — they are purely for labeling results.

---

## 2. Calculation Flow (Step-by-Step)

The calculator operates in **reverse mode**: user provides desired net hourly rate → system finds required gross salary.

### Entry point: `calculate()` (line 947)

```
1. VALIDATE
   └─ validateAllFields() → abort if any field invalid

2. PARSE INPUTS
   stawkaNettoEuro, kursEuro, iloscGodzin, dniZaGranica,
   wysokoscDiety, srednia, pitNarastajaco

3. COMPUTE TARGET NET MONTHLY (PLN)
   targetB_pln = stawkaNettoEuro × kursEuro × iloscGodzin

4. REVERSE-SOLVE FOR BRUTTO
   A_pln = findAfromB(targetB_pln, ...)
   └─ binary search calling calculateFromA() repeatedly

5. FORWARD CALCULATION (with found A_pln)
   calculateFromA(A_pln, ...) → full breakdown

6. CONVERT TO EUR & HOURLY
   PLN → EUR division, monthly → hourly division

7. DISPLAY RESULTS
   formatNumber() applied to every output value
```

### Inside `calculateFromA(A_pln, dniZaGranica, dietaEuro, kursEuro, srednia, pitNarastajaco)` (line 872):

```
Step 5.1  dietaZUS  = 1.00 × dietaEuro × kursEuro × dniZaGranica
Step 5.2  dietaPIT  = 0.30 × dietaEuro × kursEuro × dniZaGranica
Step 5.3  D         = max(A_pln − dietaZUS, srednia)          ← ZUS base
Step 5.4  E         = D × 0.1371                               ← employee ZUS
Step 5.5  F         = (D − E) × 0.09                           ← health insurance
Step 5.6  G         = (A_pln − dietaPIT) − E − 300             ← PIT base
Step 5.7  H         = calculatePIT(G, pitNarastajaco)           ← PIT tax
Step 5.8  B_pln     = A_pln − E − F − H                        ← net to employee
Step 5.9  zusPrac.  = D × 0.1952                               ← employer ZUS
Step 5.10 C_pln     = A_pln + zusPracodawcy                    ← total employer cost
```

---

## 3. Mathematical Formulas

### 3.1 Target net monthly
```
targetB_pln = stawkaNettoEuro × kursEuro × iloscGodzin
```

### 3.2 Diet deductions
```
dietaZUS = 1.00 × dietaEuro × kursEuro × dniZaGranica
dietaPIT = 0.30 × dietaEuro × kursEuro × dniZaGranica
```
- ZUS gets 100% diet deduction
- PIT gets only 30% diet deduction

### 3.3 ZUS base (with floor)
```
D = max(A_pln − dietaZUS, srednia)
```

### 3.4 Employee contributions
```
E = D × 0.1371          (employee ZUS — 13.71%)
F = (D − E) × 0.09      (health insurance — 9%)
```

### 3.5 PIT base
```
kosztyPracownicze = 300
G = (A_pln − dietaPIT) − E − kosztyPracownicze
```

### 3.6 PIT tax (`calculatePIT`, line 842)
```
prog = 120000 (annual threshold)
ytd  = pitNarastajaco

if ytd >= prog:
    podatek = G × 0.32
elif ytd + G <= prog:
    podatek = G × 0.12
else:
    czesc12 = prog − ytd
    czesc32 = G − czesc12
    podatek = czesc12 × 0.12 + czesc32 × 0.32

H = max(podatek − 300, 0)
```

### 3.7 Net to employee
```
B_pln = A_pln − E − F − H
```

### 3.8 Employer cost
```
zusPracodawcy = D × 0.1952     (employer ZUS — 19.52%)
C_pln = A_pln + zusPracodawcy
```

### 3.9 EUR/hourly conversions (display)
```
pracownikWyplataEURO = B_pln / kursEuro
pracownikWyplata1h   = pracownikWyplataEURO / iloscGodzin  (if iloscGodzin > 0)

kosztPracodawcyEURO  = C_pln / kursEuro
kosztPracodawcy1h    = kosztPracodawcyEURO / iloscGodzin    (if iloscGodzin > 0)
```

### 3.10 Reverse solver (`findAfromB`, line 911)
```
Algorithm: binary search (bisection)
Initial range: low = 0, high = max(targetB × 1.6, srednia)
Expansion: high × 1.6 up to 60 times if needed
Iterations: up to 80
Tolerance: |B_mid − targetB| < 0.01 PLN
```

---

## 4. Rounding

| Where | How | Precision |
|---|---|---|
| `formatNumber()` (all displayed results) | `num.toFixed(decimals)` | 2 decimal places (default) |
| EUR rate from NBP | `Number(mid).toFixed(4)` | 4 decimal places |
| Diet value from JSON | `Number(item.diet).toFixed(2)` | 2 decimal places |
| Binary search tolerance | `Math.abs(bMid - targetB_pln) < 0.01` | 0.01 PLN (1 grosz) |

**Key observations:**
- **No intermediate rounding** — all internal calculations use full floating-point precision
- Rounding is applied **only at display time** via `formatNumber()`
- The binary search stops at 0.01 PLN accuracy, which is the smallest meaningful unit
- Input sanitization allows decimal input but does not round inputs

---

## 5. Diet Calculation

### 5.1 Data source
- **File:** `data/diets.json`
- **Format:**
  ```json
  { "DE": { "name": "Germany", "currency": "EUR", "diet": 49 } }
  ```
- Currently only 1 country (Germany, 49 EUR/day)
- Loaded via `loadDiets()` → stored in `window.__diets`

### 5.2 Diet selection flow
1. User selects country from `dietCountry` dropdown
2. `applyDietFromCountry()` reads `window.__diets[code].diet`
3. Sets `wysokoscDiety` input value
4. Triggers `calculate()`

### 5.3 Diet in calculation
```
dietaZUS = 1.00 × wysokoscDiety × kursEuro × dniZaGranica
dietaPIT = 0.30 × wysokoscDiety × kursEuro × dniZaGranica
```

### 5.4 Inputs affecting diet
- `wysokoscDiety` — daily rate in EUR
- `kursEuro` — exchange rate
- `dniZaGranica` — number of days abroad

### 5.5 Diet impact
- **dietaZUS** reduces ZUS base (but cannot push it below `srednia`)
- **dietaPIT** reduces PIT base (no floor)
- Asymmetry: 100% vs 30% creates tax-advantageous treatment for posted workers

---

## 6. Tax / Base Calculations

### 6.1 ZUS base
```
D = max(A_pln − dietaZUS, srednia)
```
- Floor at national average (`srednia` = 8673 PLN default)
- Prevents ZUS base from dropping below minimum regardless of diet deductions

### 6.2 Employee ZUS
```
E = D × 0.1371
```
- 13.71% of ZUS base
- This is the consolidated employee social insurance rate

### 6.3 Health insurance
```
F = (D − E) × 0.09
```
- 9% of (ZUS base minus employee ZUS)
- Calculated on reduced base

### 6.4 PIT base
```
G = (A_pln − dietaPIT) − E − 300
```
- Brutto minus 30% diet minus employee ZUS minus 300 PLN fixed costs
- **No minimum floor** (unlike ZUS base)

### 6.5 PIT tax
- Progressive: 12% up to 120,000 PLN/year, 32% above
- Tracks cumulative annual base via `pitNarastajaco`
- Three-way branch: fully below / fully above / straddles threshold
- Monthly reduction: −300 PLN
- Floor at 0 (tax cannot be negative)

### 6.6 Employer ZUS
```
zusPracodawcy = D × 0.1952
```
- 19.52% of same ZUS base
- Added on top of brutto for total employer cost

---

## 7. Dependencies

### 7.1 Input → derived variable map

| Derived Variable | Depends On |
|---|---|
| `targetB_pln` | stawkaNettoEuro, kursEuro, iloscGodzin |
| `dietaZUS` | wysokoscDiety, kursEuro, dniZaGranica |
| `dietaPIT` | wysokoscDiety, kursEuro, dniZaGranica |
| `A_pln` | targetB_pln, dniZaGranica, wysokoscDiety, kursEuro, srednia, pitNarastajaco |
| `D` (ZUS base) | A_pln, dietaZUS, srednia |
| `E` (employee ZUS) | D |
| `F` (health) | D, E |
| `G` (PIT base) | A_pln, dietaPIT, E |
| `H` (PIT tax) | G, pitNarastajaco |
| `B_pln` (net) | A_pln, E, F, H |
| `C_pln` (employer cost) | A_pln, zusPracodawcy |
| `zusPracodawcy` | D |

### 7.2 Pure inputs (not derived)
- stawkaNettoEuro
- kursEuro
- iloscGodzin
- dniZaGranica
- wysokoscDiety
- srednia
- pitNarastajaco

### 7.3 Constants (hardcoded)
- Employee ZUS rate: 13.71% (`0.1371`)
- Employer ZUS rate: 19.52% (`0.1952`)
- Health insurance rate: 9% (`0.09`)
- PIT rate low: 12% (`0.12`)
- PIT rate high: 32% (`0.32`)
- PIT annual threshold: 120,000 PLN (`prog`)
- Employee costs: 300 PLN (`kosztyPracownicze`)
- PIT monthly reduction: 300 PLN
- ZUS diet factor: 100% (`1.00`)
- PIT diet factor: 30% (`0.30`)

---

## 8. Logic Complexity

### 8.1 Linear (additive/multiplicative) parts
- Diet calculations: purely multiplicative
- ZUS employee/employer: single multiplication
- Health insurance: subtraction then multiplication
- Net calculation: subtractive chain (A − E − F − H)
- Employer cost: additive (A + zusPracodawcy)
- Currency conversions: division by exchange rate
- Hourly rates: division by hours

### 8.2 Conditional logic

| Location | Condition | Effect |
|---|---|---|
| ZUS base (line 880) | `max(A − dietaZUS, srednia)` | Floor at national average |
| PIT brackets (lines 851–860) | 3-way branch on `pitNarastajaco` vs 120,000 | Changes tax rate from 12% to 32% |
| PIT floor (line 866) | `max(podatek, 0)` | Tax cannot be negative |
| PIT guard (line 845) | `G ≤ 0` | Returns 0 tax if base is non-positive |
| Division guard (lines 979, 983) | `iloscGodzin > 0` | Prevents division by zero |
| Net guard (line 912) | `targetB ≤ 0 or !isFinite` | Returns 0 if invalid target |
| Bisection expansion (line 925) | `safety < 60` | Caps search range expansion |
| Bisection iterations (line 935) | `i < 80` | Caps convergence iterations |
| Bisection tolerance (line 939) | `|error| < 0.01` | Early exit when close enough |
| EUR rate age (line 705) | `age > 10 days` | Rejects stale local rate |

### 8.3 Summary
- Core calculation is **mostly linear** (chain of multiplications and subtractions)
- **Two non-linear elements:**
  1. ZUS base floor (`Math.max`) — creates a step discontinuity
  2. Progressive PIT brackets — piecewise linear with breakpoint at 120k PLN
- **One iterative element:** binary search for reverse brutto calculation
- No loops, recursion, or complex branching beyond PIT brackets
