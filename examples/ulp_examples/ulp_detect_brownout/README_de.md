# ULP Coprozessor erkennt ESP32-Brownout
In diesem Beispiel wird die ADC-Messfunktion des ULP-Coprozessors zum Polling im Hintergrund verwendet, um den Spannungswert des Pins VDD33 zu erkennen und bei Spannungsinstabilität entsprechend zu verarbeiten.

### 1. Hardware-Anschluss

Beachten Sie das [ESP32 Technical Manual](https://www.espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf), den [ADC] des ULP( https://docs.espressif.com/projects/esp-idf/en/stable/api-guides/ulp_instruction_set.html#adc-do-measurement-with-adc) kann der Befehl ADC Kanäle und GPIO-Pins sind unten dargestellt.

<div align=center>
<img src="../../../documents/_static/ulp_detect_brownout/ulp_adc_chn.png" width="600">
</div>

In diesem Beispiel wird Kanal 7 (GPIO34) des SAR ADC1 ausgewählt und über einen externen Jumper mit Pin VDD33 zur Spannungsmessung verbunden, wie in der folgenden Abbildung gezeigt.

<div align=center>
<img src="../../../documents/_static/ulp_detect_brownout/ulp_adc_gpio.jpg" width="400">
</div>

### 2. wie die Software funktioniert 
Nachdem die Haupt-CPU im **Non-Sleep-Modus** gestartet ist, müssen folgende Einstellungen vorgenommen werden.

- Konfigurieren Sie SAR ADC1 und geben Sie den ULP-Zugriff darauf frei
- Initialisieren und Aktivieren des RTC-Watchdog-Timers
- Laden und starten Sie das ULP-Programm

Wenn das ULP-Programm startet, fragt es ständig den Spannungswert des Pins VDD33 ab.

- Wenn die Spannung unter dem programmierten Brownout-Schwellenwert liegt, wird der digitale Kern abgeschaltet, die HF-Schaltung wird abgeschaltet und die CPU wird angehalten
- Steigt der Spannungswert von Low auf die eingestellte Brownout-Schwelle, setzt der ULP das gesamte System über den RTC-WDT-Timeout zurück

> Hinweis: Da der ULP keinen Zugriff auf den digitalen Teil des Moduls hat, kann er die ADC-Kalibrierungsparameter in EFUSE nicht lesen, um die SAR-ADC1-Messungen zu kalibrieren, so dass die endgültigen SAR-ADC1-Messungen je nach Modultyp einige Abweichungen aufweisen werden.

### 3. assembly code

Die folgenden Unterabschnitte beschreiben den Zweck und die Implementierung des Assembly-Codes.

- Der ULP tastet den Spannungswert des Pins VDD33 mehrfach ab und mittelt ihn mit Hilfe der ADC-Anweisung.
``Montage
messen:
    /* Messen und Wert zum Akkumulator addieren */
    adc r1, 0, adc1_chn_6 + 1
    r0, r0, r1 addieren
    /* Schleifenzähler inkrementieren und Exit-Bedingung prüfen */
    stage_inc 1
    Sprünge messen, Überstichprobe, lt

    /* Akkumulator durch Oversample dividieren.
       Da sie als Zweierpotenz gewählt wird, verwenden Sie die Rechtsverschiebung */
    rsh r0, r0, oversample_power
    /* gemittelter Wert steht jetzt in r0, speichert ihn im letzten Ergebnis */
    move r3, letztes_Ergebnis
    st r0, r3, 0
```

- Wenn die vom ULP gelesene Spannung am VDD33-Pin unter den eingestellten Schwellenwert fällt, setzen Sie das Brownout-Flag-Bit und schalten gleichzeitig den digitalen Kern und die HF-Schaltung aus und stoppen die CPU: `` ` ` `
```
niedrige_Spannung:
    /* brownout_flag auf 1 setzen */
    move r3, brownout_flag
    r2, 1 verschieben
    st r2, r3, 0
    /* Digitalen Kern im Ruhezustand abschalten */
    WRITE_RTC_REG(RTC_CNTL_DIG_PWC_REG, RTC_CNTL_DG_WRAP_PD_EN_S, 1, 0)
    /* Wi-Fi im Ruhezustand abschalten */
    WRITE_RTC_REG(RTC_CNTL_DIG_PWC_REG, RTC_CNTL_WIFI_PD_EN_S, 1, 0)
    /* Software-Stillstand CPU */
    WRITE_RTC_REG(RTC_CNTL_SW_CPU_STALL_REG, RTC_CNTL_SW_STALL_PROCPU_C1_S, 6, 0x21)
    WRITE_RTC_REG(RTC_CNTL_SW_CPU_STALL_REG, RTC_CNTL_SW_STALL_APPCPU_C1_S, 6, 0x21)
    WRITE_RTC_REG(RTC_CNTL_OPTIONS0_REG, RTC_CNTL_SW_STALL_PROCPU_C0_S, 2, 2)
    WRITE_RTC_REG(RTC_CNTL_OPTIONS0_REG, RTC_CNTL_SW_STALL_APPCPU_C1_S, 2, 2)
    Sprung feed_dog
```

- Die Feeddog-Routine wird nur dann übersprungen, wenn der Spannungswert am VDD33-Pin **von niedrig auf den eingestellten Brownout-Schwellenwert** angehoben wird, wobei darauf gewartet wird, dass der RTC-Watchdog-Timer eine Zeitüberschreitung erfährt und der gesamte Chip zurückgesetzt wird. In allen anderen Fällen wird der RTC-WDT periodisch mit Hunden gefüttert: der

``Montage
feed_dog:
    /* Schreiben Sie 0x50d83aa1 in RTC_CNTL_WDTWPROTECT_REG, um die RTC-WDT-Register zu entsperren,
        siehe soc/rtc_cntl_reg.h für weitere Informationen */
    WRITE_RTC_REG(RTC_CNTL_WDTWPROTECT_REG, 0, 8, 0xa1)
    WRITE_RTC_REG(RTC_CNTL_WDTWPROTECT_REG, 8, 8, 0x3a)
    WRITE_RTC_REG(RTC_CNTL_WDTWPROTECT_REG, 16, 8, 0xd8)
    WRITE_RTC_REG(RTC_CNTL_WDTWPROTECT_REG, 24, 8, 0x50)
    /* RTC-WDT einspeisen */
    WRITE_RTC_REG(RTC_CNTL_WDTFEED_REG, RTC_CNTL_WDT_FEED_S, 1, 1)
    /* beliebige Daten in die RTC-WDT-Register schreiben */
    WRITE_RTC_REG(RTC_CNTL_WDTWPROTECT_REG, 0, 8, 0)
    halt
```

### 4. ergebnisse des Programmlaufs

Nach dem Start der Haupt-CPU werden die VDD33-Pin-Spannungswerte (ADC roh und konvertiert), die vom ULP gelesen werden, zeitgesteuert gedruckt. Der ULP-Coprozessor ist so eingestellt, dass er ein 100ms-Wakeup durchführt, um den VDD33-Pin-Spannungswert zu erkennen. Der Brownout-Schwellenwert ist in diesem Beispiel auf 2,76v eingestellt.

Im Folgenden finden Sie ein Protokoll des Programmlaufs.

````tex
rst:0x1 (POWERON_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
ets Jun 8 2016 00:22:57

rst:0x10 (RTCWDT_RTC_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
configsip: 0, SPIWP:0xee
clk_drv:0x00,q_drv:0x00,d_drv:0x00,cs0_drv:0x00,hd_drv:0x00,wp_drv:0x00
Modus:DIO, Taktteilung:2
load:0x3fff0018,len:4
load:0x3fff001c,len:5152
ho 0 tail 12 Raum 4
load:0x40078000,len:10768
load:0x40080400,len:6300
Eintrag 0x4008070c
Nicht ULP-Wakeup, ULP-Programm starten
ulp_low_thr: 3825
System im aktiven Modus [0s], Spannung: [0][0.000]
System im aktiven Modus [1s], Spannung: [4085][3.272]
System im aktiven Modus [2s], Spannung: [4091][3.289]
System im aktiven Modus [3s], Spannung: [4011][3.069]
System im aktiven Modus [4s], Spannung: [3980][2.994]
System im aktiven Modus [5s], Spannung: [3959][2.972]
System im aktiven Modus [6s], Spannung: [3952][2.964]
System im aktiven Modus [7s], Spannung: [3949][2.961]
System im aktiven Modus [8s], Spannung: [3926][2.937]
System im aktiven Modus [9s], Spannung: [3902][2.912]
System im aktiven Modus [10s], Spannung: [3868][2.804]
System im aktiven Modus [11s], Spannung: [3854][2.788]
System im aktiven Modus [12s], Spannung: [3853][2.787]
ets Jun 8 2016 00:22:57

rst:0x10 (RTCWDT_RTC_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
configsip: 0, SPIWP:0xee
clk_drv:0x00,q_drv:0x00,d_drv:0x00,cs0_drv:0x00,hd_drv:0x00,wp_drv:0x00
Modus:DIO, Taktteilung:2
load:0x3fff0018,len:4
load:0x3fff001c,len:5152
ho 0 tail 12 Raum 4
load:0x40078000,len:10768
load:0x40080400,len:6300
Eintrag 0x4008070c
Nicht ULP-Wakeup, ULP-Programm starten
ulp_low_thr: 3825
System im aktiven Modus [0s], Spannung: [0][0.000]
System im aktiven Modus [1s], Spannung: [3858][2.791]
System im aktiven Modus [2s], Spannung: [3859][2.792]
System im aktiven Modus [3s], Spannung: [3860][2.793]
System im aktiven Modus [4s], Spannung: [3858][2.791]
System im aktiven Modus [5s], Spannung: [3890][2.896]
System im aktiven Modus [6s], Spannung: [3898][2.907]
System im aktiven Modus [7s], Spannung: [3935][2.946]
System im aktiven Modus [8s], Spannung: [3971][2.984]
System im aktiven Modus [9s], Spannung: [4022][3.099]
````

Die Analyse der Protokolle zeigt, dass das ULP die CPU heruntergefahren hat, als es feststellte, dass der Spannungswert des Pins VDD33 unter dem Schwellenwert für Brownout lag, d. h. das System hat nach dem Drucken des folgenden Protokolls aufgehört zu drucken.

``tex
System im aktiven Modus [12s], Spannung: [3853][2.787]
```

Danach, wenn der Spannungswert des Pins VDD33 von niedrig auf den eingestellten Brownout-Schwellenwert zurückkehrt, stoppt der ULP den Dogfeed-Vorgang, der RTC-Watchdog-Timer läuft aus und setzt das gesamte System zurück, und das System läuft wieder.
