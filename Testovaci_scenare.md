\# Testovací scénáře a kritéria úspěšnosti



\## Hardwarové testovací prostředí

\- Místo: Praha, údolí, jižní svah.

\- Podmínky: Teplotní rozsah -7 °C až +20 °C, vlhkost až 80 %, přítomnost volně žijících zvířat (kočky, srnčí zvěř, lišky, divoká prasata).



\## Testovací scénáře



| ID | Scénář | Očekávané chování |

|----|----------|---------------------|

| TC01 | Kočka (≤5 kg) šlápne na váhovou podložku | PIR probudí systém, naměřená hmotnost <35 kg → zaznamenáno, kamera se nespustí, systém přejde do režimu spánku. |

| TC02 | Srnec (20–35 kg) šlápne na váhovou podložku | Stejně jako TC01 – zaznamenáno, bez kamery. |

| TC03 | Dospělý člověk (>40 kg) šlápne na váhovou podložku | PIR probudí systém, naměřená hmotnost >40 kg → kamera pořídí 3 fotografie nebo 10s video, uloží na SD kartu, zaznamená událost, volitelně odešle do cloudu, poté přejde do režimu spánku. |

| TC04 | Vítr / listí spustí PIR, žádná hmotnost | PIR probudí systém, časový limit (10 s) bez změny hmotnosti → režim spánku, žádný záznam. |

| TC05 | Váha zatížena po delší dobu (např. zvíře zůstává) | Systém změří hmotnost, pokud po 30 s nedojde ke změně → režim spánku (žádné opakované spuštění kamery). |

| TC06 | Změna teploty z 0 °C na 20 °C | Automatická tara (EMA) udržuje stabilní nulový offset → žádné falešné údaje o hmotnosti. |

| TC07 | Zapnutí po odpojení baterie | Systém se spustí, načte kalibraci z EEPROM, přejde do hlubokého spánku, obnoví normální provoz. |

| TC08 | Spuštění kamery v noci | IR-LED (je-li přítomna) osvětluje scénu; obraz se ukládá s časovým razítkem. |



\## Kritéria úspěšnosti

\- \*\*Míra falešných poplachů:\*\* 0 spuštění kamery způsobených divokou zvěří nebo vlivy prostředí během 7 po sobě jdoucích dnů.

\- \*\*Míra detekce:\*\* Zachycení 100 % vniknutí osob (≥40 kg).

\- \*\*Spotřeba energie:\*\* Průměrná spotřeba <1 Wh/den (včetně režimu hlubokého spánku a občasného probuzení).

\- \*\*Stabilita:\*\* Žádné zablokování systému ani selhání SD karty během testovacího období.



\## Postup kalibrace

1\. Umístěte závaží (20 kg, 50 kg) na podložku.

2\. Zaznamenejte měření HX711.

3\. Vypočítejte měřítko a offset.

4\. Uložte do EEPROM.

5\. Ověřte pomocí testovacího závaží o hmotnosti 40 kg.



