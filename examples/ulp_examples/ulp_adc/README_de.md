# ULP ADC Beispiel

Dieses Dokument beschreibt ein Beispiel für das Lesen des Spannungswerts des NTC-Thermistors und die Berechnung des aktuellen Raumtemperaturwerts durch SAR_ADC im Ultra-Low-Power-Modus.

### 1. Sukzessive Annäherung Digital-Analog-Wandler
Der ULP-Coprozessor unterstützt ADC-Messbefehle im Ultra-Low-Power-Modus, eine komfortable Analog-Digital-Wandlung, die zahlreiche Möglichkeiten für Anwendungen im Ultra-Low-Power-Modus eröffnet.

Die folgende Tabelle zeigt die Kanal- und zugehörigen Pin-Informationen für den SAR ADC.

![](... /... /... /documents/_static/ulp_adc/3.png)

### 2. Hardware-Schaltplan
In diesem Beispiel verwenden wir den SAR ADC1 mit dem Eingangspin GPIO34, der dem SAR_MUX Kanal 7 entspricht, und den NTC-Thermistor Typ 3950 100K 1%.

Der NTC-Thermistortyp ist 3950 100K 1%.

![](... /... /... /documents/_static/ulp_adc/1.png)

### 3. Software-Teil
Der NTC-Thermistor ist nichtlinear mit der Temperatur, daher haben wir eine Tabelle mit ADC-Werten von -5°C bis 100°C im Assemblerprogramm mit einer Lookup-Methode erstellt.

Es folgt eine Teiltabelle der Spannungswerte. Um den Überlauf der Tabelle zu stoppen, setzen wir den Wert 0xffff am Ende der Tabelle als Markierung für das Ende der Tabelle.
```
.global thermistor_lut
thermistor_lut:
		.long 0x0D33 // -5 Grad Celsius
		.long 0x0D1F // -4 
		.long 0x0D0C // -3 
		.long 0x0CF8 // -2 
		.long 0x0CE4 // -1 
		.long 0x0CD0 // 0
		.long 0x0CBB // 1
		.long 0x0CA6 // 2
		.long 0x0C90 // 3
		.long 0x0C7A // 4
		.long 0x0C63 // 5
		.long 0x0C4B // 6
		.long 0x0C32 // 7
		.long 0x0C19 // 8
		.long 0x0BFE // 9
        ...
		.lang 0x0250 // 90
		.long 0x0240 // 91
		.long 0x0230 // 92
		.long 0x0222 // 93
		.long 0x0213 // 94
		.long 0x0205 // 95
		.long 0x01F8 // 96
		.long 0x01EB // 97
		.long 0x01DE // 98
		.long 0x01D2 // 99
		.long 0x01C5 // 100
		.long 0xffff // Ende
````

Der Assembler-Code liest zunächst die Spannung des NTC-Thermistors per ADC-Befehl aus, erhält den ADC-Wert nach der Analog-Digital-Wandlung und vergleicht dann diesen ADC-Wert mit dem Wert in der Tabelle thermistor_lut nach mehreren Erfassungen, um die aktuelle Raumtemperatur zu erhalten.

````
	move r3, Temperatur
	ld r1, r3, 0
	/* Temperatur in jeder Berechnungsschleife initialisieren */
	r1, 0 bewegen 
	st r1, r3, 0
	/* r3 als Zeiger für thermistor_lut verwenden */
	move r3, thermistor_lut 
look_up_table:
	/* Tabellendaten in R2 laden */
	ld r2, r3, 0 

	/* prüfen, ob am table_end */
	sub r1, r2, 0xffff
	/* wenn das Tabellenende getroffen wird, Sprung zu table_end */
	Sprung aus_dem_Bereich, eq
	
	/* thermistor_lut - last_result */
	sub r2, r0, r2
	/* adc_value < table_value */
	Sprung move_step, ov 
	/* adc_value > table_value, aktuellen Wert holen */
	Sprung wake_up
out_of_range:
	move r1, Temperatur
	ld r2, r1, 0
	/* Maximaler Bereich 100 Grad Celsius */
	r2 bewegen, 105 
	st r2, r1, 0
	Sprung wake_up
move_step:
	/* Zeiger um einen Schritt verschieben */
	r3, r3, 1 addieren 

	move r1, Temperatur
	ld r2, r1, 0
	/* Temperaturzählung 1 */
	r2, r2, 1 addieren 
	st r2, r1, 0
	Sprung look_up_table
````

Der obige Assembler-Code ist ein Tabellen-Lookup-Prozess, bei dem der Temperaturwert für jeden Schritt um eins addiert wird. Beachten Sie jedoch, dass diese Tabelle bei -5°C beginnt, so dass dieser Offset beim abschließenden Druck subtrahiert werden muss, um den korrekten Temperaturwert zu erhalten.

```C
/* Temperatur ab -5 °C zählen, also ulp_temperature minus 5 */
printf("Temperatur:%d ℃ \n", (int16_t)ulp_temperature - 5); 
```

### 4. endgültiger Effekt
Nach der Berechnung der Raumtemperatur wecken Sie die CPU auf, um den Wert der Raumtemperatur zu drucken. Der folgende Screenshot zeigt den Test mit einem Finger, der Wärme an den Thermistor abgibt, und den Druck der seriellen Schnittstelle.

![](... /... /... /documents/_static/ulp_adc/2.png)

