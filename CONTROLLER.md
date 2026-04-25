# Dubbele Batterij Hoofdcontroller

Dit document beschrijft de werking van `main` (Node-RED function code) op hoofdlijnen.

## Doel

De regeling balanceert tegelijk:

- geen onnodige netimport
- geen onwenselijke export
- lokaal opnemen van PV-overschot
- ruimte vrijhouden voor verwachte zon
- negatieve verkoopprijs vermijden
- negatieve aankoopprijs benutten
- Bat1 en Bat2 niet tegen elkaar laten werken
- kleine PV pas begrenzen als laatste redmiddel

## Verwachte input op `msg`

Minimaal:

- `p1` (kritiek; verplicht)
- `soc1` of `soc`
- `soc2`
- `nord.state` en/of `nord.attributes.data[]` (prijsinformatie)

Aanvullend (fallback naar 0 indien afwezig):

- `over`
- `pv_tomorrow`
- `pv12h`
- `pv24h`
- `wkleinezon`
- `p1q15`

## Veilige foutmodi

- Ongeldige `p1` ⇒ directe fail-safe met veilige outputs.
- Ontbrekende `soc1` én `soc2` ⇒ directe fail-safe.

## Prijslogica

Uit spotprijs wordt afgeleid:

- `prijsKoopNu`
- `prijsVerkoopNu`
- `q` (relatieve prijsscore 0..1)
- toekomstige negatieve uren
- beste toekomstige koopprijs
- slechtste toekomstige verkoopprijs

## Hoofdmodi

De controller prioriteert in deze volgorde:

1. negatieve koopprijs benutten
2. actueel PV-overschot absorberen bij negatieve verkoopprijs
3. nachtelijke anti-import bij veel zon morgen
4. vooraf leegmaken bij verwachte zon
5. normale prijs/importregeling

## Bat2-regeling

Bat2 stuurt via:

- `gdes` (vermogensdoel)
- `blo`
- `bup`

Belangrijk:

- `bup = -1` betekent laden blokkeren.
- `gdes` gaat via slew-rate naar het doel (geen sprongen).

### Bat2 response guard (nieuw)

Wanneer `soc2` ontbreekt kan extern nog steeds veilig begrensd worden door de batterijcontroller.
Deze code voegt een extra softwarelaag toe:

- Als `soc2` ontbreekt én Bat2-regeling actief is, wordt gekeken of `p1` voldoende volgt op het gevraagde setpoint (`|p1 - gdes|`).
- Bij aanhoudende mismatch gedurende meerdere cycli gaat een tijdelijke cooldown aan:
  - `gewensteGdes = 0`
  - `bup = -1`
  - `gdes` rustig richting 0
- Na cooldown probeert regeling opnieuw.

Configureerbaar via `flow.cfg`:

- `bat2_response_guard_enabled` (default `1`)
- `bat2_response_error_w` (default `1200`)
- `bat2_response_streak` (default `6`)
- `bat2_response_cooldown_ms` (default `300000`)
- `bat2_response_release_step_w` (default `300`)

Debugvelden:

- `bat2ResponsRegelaarActief`
- `bat2ResponsGuardActief`
- `bat2VolgtSetpoint`
- `bat2RegelFoutW`
- `bat2MismatchStreak`

## Bat1-regeling

Bat1 is secundair:

- ontlaadt bij piek/duur/pre-discharge situaties
- laadt bij goedkope/negatieve prijs of absorptie-situatie
- blokkeert laden bij conflicterende modi (nacht anti-import, vooraf leegmaken, etc.)

## Kleine PV-curtailment

Kleine PV wordt alleen begrensd als export ongewenst is en batterijen overschot niet voldoende opnemen.
Het limiet gaat met slew-rate en blijft binnen `0..PV2_MAX_W`.

## Outputs

De functie retourneert 7 outputs:

1. Bat1 charge
2. Bat1 discharge
3. Bat2 `gdes`
4. Bat2 `blo`
5. Bat2 `bup`
6. uitgebreide debug
7. kleine PV limiet
