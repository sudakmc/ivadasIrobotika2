# HW2 ataskaita: Širdies ritmo ekranas

**Autorius:** Danielius Starkutis · **Plokštė:** Arduino Uno (ATmega328P, 16 MHz)

## Dizainas

Užduoties sritis – **matavimas ir atvaizdavimas**. TAP mygtuku vartotojas „pataksi“ ritmą, programa išmatuoja intervalus tarp paspaudimų ir apskaičiuoja BPM (`60000 / 4 paskutinių intervalų vidurkis`). Širdies animacija 8×8 matricoje plaka tuo greičiu: 4 kadrų trukmė parenkama taip, kad visa animacija būtų vienas dūžis. ANIM mygtukas keičia animaciją (trumpas paspaudimas) ir šviesumą (ilgas). LCD rodo animaciją, BPM, šviesumą ir įvykius.

Programa grįsta įvykiais: **Timer1** (CTC, 1 kHz) ir **INT0** pertraukimai tik kelia vėliavas, o `loop()` jas apdoroja ir valdo būsenų mašiną (PALEIDIMAS → VEIKIA ⇄ MATUOJA, VEIKIA → ISSAUGOTA → VEIKIA). `delay()` nenaudojamas. INT0 fiksuoja paspaudimo laiką `micros()` tikslumu, todėl matavimui netrukdo tai, ką tuo metu daro pagrindinė programa. EEPROM saugo 3 reikšmes (animacija, BPM, šviesumas) su magic, versija ir kontroline suma. Rašomi tik pasikeitę baitai, įrašymas atidedamas 1,5 s.

## Laiko biudžetas

| Veiksmas | Trukmė | Kada |
|----------|--------|------|
| Timer1 ISR | ~3–5 µs | Kas 1 ms (< 0,5 % CPU) |
| INT0 ISR | ~10 µs | Kiekvienas D2 lygio pasikeitimas |
| ANIM mygtuko tikrinimas | < 20 µs | Kas 10 ms |
| Matricos kadras (8 eilutės, LedControl) | ~1–1,5 ms | Kas 62–600 ms |
| LCD eilutės perrašymas | ~3–4 ms | Tik pasikeitus tekstui, ne dažniau kaip kas 100 ms |
| EEPROM įrašymas | ~3,3 ms × pakitę baitai (≤ 20 ms) | Tik po pakeitimų, ne matavimo metu |

Ilgiausias `loop()` žingsnis – EEPROM įrašymas (≤ 20 ms). Jis vyksta tik tada, kai vartotojas nieko nedaro, todėl matavimo nepaveikia. Paspaudimo laikas fiksuojamas pertraukime, todėl `loop()` vėlavimas paveikia tik rodymą, ne matavimą.

## Testai ir tikslumas

<!-- Užpildykite po bandymų. Metronomui tinka bet kuri telefono programėlė. -->

| Testas | Rezultatas |
|--------|-----------|
| Metronomas 60 BPM, 8 paspaudimai | Išmatuota: ___ BPM (paklaida ___ %) |
| Metronomas 90 BPM, 8 paspaudimai | Išmatuota: ___ BPM (paklaida ___ %) |
| Metronomas 120 BPM, 8 paspaudimai | Išmatuota: ___ BPM (paklaida ___ %) |
| Virpėjimas: 50 greitų TAP paspaudimų | Dvigubų suveikimų: ___ |
| Trumpas / ilgas ANIM paspaudimas, po 20 kartų | Klaidingų atpažinimų: ___ |
| EEPROM: pakeisti nustatymus, išjungti ir įjungti | Nustatymai išliko: taip / ne |
| EEPROM: sugadintas baitas (pakeistas magic) | Įkeltos numatytosios reikšmės: taip / ne |

Teorinė skiriamoji geba: laikas fiksuojamas 4 µs tikslumu, intervalas skaičiuojamas sveikomis milisekundėmis. Ties 120 BPM (500 ms) ±1 ms paklaida atitinka ±0,24 BPM. Didžiausią paklaidą lemia žmogaus paspaudimų netolygumas, todėl imamas 4 intervalų vidurkis.

## Žinomos problemos

- **Pirmi du paspaudimai** duoda tik vieną intervalą, todėl pradinis BPM būna netikslus. Rezultatas stabilizuojasi po 4–5 paspaudimų.
- **Matricos orientacija** priklauso nuo to, kaip pasuktas modulis. Apvertus modulį, kadrus reikia apversti kode.
- **LCD neturi lietuviškų raidžių**, todėl tekstai be diakritikų (`Sirdis`, `Sviesa`).
- **EEPROM įrašymas** trumpam (≤ 20 ms) sustabdo `loop()`. Tai neturi įtakos matavimui, bet teoriškai gali vėluoti vienas animacijos kadras.
- **BPM ribos** 30–240. Lėtesnis arba greitesnis taksėjimas ignoruojamas.
