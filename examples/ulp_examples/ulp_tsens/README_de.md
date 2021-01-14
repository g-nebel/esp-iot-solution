# ULP-Coprozessor liest On-Chip-Temperatursensor im Low-Power-Modus

Dieser Artikel beschreibt, wie der ULP-Coprozessor den On-Chip-Temperatursensor TSENS im Low-Power-Modus ausliest 

## 1. On-Chip-Temperatursensor
Der ESP32-Chip verfügt über einen On-Chip-Temperatursensor, der Temperaturen zwischen -40°C und 125°C misst. Durch die Messung der von der Umgebung (z.B. WiFi-Schaltung) erzeugten Wärme können Sie die Lastverluste und die Umgebungstemperatur des Systems grob abschätzen.
Es sollte beachtet werden, dass die Temperatur-Spannungs-Charakteristik jedes Chips aufgrund von Prozessabweichungen variieren kann, so dass das Hauptanwendungsszenario für den Temperatursensor darin besteht, den Betrag der Temperaturschwankung und nicht den absoluten Wert der Temperatur zu messen.

## Beispiel für die Messung des Temperatursensors
Dieser Beispiel-ULP-Coprozessor wacht alle 3 S auf. Nach dem Aufwachen liest er den On-Chip-Temperatursensorwert über die Anweisung `TSENS` im Low-Power-Modus aus, weckt dann die Haupt-CPU auf, um den erfassten Wert auszudrucken und geht wieder in den DeepSleep-Zustand über.
Die folgende Abbildung zeigt, dass das Terminal einen Wert von 142 bis 144 ausgibt, wenn man mit Daumen und Zeigefinger auf die Hauptplatine drückt und die Wärme vom Finger über die Platine auf den Chip überträgt (Hinweis: Dieser Wert kann bei verschiedenen Chips unterschiedlich sein).

![](../../../documents/_static/ulp_tsens/tsens.png)

Es ist zu beachten, dass der TSENS-Wert ein Byte ist, im Bereich von 0 - 255 liegt und sein Wert ungefähr linear mit der Umgebungstemperatur variiert, so dass der Benutzer den entsprechenden externen Temperaturwert definieren und messen muss.

## 3. Software
Die C-Compiler-Umgebung für ESP32 wird installiert und konfiguriert wie in [link](https://docs.espressif.com/projects/esp-idf/en/stable/get-started/index.html#setup-toolchain) beschrieben, zusätzlich Der ULP-Coprozessor unterstützt derzeit nur die Assembler-Programmierung, daher müssen Sie auch die Assembly-Toolchain installieren. Im Folgenden wird die Installation und Konfiguration der Assembly-Toolchain beschrieben.
#### 3.1 Konfiguration der Montageumgebung
Um die Assembler-Toolchain für den ULP-Coprozessor zu konfigurieren, sind nur zwei Schritte zur Installation und Konfiguration erforderlich. /ulp.html) für weitere Informationen zur ULP-Programmierung
>* Schritt 1, laden Sie die Toolchain `binutils-esp32ulp toolchain` [link]( https://github.com/espressif/binutils-esp32ulp/wiki#downloads) herunter, entpacken Sie sie in das Verzeichnis, in dem Sie sie installieren möchten
>* Schritt 2, fügen Sie das Verzeichnis `bin` der Toolchain zur Systemumgebungsvariablen `PATH` hinzu. Wenn mein extrahiertes Verzeichnis zum Beispiel `/opt/esp32ulp-elf-binutils` ist, dann fügen Sie die Zeile `export PATH=/opt/esp32ulp-elf-binutils/bin:$PATH` in die letzte Zeile der versteckten Datei `.bashrc` im /home-Verzeichnis ein Speichern Sie die geschlossene Datei und verwenden Sie den Befehl `source .bashrc`, damit die obigen Umgebungsvariablen wirksam werden

#### 3.2 Konfigurieren des Compiler-Brenners
Nachdem die Assembler-Kompilierumgebung installiert ist, führen Sie die folgenden Befehle im Verzeichnis ulp_tsens/ aus, um das Programm zu konfigurieren, zu kompilieren und zu brennen.

Fabrikat:
>* make defconfig
>* make all -j8 && make flash monitor

CMake
>* idf.py defconfig
>* idf.py flash monitor


#### 3.3 Einführung in den Assembler-Code

Der ULP verfügt bereits über einen eingebauten Assembler-Befehl `TSENS` zum Auslesen des On-Chip-Temperatursensors, der im folgenden Beispiel verwendet wird:
```
 TSENS R1, 1000 /* Temperatursensor für 1000 Zyklen messen, und Ergebnis in R1 speichern */                  
```

Beachten Sie, dass dieser ``TSENS``-Befehl allein nicht den tatsächlichen Wert des Temperatursensors ermitteln kann, da die entsprechende Schaltung im Chip zu diesem Zeitpunkt ausgeschaltet ist. Sie müssen das ``SENS_FORCE_XPD_SAR``-Register im ``SENS_SAR_MEAS_WAIT2_REG``-Register manuell setzen, um die entsprechende Schaltung einzuschalten.

```
	/* Einschalten erzwingen */
	WRITE_RTC_REG(SENS_SAR_MEAS_WAIT2_REG,SENS_FORCE_XPD_SAR_S,2,SENS_FORCE_XPD_SAR_PU)
    
```

Danach wird der On-Chip-Temperaturtransmitter mit dem Befehl `` TSENS`` ausgelesen und der Temperaturwert an ``tsens_value`` übergeben 
```
	tsens r0, 1000
	move r3, tsens_wert
	st r0, r3, 0	
	Sprung wake_up

```
Next Polling Register `RTC_CNTL_DIAG0_REG` für Haupt-CPU-Wakeup

```
	/* ULP wieder in den Ruhezustand bringen */
	.global exit
Ausgang:
	halt

	.global wake_up
wach_auf:
	/* Prüfen, ob der SoC aufgeweckt werden kann */
	READ_RTC_REG(RTC_CNTL_DIAG0_REG, 19, 1)
	und r0, r0, 1
	Sprungausgang, eq

	/* Aufwachen des SoC und Stoppen des ULP-Programms */
	wecken
	/* Stoppen Sie den Wakeup-Timer, damit er ULP nicht neu startet */
	WRITE_RTC_FIELD(RTC_CNTL_STATE0_REG, RTC_CNTL_ULP_CP_SLP_TIMER_EN, 0)
	halt
```
## 4. aktuelle Schätzung
In diesem Beispiel arbeitet der ULP-Coprozessor 2 ms lang mit 1,4 mA, der ULP-Coprozessor schläft 3 s lang, und der durchschnittliche Strom beträgt 5 uA. Berechnen wir die Stromdaten nach dem 1ms-Abtastpunkt, erhalten wir den durchschnittlichen Strom des gesamten Prozesses von etwa 5,9uA 

## 5. Zusammenfassung
Außerdem schaltet der ULP-Coprozessor, wenn er wieder in den Ruhezustand übergeht, alle Schaltkreise mit Ausnahme der 150K-Wake-up-Clock ab. Wenn der ULP-Coprozessor also das nächste Mal aufwacht, muss er immer noch das Register `SENS_FORCE_XPD_SAR` setzen, um die relevanten internen Schaltkreise einzuschalten (POWER UP).


 