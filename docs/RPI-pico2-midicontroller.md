# goal

low-cost, easy-to-build, **universal** fatar→midi controller. supports 2-switch-per-key velocity (br/mk), scans common fatar matrices (tp/9, tp/8, etc), adds 9+ analog inputs via 4051 mux(es). usb‑midi class device on pico 2; optional din midi out/in.

---

# big picture architecture

* **mcu**: raspberry pi pico 2 (rp2350, 3.3 v i/o, usb device). why pico 2?

  * cheap, widely available, long-term availability.
  * usb‑midi via tinyusb; plenty of cpu for tight scan loops.
  * **pio** blocks and dma for fast, low‑jitter scanning/shift‑io.
* **keyboard front-end** (universal, 5 v domain):

  * **drivers (to rows/BR/MK lines)**: 74**AHCT**244/541 at 5 v → accepts 3.3 v as logic‑high, outputs clean 5 v for long harnesses.
  * **receivers (from sense/T lines)**: 74**LVC**245/244 at 3.3 v → **5 v‑tolerant inputs**, level‑shift down to mcu.
  * **series resistors** (≈100 Ω) on all long lines; optional **schmitt** inputs (74HC14 on sense side) if needed.
  * **tvs/esd** arrays at the board connector.
* **i/o compression** for large matrices:

  * **74hc595** chain for driver lines (shift‑out; latched).
  * **74hc165** chain for sense lines (shift‑in; parallel‑load + clock).
  * clocks/latch via **spi or pio** → only 4–5 mcu pins total.
* **analogue inputs**: 1–2× **74hc4051** to feed 1 adc pin via **op‑amp buffer** (rail‑to‑rail, e.g. mcp6002) + rc sample/hold.
* **midi i/o**:

  * **usb‑midi** (class compliant) over pico 2.
  * **din midi out** (uart @ 31 250 bps) + **din midi in** (6n138 optocoupler) optional.

---

# why 5 v for the keybed domain?

* veel fatar‑keybeds (en andere) verwachten 5 v matrixtiming en hebben in de praktijk langere kabels/hogere lekstromen. 5 v geeft marge, lagere foutkans, en compatibiliteit met “diode‑richting varianten”.
* pico 2 blijft veilig op 3.3 v; alle verbindingen gaan via **hct‑drivers (upshift)** en **lvc‑buffers (downshift)**.

---

# matrix & velocity (br/mk)

* 2 contacten per toets: **br** (eerste) en **mk** (tweede). velocity = Δt tussen br‑sluiten en mk‑sluiten. bij loslaten opent mk eerst, dan br.
* **varianten** in het veld: diodepolarity soms omgekeerd; soms br/mk gewisseld. doel: **auto‑detect** en **profielen**.

## auto‑detect bij boot

1. zet front‑end in **profiel A**: “active‑low drive” (stuur één br/mk‑lijn **laag**; sense‑lijnen met pull‑ups). scan 10–20 ms terwijl toetsen worden bewogen.
2. geen plausibele events? wissel naar **profiel B**: “active‑high drive”.
3. binnen profiel test **br↔mk swap** heuristiek: bij indrukken moet br→mk volgorde (Δt 0.3–20 ms) vaak voorkomen, bij loslaten mk→br.
4. kies het profiel met consistente timingstatistiek; **persist** in flash; fallback naar manual override via sysex/config.

---

# scanstrategie (met shiftregisters)

* **drivers**: n×74hc595 → bv. 16 stuur‑lijnen (br+mk banken) voor 61/76/88 toetsen.
* **sensors**: m×74hc165 → bv. 12 sense‑lijnen (t0..t11) typisch.
* **loop** (pio/spi dma, vaste cadence 1–2 khz):

  1. shift driver‑bitmask (exact 1 “actieve” lijn) → latch.
  2. korte settle (0.5–2 µs) → parallel‑load 165, dan clock bits in.
  3. edge‑detect per contact; update **per‑key state‑machine**.
  4. velocity: bij br‑edge → `t0[key]=micros()`, bij mk‑edge → `dt=micros()-t0` → curve‑map 1..127.
  5. queue midi events.
