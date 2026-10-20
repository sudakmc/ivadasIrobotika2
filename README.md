# Širdies ritmo ekranas (Heartbeat Display)

Arduino Uno projektas iš **matavimo ir atvaizdavimo** srities. Vartotojas mygtuku „pataksi“ ritmą, Arduino išmatuoja laiką tarp paspaudimų ir apskaičiuoja **BPM**, o 8×8 LED matricoje plakanti širdis prisitaiko prie to greičio. Antru mygtuku keičiama animacija ir šviesumas, LCD ekrane rodoma būsena, o nustatymai išsaugomi **EEPROM** ir išlieka išjungus maitinimą.

Projektas sukurtas kaip robotikos kurso 2-asis namų darbas (teminė užduotis) ir veikia su tikra aparatūra.

![Sujungimų schema](docs/schema.png)

## Pristatymas

Projektas pademonstruotas dėstytojui paskaitos metu: paleidimas, ritmo matavimas, INT0 pranešimai Serial Monitor lange, animacijų ir šviesumo keitimas, EEPROM išlikimas po maitinimo išjungimo.

Trumpa ataskaita (dizainas, laiko biudžetas, testai, žinomos problemos): [`ATASKAITA.md`](ATASKAITA.md)

---

## Turinys

- [Kaip veikia](#kaip-veikia)
- [Reikalavimų atitikimas](#reikalavimų-atitikimas)
- [Komponentai](#komponentai)
- [Schema ir sujungimai](#schema-ir-sujungimai)
- [Paleidimas](#paleidimas)
- [Programos architektūra](#programos-architektūra)
- [Laikmačio konfigūracija](#laikmačio-konfigūracija)
- [Pertraukimų (ISR) vaidmenys](#pertraukimų-isr-vaidmenys)
- [EEPROM išdėstymas](#eeprom-išdėstymas)
- [Keičiami parametrai](#keičiami-parametrai)
- [Kaip pridėti animaciją](#kaip-pridėti-animaciją)
- [Repozitorijos struktūra](#repozitorijos-struktūra)

---

## Kaip veikia

| Veiksmas | Rezultatas |
|----------|-----------|
| Įjungimas | Pyptelėjimas, matrica užsipildo eilutė po eilutės, LCD: `Sirdies ekranas / Kraunama...` |
| **TAP** mygtukas (D2) spaudžiamas ritmu | LCD: `Matuoja 90 (4)`, širdis plaka išmatuotu greičiu |
| 2 s be paspaudimų | Matavimas baigiamas, po 1,5 s nustatymai įrašomi į EEPROM, LCD: `Issaugota EEPROM` |
| **ANIM** mygtukas (D3), trumpas paspaudimas | Kita animacija: Širdis → Šypsena → Pac-Man |
| **ANIM** mygtukas, ilgas paspaudimas (≥ 0,6 s) | Kitas šviesumo lygis (1–5) |
| Maitinimo išjungimas ir įjungimas | Grįžta ta pati animacija, BPM ir šviesumas |

**Matavimas:** `BPM = 60000 / (4 paskutinių intervalų tarp paspaudimų vidurkis, ms)`. Intervalai už 250–2000 ms ribų (240–30 BPM) atmetami. Širdies animacija turi 4 kadrus, o kadro trukmė parenkama taip, kad visa animacija truktų lygiai vieną širdies dūžį.

LCD pavyzdys veikimo metu:

```
1/3 Sirdis
BPM:72 Sviesa:2
```

## Reikalavimų atitikimas

| Reikalavimas | Kaip įgyvendinta | Kur kode |
|--------------|------------------|----------|
| Asinchroninis, įvykiais grįstas dizainas | Nėra `delay()`. Pertraukimai kelia vėliavas, `loop()` jas apdoroja, būsenų mašina valdo eigą | `// BENDRI SU ISR`, `updateState()`, `loop()` |
| ≥ 1 išorinis pertraukimas | INT0 (D2) fiksuoja paspaudimo laiką `micros()` tikslumu | `onTapEdge()` |
| ≥ 1 laikmačio pertraukimas | Timer1 Compare Match A kas 1 ms | `ISR(TIMER1_COMPA_vect)` |
| Aparatinis laikmatis | Timer1, CTC, 1 kHz: kadrų, mygtuko ir LCD tvarkaraštis | `setupTimer1()` |
| EEPROM ≥ 2 reikšmės | 3 reikšmės: animacija, BPM, šviesumas | `// EEPROM` |
| Validacija (magic/versija) | Magic baitas, versija, kontrolinė suma, reikšmių ribos | `loadSettings()` |
| Write-on-change | Rašomi tik pasikeitę baitai, įrašymas atidedamas 1,5 s | `writeIfChanged()`, `markDirty()` |
| Grįžtamasis ryšys | Matrica, LCD, garsiakalbis, Serial žurnalas | `drawFrame()`, `renderLcd()` |
| Saugumas ir surinkimas | Visi kontaktai vienoje vietoje, ribos konstantose, vidiniai pull-up, 220 Ω LCD apšvietimui | `// KONTAKTAI`, `// NUSTATYMAI` |

## Komponentai

| Kiekis | Komponentas | Paskirtis |
|--------|-------------|-----------|
| 1 | Arduino Uno R3 | Valdiklis |
| 1 | 8×8 LED matrica su MAX7219 (modulis) | Animacijos |
| 1 | LCD 1602A (16×2, be I2C) | Tekstinė informacija |
| 1 | Potenciometras 10 kΩ | LCD kontrastas |
| 1 | Rezistorius 220 Ω | LCD apšvietimo srovės ribojimas |
| 2 | Mygtukas | TAP (matavimas) ir ANIM (valdymas) |
| 1 | Pasyvus garsiakalbis | Garsinis grįžtamasis ryšys |
| 1 | Maketavimo plokštė + laidai | Sujungimai |

## Schema ir sujungimai

Schema aukščiau rodo visus laidus ant maketavimo plokštės. Žymos viršuje – Arduino kontaktai, į kuriuos eina laidai.

| Arduino | Prijungta prie | Paskirtis kode |
|---------|----------------|----------------|
| D2 | TAP mygtukas (kita koja į GND) | Išorinis pertraukimas INT0 |
| D3 | ANIM mygtukas (kita koja į GND) | Tikrinamas kas 10 ms |
| D4 | LCD 14 (D7) | LiquidCrystal duomenys |
| D5 | LCD 13 (D6) | LiquidCrystal duomenys |
| D6 | LCD 12 (D5) | LiquidCrystal duomenys |
| D7 | LCD 11 (D4) | LiquidCrystal duomenys |
| D8 | LCD 6 (E) | LiquidCrystal enable |
| D9 | LCD 4 (RS) | LiquidCrystal register select |
| D10 | Matricos CS | LedControl |
| D11 | Matricos DIN | LedControl |
| D12 | Matricos CLK | LedControl |
| A0 | Garsiakalbio + | `tone()` |
| 5V | + linija | Maitinimas |
| GND | − linija | Masė |

LCD papildomai:

| LCD kontaktas | Prijungta prie |
|---------------|----------------|
| 1 VSS, 5 RW, 16 K | GND |
| 2 VDD | 5V |
| 3 V0 | Potenciometro vidurinis kontaktas (kraštiniai į 5V ir GND) |
| 15 A | 5V per 220 Ω rezistorių |
| 7–10 (D0–D3) | Neprijungti (4 bitų režimas) |

### Kodėl būtent šie kontaktai

- **D2 ir D3** yra vieninteliai Arduino Uno kontaktai su išoriniais pertraukimais (INT0, INT1), todėl TAP mygtukas yra D2.
- **D0 ir D1** palikti laisvi, nes juos naudoja USB/Serial ryšys.
- **Timer1** naudojamas sistemos laikrodžiui, todėl `analogWrite()` D9 ir D10 kontaktuose neveiktų. D9 ir D10 čia naudojami tik kaip skaitmeniniai išėjimai, todėl konflikto nėra.
- **Timer0** paliktas `millis()` / `micros()`, **Timer2** – `tone()` funkcijai.

### Saugumas

- Mygtukai jungiami tarp kontakto ir GND, naudojami vidiniai pull-up rezistoriai (`INPUT_PULLUP`). Kontaktas niekada nėra jungiamas tiesiai prie 5V.
- LCD apšvietimas ribojamas 220 Ω rezistoriumi.
- Matricos šviesumas ribojamas lygiais `{1, 4, 8, 12, 15}` iš 15, numatytasis – 4. Taip srovė iš USB lieka gerokai mažesnė nei 500 mA.

## Paleidimas

1. Įdiekite **Arduino IDE 2.x**. Ubuntu sistemoje leiskite naudoti serijinį prievadą:
   ```bash
   sudo usermod -aG dialout $USER   # tada atsijunkite ir vėl prisijunkite
   ```
2. **Library Manager** lange įdiekite **LedControl** (Eberhard Fahle). `LiquidCrystal` ir `EEPROM` jau yra IDE.
3. Sujunkite grandinę pagal lentelę ir įkelkite [`tools/HardwareTest`](tools/HardwareTest/HardwareTest.ino) – jis patikrina LCD, matricą, mygtukus ir garsiakalbį.
4. Atidarykite [`src/HeartbeatDisplay/HeartbeatDisplay.ino`](src/HeartbeatDisplay/HeartbeatDisplay.ino), pasirinkite **Arduino Uno** ir teisingą prievadą, spauskite **Upload**.
5. Atidarykite **Serial Monitor** (115200 baud), kad matytumėte įvykių žurnalą.

Su `arduino-cli`:
```bash
arduino-cli lib install LedControl
arduino-cli compile --fqbn arduino:avr:uno src/HeartbeatDisplay
arduino-cli upload  --fqbn arduino:avr:uno -p /dev/ttyUSB0 src/HeartbeatDisplay
```

Serial žurnalo pavyzdys:
```
== Širdies ritmo ekranas v1 ==
[EEPROM] nuskaityta: animacija=0 bpm=72 šviesumas=2
[0 ms] -> PALEIDIMAS
[1500 ms] -> VEIKIA
[5210 ms] -> MATUOJA
  INT0 paspaudimas, intervalas 668 ms -> BPM 89
  INT0 paspaudimas, intervalas 655 ms -> BPM 90
[9120 ms] -> VEIKIA
[EEPROM] įrašyta baitų: 2
[10620 ms] -> ISSAUGOTA
```

## Programos architektūra

```
 INT0 ISR (D2)      ──► gTapUs = micros(), fTap = true ───────────────┐
 Timer1 ISR (1 kHz) ──► gMs++, fFrame / fInput (10 ms) / fLcd (100 ms) ┤
                                                                       ▼
 loop():  pollAnimButton()  ─►  handleTap()  ─►  updateState()  ─►  nextFrame()  ─►  renderLcd()
```

Programoje nėra nė vieno `delay()`. Kiekvienas veiksmas vykdomas tik tada, kai pakelta jo vėliava, todėl animacija, mygtukai, matavimas ir LCD veikia lygiagrečiai.

### Būsenų mašina

```
PALEIDIMAS ──1,5 s──► VEIKIA ◄──2 s be paspaudimo── MATUOJA
                        │  ▲                           ▲
                        │  └──1,2 s── ISSAUGOTA        │
                        └──TAP paspaudimas─────────────┘
```

| Būsena | Ką daro | Kada baigiasi |
|--------|---------|---------------|
| PALEIDIMAS | Matrica pildoma eilutėmis, mygtukai ignoruojami | Po 1,5 s |
| VEIKIA | Rodoma animacija, laukiama įvykių | TAP paspaudimas arba įrašymas į EEPROM |
| MATUOJA | Skaičiuojami intervalai ir BPM | 2 s be paspaudimo |
| ISSAUGOTA | LCD rodo `Issaugota EEPROM` | Po 1,2 s |

## Laikmačio konfigūracija

| Laikmatis | Naudoja | Nustatymai |
|-----------|---------|------------|
| Timer0 | Arduino `millis()` / `micros()` | Nekeičiamas. Duoda 4 µs paspaudimo laiko tikslumą |
| **Timer1** | **Sistemos laikrodis** | CTC režimas (`WGM12`), dalytuvas 64 (`CS11 \| CS10`), `OCR1A = 249` → 16 MHz / 64 / 250 = **1000 Hz** |
| Timer2 | `tone()` garsiakalbiui | Arduino branduolys |

Timer1 pertraukimas skaičiuoja tris skaitiklius ir kelia vėliavas:

| Vėliava | Periodas | Kam |
|---------|----------|-----|
| `fFrame` | `gFrameMs` (pagal animaciją arba BPM) | Kitas animacijos kadras |
| `fInput` | 10 ms | ANIM mygtuko tikrinimas |
| `fLcd` | 100 ms | LCD atnaujinimas |

## Pertraukimų (ISR) vaidmenys

| ISR | Iššaukia | Ką daro |
|-----|----------|---------|
| `TIMER1_COMPA_vect` | Kas 1 ms | Didina `gMs`, kelia `fFrame`, `fInput`, `fLcd`. Jokio lėto darbo |
| `onTapEdge` (INT0) | Kiekvienas D2 lygio pasikeitimas (`CHANGE`) | Nuskaito kontaktą per `PIND`, jei tai paspaudimas po ≥ 20 ms ramybės – įrašo `micros()` į `gTapUs` ir kelia `fTap` |

**Debounce pertraukime.** Paspaudimas priimamas tik tada, kai linija prieš tai buvo rami bent 20 ms. Taip pirmasis tikro paspaudimo kraštas priimamas iš karto (be vėlinimo), o kontaktų virpėjimas po jo ir atleidžiant ignoruojamas.

**Bendri kintamieji** pažymėti `volatile`, o `loop()` juos skaito `ATOMIC_BLOCK` viduje (`nowMs()`, `takeFlag()`), nes 8 bitų AVR 32 bitų skaičių nuskaito ne vienu veiksmu.

## EEPROM išdėstymas

| Adresas | Dydis | Laukas | Reikšmė |
|---------|-------|--------|---------|
| 0 | 1 B | magic | `0xB7` – rodo, kad duomenys priklauso šiai programai |
| 1 | 1 B | versija | `1` – išdėstymo versija |
| 2 | 1 B | animacija | 0 … animacijų skaičius − 1 |
| 3 | 1 B | BPM | 30 … 240 |
| 4 | 1 B | šviesumas | lygio indeksas 0 … 4 |
| 5 | 1 B | kontrolinė suma | `magic ^ versija ^ anim ^ bpm ^ šviesumas ^ 0x5A` |

- **Validacija:** paleidžiant tikrinamas magic, versija, kontrolinė suma ir kiekvienos reikšmės ribos. Bent vienam neatitikus, įkeliamos numatytosios reikšmės ir jos įrašomos.
- **Write-on-change:** `writeIfChanged()` lygina kiekvieną baitą su išsaugotu ir rašo tik skirtingus. Serial žurnale matyti, kiek baitų įrašyta.
- **Atidėtas įrašymas:** po pakeitimo laukiama 1,5 s, o matavimo metu nerašoma visai. Dešimt greitų paspaudimų = vienas įrašymas.

## Keičiami parametrai

Visi nustatymai sudėti failo viršuje:

| Parametras | Reikšmė | Ką keičia |
|------------|---------|-----------|
| `BPM_MIN`, `BPM_MAX` | 30 / 240 | Leistinas BPM intervalas (≤ 255) |
| `BPM_DEFAULT` | 72 | BPM pirmą kartą paleidus |
| `BRIGHT_LEVELS[]` | `{1, 4, 8, 12, 15}` | Šviesumo lygiai (0–15) |
| `BRIGHT_DEFAULT` | 1 | Pradinio lygio indeksas |
| `BEAT_SOUND` | `true` | `false` išjungia pyptelėjimą ant kiekvieno dūžio |
| `TAP_DEBOUNCE_US` | 20000 µs | Virpėjimo filtras TAP mygtukui |
| `TAP_TIMEOUT_MS` | 2000 ms | Kada baigiamas matavimas |
| `LONG_PRESS_MS` | 600 ms | Ilgo paspaudimo riba |
| `SAVE_DELAY_MS` | 1500 ms | EEPROM įrašymo delsa |

Pakeitus EEPROM išdėstymą, būtina padidinti `EE_VERSION` – tada seni duomenys bus atmesti.

## Kaip pridėti animaciją

1. Aprašykite kadrų masyvą (kiekviena eilutė – vienas baitas, `1` = LED įjungtas):
   ```cpp
   const uint8_t ARROW_FRAMES[][8] PROGMEM = {
     { 0b00011000, 0b00111100, 0b01111110, 0b11111111,
       0b00011000, 0b00011000, 0b00011000, 0b00011000 },
     { 0b00000000, 0b00011000, 0b00111100, 0b01111110,
       0b11111111, 0b00011000, 0b00011000, 0b00011000 },
   };
   ```
2. Pridėkite eilutę į `ANIMS[]`:
   ```cpp
   ANIM("Rodykle", ARROW_FRAMES, 150),
   ```

Kadrų skaičius apskaičiuojamas automatiškai. Greitis `0` reiškia, kad animacija plaka pagal išmatuotą BPM.

## Repozitorijos struktūra

```
2ND/
├── src/
│   └── HeartbeatDisplay/
│       └── HeartbeatDisplay.ino   # Pagrindinė programa
├── tools/
│   └── HardwareTest/
│       └── HardwareTest.ino       # Sujungimų testas
├── docs/
│   └── schema.png                 # Sujungimų schema
├── ATASKAITA.md                   # Trumpa ataskaita (≤ 1 psl.)
└── README.md
```

## Licencija

MIT — žr. [LICENSE](../LICENSE).
