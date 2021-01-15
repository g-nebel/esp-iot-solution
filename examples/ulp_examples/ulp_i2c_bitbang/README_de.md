(Englische Version wird demnächst veröffentlicht)

# ULP-Coprozessor-Software-Emulation der I2C-Lichtsensorauslesung im Ultra-Low-Power-Modus (Assembly)
Dieser Artikel ist ein Beispiel für einen ULP-Coprozessor, der einen I2C-Host simuliert, der den Lichtsensor BH1750 im Low-Power-Modus ausliest  

## 1. I2C Pin Belegung
Das per Software simulierte I2C-Beispiel verwendet zwei Pins RTC_GPIO9, RTC_GPIO8, die entsprechenden GPIO-Pins sind wie folgt 

|I2C_PIN|RTC_GPIO|GPIO|
|---|---|---|
|I2C_SCL|RTC_GPIO9|GPIO32|
|I2C_SDA|RTC_GPIO8|GPIO33|

## 2. die Konfiguration der Softwareumgebung
Die C-Kompilierumgebung für ESP32 ist installiert und konfiguriert wie in [Linkadresse](https://docs.espressif.com/projects/esp-idf/en/stable/get-started/index.html#setup-toolchain) beschrieben. Der ULP-Coprozessor unterstützt derzeit nur die Assembler-Programmierung, daher müssen Sie auch die Assembly-Toolchain installieren. Im Folgenden wird die Installation und Konfiguration der Assembly-Toolchain beschrieben.
#### 2.1 Konfiguration der Montageumgebung
Um die Assembler-Toolchain für den ULP-Coprozessor zu konfigurieren, sind nur zwei Schritte zur Installation und Konfiguration erforderlich. /ulp.html) für weitere Informationen zur ULP-Programmierung
>* Schritt 1, laden Sie die Toolchain `binutils-esp32ulp toolchain` [link]( https://github.com/espressif/binutils-esp32ulp/wiki#downloads) herunter, entpacken Sie sie in das Verzeichnis, in dem Sie sie installieren möchten
>* Schritt 2, fügen Sie das Verzeichnis `bin` der Toolchain zur Systemumgebungsvariablen `PATH` hinzu. Wenn mein extrahiertes Verzeichnis zum Beispiel `/opt/esp32ulp-elf-binutils` ist, dann fügen Sie die Zeile `export PATH=/opt/esp32ulp-elf-binutils/bin:$PATH` in die letzte Zeile der versteckten Datei `.bashrc` im /home-Verzeichnis ein Speichern Sie die geschlossene Datei und verwenden Sie den Befehl `source .bashrc`, damit die obigen Umgebungsvariablen wirksam werden

#### 2.2 Konfigurieren des Compiler-Brenners
Nachdem nun die Assembly-Umgebung installiert ist, führen Sie die folgenden Befehle im Verzeichnis ulp_i2c_bitbang/ aus, um die Standardkonfiguration zu konfigurieren und das Programm zu kompilieren und zu brennen.

Fabrikat:
>* make defconfig
>* make all -j8 && make flash monitor

CMake
>* idf.py defconfig
>* idf.py flash monitor

## 3. stack.S Stack-Makro Beschreibung
Da die aktuelle ULP-Coprozessor-Assembly-Umgebung keine Operationen in Bezug auf Unterfunktionsaufrufe und das Speichern des Rückgabestapels implementiert, muss dieser Teil vom Anwender implementiert und konstruiert werden. Also opfern wir das R3-Register, um als Stack-Pointer unter den vier allgemeinen Registern verwendet zu werden und implementieren vier Makros PUSH, POP, PSR, RET über Stack-Operationen, die oben genannten Makros sind im Stack.

1. push, diese Operation besteht darin, die Daten von Rx (R0, R1, R2) in den Stack zu drücken, auf den R3 zeigt, der Stack-Zeiger wächst nach unten
```
.macro push rx
	st \rx,r3,0
	sub r3,r3,1
.endm
```
2. pop, die Daten auf dem Stack, auf die R3 zeigt, in Rx(R0,R1,R2) speichern, Stapelzeiger zurück
```
.macro pop rx
	r3,r3,1 addieren
	ld \rx,r3,0
.endm
```
3. psr, dieses Makro dient dazu, die Adresse zu berechnen, die die Funktion nach der Ausführung zurückgeben muss, und sie auf dem Stack zu speichern, d. h. die aktuelle Adresse plus Offset 16 ist die Adresse, die die Unterfunktion nach der Sprungausführung zurückgeben muss
```
.macro psr 
	.set addr, (. +16)
	move r1, addr
	r1 drücken
.endm
```
4. ret, dieses Makro entnimmt die auf dem Stack gespeicherte Rücksprungadresse und springt zu dieser Adresse
```
.macro ret 
	pop r1
	Sprung r1
.endm
```

## 4. i2c.S Baugruppen-Funktionsbeschreibung
Es steht dem Anwender frei, die Assembler-Funktionen in der folgenden Tabelle aufzurufen, um sie zu eigenen I2C-Lese-/Schreibfunktionen zu kombinieren, und er sollte darauf achten, vor dem Aufruf der Unterfunktionen das Makro psr einzufügen, um die Rücksprungadresse der Assembler-Unterfunktionen zu erhalten.

|Nr.|Assemblerfunktion | Eingangsargumente | Rückgabeargumente | Übergabeform | Funktionsbeschreibung|
|:---:|:---|:---:|:---:|:---:|---|
|1|i2c_start_cond|i2c_didInit, i2c_started|i2c_didInit, i2c_started|globale Variablen|Prüfen, ob I2C-Pin initialisiert ist und I2C_START erfolgt ist, I2C-Pin initialisieren und I2C_START-Signal senden|
|2|i2c_stop_cond| - |i2c_started |globale Variable |sende I2C_STOP Signal, lösche i2c_started Wert|
|3|i2c_write_bit|R0| - |register|Senden von BIT0 oder BIT1 je nach Wert von R0 |
|4|i2c_read_bit| - | R0 |register| Liest 1 Bit Daten von der SDA-Datenleitung und speichert es in R0|
|5|i2c_write_byte| R2 |R0|register|Senden Sie die Daten in R2, lesen Sie das ACK des Slaves und speichern Sie es in R0, und geben Sie 0 zurück, wenn das ACK vom Slave empfangen wird |
|6|i2c_read_byte| R0 |R0|register| Liest 1 Byte Daten von der SDA-Datenleitung und antwortet ACK oder NACK entsprechend dem übergebenen Wert von R0, die zurückgegebenen Daten werden in R0 gespeichert |# 5.

## 5. i2c_dev.S Assembly Funktion
|Nr.|Assemblerfunktion | Eingangsargumente | Rückgabeargumente | Form der Übergabe | Funktionsbeschreibung
|:---:|:---|:---:|:---:|:---:|--:|---|
|1|Read_BH1750|BH1750_ADDR_R, R2| R2 |Stack, globale Variable|Read 16bit value from I2C BUS|
|2|Cmd_Write_BH1750|Dev_addr, Command| - |Stack | Write Command|
|3|Start_BH1750|Dev_addr, Befehl| - |Stack|Betriebsart BH1750 einstellen|
|4|Aufgabe_BH1750| - | - | - | BH1750 Lichtwerte einmalig lesen
|5|waitMs|R2| - |Register|R2 Übergabe der zu verzögernden MS, z. B. R2 = 5, dann verzögern 5MS |

## 6. den Lichtsensor auslesen
Das Timing des I2C-Setups des BH1750 ist wie folgt
![](../../../documents/_static/ulp_i2c_bitbang/i2c_command.png)

I2C-Messung des Lichtergebnisses des BH1750 mit dem folgenden Timing
![](../../../documents/_static/ulp_i2c_bitbang/i2c_get.png)

Der ULP-Coprozessor liest die Lichtintensität des BH1750 über die Software-Emulation von I2C aus und gibt sie in der folgenden Abbildung aus, wobei unterschiedliche Lichtwerte durch Maskieren des Rasterfensters des Sensors erhalten werden können.

![](../../../documents/_static/ulp_i2c_bitbang/light_result.png)

## 7. Zusammenfassung
Wir haben 4 Makros für die Stack-Verwaltung konstruiert, um Funktionssprünge und Funktionsrücksprünge zu erleichtern, was es relativ einfach macht, große, komplexe ESP32-Ultra-Low-Power-Anwendungen zu implementieren.

 