* **debounce**: puur digitaal (2–4 gelijke samples op een contact) + contact‑timeout (max Δt 25–30 ms).
* **latency target**: volledige matrixscan < 1 ms → midi event binnen \~1–2 ms.

---

# adc + 4051 analoge ingangen

* **kanalen**: 8 (1×4051) of 16 (2×4051). voor 9 potmeters: 2×4051 is “future‑proof”.
* **select**: deel s0..s2 (3 lijnen) voor beide 4051’s; per 4051 een **enable** (2 lijnen) of gebruik één 4051 + 1 rechtstreekse adc voor pot #9.
* **buffering**: 4051‑uit → **op‑amp follower** → adc; daarnaast **100 Ω** serie + **100 nF** bij adc voor sample/hold.
* **ratiometrisch**: potmeters tussen 3.3 v en gnd; adc\_ref = 3.3 v.
* **filtering**: ema (α≈0.1–0.3) + **deadband** ±1–2 lsb; midi cc alleen bij verandering > threshold of elke 10–20 ms.

---

# pin‑budget (pico 2)

* **shift/spi**: sck, mosi, miso, latch (drivers), pl (sensors) → **5 pins**.
* **4051 selects**: s0..s2 → **3 pins**.
* **4051 enables**: **1–2 pins**.
* **adc input** (na buffer): **1 pin**.
* **uart** voor din‑midi out/in: **1–2 pins** (usb‑midi gebruikt usb, geen extra pins).
* **opties**: sustain/footswitch (1–2 pins).

> totaal typisch **12–14 gpio** → ruim binnen de **26** beschikbare headers op pico 2.

---

# firmware ontwerp

* **usb‑midi**: tinyusb device stack, 1× midi in/out cable; rate limit status/cc.
* **event engine**: ringbuffer; isr/pio‑dma vult buffers; midi task verzendt.
* **velocity‑curves**: linear, log, exp, "piano", user table; sysex settable.
* **matrix profiler**: startup auto‑detect + manual override via sysex.
* **config store**: flash (wear‑leveling klein), versioned params.
* **diagnostics**: hid/serial console: matrix live‑view, Δt histogram, adc monitor.

---

# electrical details & checks

* **pull‑ups**:

  * sense‑lijnen: 10–22 k naar 5 v (of 3.3 v bij profiel‑variant); plaats dichtbij front‑end.
  * gebruik **hct drivers** (5 v) om één br/mk‑lijn per keer te forceren.
* **kabelgedrag**:

  * 100 Ω serie op elke driver/sense lijn; bij problemen: schmitt (hc14) vóór lvc245, of rc 47–100 pF naar gnd.
  * tvs op connectorbank; gnd‑guard sporen naast snelle lijnen.
* **over‑voltage bescherming**: never feed >3.6 v naar adc/gpio; lvc245 buft alle 5 v→3.3 v paden; adc krijgt uitsluitend 3.3 v‑potnet via buffer.
* **power**: 5 v voor keybed‑domain en midi; 3.3 v via pico 2 smps (vsys 1.8–5.5 v).

---

# bring‑up & validation plan

1. **bench‑harness** (één octaaf): maak kabel naar fatar‑connector; label alle lijnen; bevestig dioderichting met multimeter (diode test) *zonder* de carbonsporen te verhitten.
2. **profile A scan**: active‑low drive; meet met logic analyzer (sample ≥5 mhz). bevestig: één driver actief; sense‑bits veranderen bij toets; scanperiode <1 ms.
3. **profile B scan**: active‑high; vergelijk. kies profiel met geldige events.
4. **velocity calibratie**: log Δt voor zachte/medium/harde aanslag; fit naar 1..127; sla curve op.
5. **chording/ghosting**: toetscombinaties (3–4 tegelijk) → geen valse noten (diodes doen werk).
6. **adc sweep**: alle potkanalen; ruis/jitter < 1–2 lsb na filtering; geen crosstalk bij snel schakelen (zonodig verhoog settle‑tijd of buffer‑bandbreedte).
7. **midi soak**: glissando + pot‑sweeps 5 min → geen buffer overruns; usb‑midi stabiel; din out op 31 250 bps.

