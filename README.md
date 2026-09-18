Task Report: Remove Default Credentials and Secure Installer Logging
Work Item

Check where this part of code is coming from and remove default credentials.

Dodatne zahteve:

odstraniti hardcodane credentiale iz Symbiot MeterConfigurator,
preveriti SMC setup custom action,
preveriti MSI installer logiranje (msiexec /i setup.msi /l*v log.txt),
odstraniti privzete credentiale iz MAS RAS driverja.
1. Analiza problema

Med pregledom kode sem našel hardcodane SQL credentiale v:

Plain Text
1
DatabaseManager\SetupData.cs
2
`
Show more lines

Originalna implementacija:

C#
1
public const string Sep2MeterViewDbLogin = @"sa";
2
public const string Sep2MeterViewDbPassword = @"?sa\AV3S1\11?";
Show more lines

Credentiala sta bila uporabljena v:

Plain Text
1
DatabaseManager\DbUpgrade.cs
Show more lines

za migracijo stare SEP2 SQL Server 2008 baze.

2. Raziskava obstoječe implementacije

Pregledana je bila celotna migracijska logika v:

Plain Text
1
DatabaseManager\DbUpgrade.cs
2
``
Show more lines

Najdene so bile uporabe:

C#
1
CopyPreviousDbToCurrentDb(
2
...,
3
SetupData.Sep2MeterViewDbLogin,
4
SetupData.Sep2MeterViewDbPassword,
5
...
6
);
Show more lines

ter:

C#
1
var conn = new ServerConnection
2
{
3
LoginSecure = false,
4
Login = SetupData.Sep2MeterViewDbLogin,
5
Password = SetupData.Sep2MeterViewDbPassword
6
};
Show more lines
3. Ugotovitev

V isti kodi je bilo ugotovljeno, da sistem že podpira Windows Authentication.

Primer:

C#
1
CopyPreviousDbToCurrentDb(
2
previousServerInstance,
3
latestPreviousDbName,
4
...,
5
true,
6
null,
7
null,
8
...
9
);
Show more lines

in:

C#
1
connection.LoginSecure = true;
Show more lines

Dodatno je bilo v:

Plain Text
1
DatabaseManager\SetupData.cs
Show more lines

ugotovljeno, da trenutna MeterConfigurator baza uporablja:

C#
1
Integrated Security=SSPI
Show more lines

kar pomeni Windows Authentication.

4. Izvedene spremembe
4.1 Odstranitev SQL credentialov iz migracije

V:

Plain Text
1
DatabaseManager\DbUpgrade.cs
Show more lines

je bilo:

C#
1
false,
2
SetupData.Sep2MeterViewDbLogin,
3
SetupData.Sep2MeterViewDbPassword
Show more lines

zamenjano z:

C#
1
true,
2
null,
3
null
Show more lines

S tem migracija uporablja Windows Authentication.

4.2 Posodobitev preverjanja obstoja baze

Originalno:

C#
1
var conn = new ServerConnection
2
{
3
ServerInstance = SetupData.Sep2MeterViewDbServerInstance,
4
LoginSecure = false,
5
Login = SetupData.Sep2MeterViewDbLogin,
6
Password = SetupData.Sep2MeterViewDbPassword
7
};
Show more lines

Spremenjeno:

C#
1
var conn = new ServerConnection
2
{
3
ServerInstance = SetupData.Sep2MeterViewDbServerInstance,
4
LoginSecure = true
5
};
Show more lines
4.3 Odstranitev mrtve kode

Po odstranitvi vseh referenc sta bili iz:

Plain Text
1
DatabaseManager\SetupData.cs
Show more lines

odstranjeni vrstici:

C#
1
public const string Sep2MeterViewDbLogin = @"sa";
2
public const string Sep2MeterViewDbPassword = @"?sa\AV3S1\11?";
Show more lines

Preverjeno z:

PowerShell
1
Select-String "Sep2MeterViewDbLogin"
2
Select-String "Sep2MeterViewDbPassword"
Show more lines

Rezultat:

Plain Text
1
0 references found
Show more lines
5. MSI Logging Investigation

Pregledani so bili vsi:

C#
1
session.Log(...)
Show more lines

klici.

Posebej pregledano:

Plain Text
1
Setups\MeterViewCA\MVCustomActions.cs
Show more lines

Najdeni so bili:

C#
1
session.Log(e.ToString());
Show more lines

in:

C#
1
e.Message
Show more lines

Ti predstavljajo potencialno mesto, kjer bi se lahko občutljivi podatki zapisali v MSI log.

6. Dodana zaščita za maskiranje exceptionov

V:

Plain Text
1
DatabaseManager\DbUpgrade.cs
Show more lines

je bila dodana metoda:

C#
1
MaskSensitiveException(Exception ex)
Show more lines

Namen:

maskiranje uporabniških imen,
maskiranje gesel,
maskiranje connection stringov,

pred zapisom v installer log.

7. MSI Installer Test

Zgrajen installer:

Plain Text
1
SymbiotMeterConfigurator.msi
Show more lines

Installer zagnan z:

BAT
1
msiexec /i SymbiotMeterConfigurator.msi /l*v C:\Temp\smc.log
Show more lines

Generirana log datoteka:

Plain Text
1
C:\Temp\smc.log
Show more lines

Velikost:

Plain Text
1
3.7 MB
Show more lines
8. Rezultat preverjanja log datoteke

Preverjeno:

PowerShell
1
Select-String "AV3S1"
2
Select-String "remoteie"
3
Select-String "p2lpc00000"
4
Select-String "Password"
5
Select-String "User ID"
Show more lines

Rezultat:

Plain Text
1
No matches found
Show more lines

Zaključek:

SQL credentiali niso prisotni v MSI logu.
RAS credentiali niso prisotni v MSI logu.
Ni bilo mogoče reproducirati izpostavitve gesel preko msiexec logginga.
9. Dodatne najdbe
SQL Express bootstrapper

Najdeno v:

Plain Text
1
Setups\SqlExpress2008\Binaries\SqlExpress2008 - MeterView\en\package.xml
Show more lines
XML
1
/sapwd="?sa\AV3S1\11?"
Show more lines

Čaka na potrditev, ali se ta bootstrapper še uporablja in ali ga je potrebno dodatno spremeniti.

RAS Driver

Najdeno v:

Plain Text
1
SEP2ConnectionServer\Drivers\RAS\Source\Code\Driver\RasDriver.cs
Show more lines
C#
1
private string m_userName = @"p2lpc00000";
2
private string m_password = @"remoteie";
Show more lines

Ugotovljeno je bilo, da driver že podpira prepis preko:

C#
1
case @"USERNAME":
2
case @"PASSWORD":
Show more lines

zato je predvidena odstranitev privzetih vrednosti.

FTP Driverji

Najdeno še v:

Plain Text
1
MeterConcentratorFTP
2
MeterConcentratorFTPDLC
Show more lines

kjer se uporablja:

C#
1
m_password = "remoteie";
Show more lines

To zahteva dodatno odločitev glede obsega sprememb.

Zaključek

Glavni del naloge je bil uspešno izveden:

odstranjeni hardcodani SQL credentiali,
migracija preklopljena na Windows Authentication,
odstranjene vse reference na SQL login/password,
preverjeno MSI logiranje,
identificirane dodatne lokacije z remoteie in p2lpc00000 za nadaljnjo obravnavo.