---

# work breakdown & timeline (indicatief)

**week 0–1 — specificatie & inkoop**

* bevestig doel‑keybed (61/76/88) en connector type; bestel pico 2, hct244/541, lvc245/244, hc595/165, hc4051, mcp6002, tvs, 6n138, weerstanden/headers; bestel kabels/adapters.
* risico: fatar levertijd onzeker → plan **alternatief**: gebruikte keybed (donor) of test‑rig voor één octaaf.

**week 2 — front‑end proto (1‑octaaf rig)**

* bouw klein pcb of perfboard met 1× hc595, 1× hc165, hct244, lvc245, connector naar rig.
* schrijf **firmware skeleton**: spi/pio shift i/o, scanloop, usb‑midi echo, adc via 4051.
* test profiel A/B + Δt logging; iteraties op pull‑ups/series R.

**week 3 — volledige matrix & 9 pots**

* schaal shiftketens (meerdere 595/165); integreer 2×4051 + op‑amp.
* implementeer velocity‑curves, debouncing, cc‑mapping, sysex config.
* usb‑midi + optionele din out/in.

**week 4 — validatie & behuizing**

* langdurige soak‑testen; edge‑cases (snelle repetities, legato, grote akkoorden).
* meet latency/jitter; stel scanfrequentie en timers af.
* documenteer pinout, kabelmapping, bouwinstructies; design een eenvoudige 2‑laag pcb.

---

# answers to key feasibility questions

* **genoeg io op pico 2?** ja **met shiftregisters**: \~12–14 gpio totaal. **zonder** i/o‑compressie red je het niet voor 61/76/88.
* **adc‑kanalen genoeg?** ja; 1 adc‑pin volstaat voor mux‑uit; we gebruiken 3 gpio voor s0..s2 (+1–2 enables).
* **3.3 v vs 5 v**: keybed op 5 v via hct/lvc level‑shift is veilig/universeel; pico blijft 3.3 v. (optioneel profiel voor 3.3 v als het specifieke keybed het toelaat.)
* **lange kabels**: drivers op 5 v (hct) + series R + schmitt‑ontvangst/afscherming; zo houd je randen schoon en storingsvrij.

---

# midi mapping (voorstel)

* kanaal 1; note on/off met gemapte velocity; note off bij mk→br release of na sustainlogica.
* 9 pots → cc: \[mod(1), expression(11), brightness(74), resonance(71), attack(73), pan(10), volume(7), reverb(91), chorus(93)]. aanpasbaar via sysex.
* startup: all notes off, reset controllers (cc121), omni off/poly on.

---

# bom (indicatief)

* pico 2; 74**hc595** × 2–4; 74**hc165** × 2–4; 74**ahct244/541** × 2–4; 74**lvc245/244** × 2–4; 74**hc4051** × 2; **mcp6002**; 6n138 (din‑in); transistor + 220 Ω weerstanden (din‑out); tvs arrays; r/c netwerk; headers/connectoren (micromatch/flatcable adapter);
* optional: oled (i2c) + encoder voor field‑config.

---

# risks & mitigations

* **fatar leveronzekerheid**: ontwerp **universele interface** + test‑rig met los octaaf; accepteer donor‑keybeds.
* **matrixvarianten**: auto‑detect + manuele overrides; modulaire kabeladapter.
* **ruis/jitter**: pio/dma voor constante scans; hardware‑buffers en rc; usb iso‑chronous niet nodig (midi is interrupt‑in); vermijd lange ground‑loops.

---

# next steps

1. bevestig aantal toetsen en connector type (foto’s/nummering).
2. kies 5 v‑front‑end onderdelen (hct/lvc varianten op voorraad) en bestel.
3. bouw 1‑octaaf rig + firmware skeleton; run auto‑detect en Δt histogram.
4. itereren tot scan<1 ms, stabiele velocity, geen ghosting; dan opschalen en pcb tekenen.
