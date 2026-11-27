# DCSync útok

## Detekce
| ID Pravidla | Kde detekováno | Log | Detekovaná akce |
|-------------|----------------|-----|-----------------|
| 110001 | Domain Controler | Windows Security event | DC byl požádán o replikaci (sync) dat|

## Reakce

### 0. Kontrola false positive
Pokud je původcem DCSync synchronizační služba z IP adresy legitimního DC (Domain Controller), jedná se o false positive.

IP adresy legitimních DC:
- **TODO: IP adresy DC**

Služby provádějící synchronizaci DC:
- *například: AD DS Connector v MS ENTRA.  Účet má prefix `MSOL_`*

### 1. Zjištění stavu
1. Kontaktování administrátorů, zda je daná komunikace validní. Pokud ne, nahlášení incidentu.
2. V síťovém logu hledejte komunikaci přes DRSUAPI.
    - Filtr ve Wireshark pro detekci DRSUAPI: `dcerpc.cn_bind_to_uuid == e3514235-4b06-11d1-ab04-00c04fc2dcd2`.
    - Filtr ve Wireshark pro `GetNCChanges` requesty: `drsuapi.opnum == 3`.
    - Nejprve bude inicializace spojení přes port TCP/135 a následně komunikace přes některý z vyšších portů.
3. Hledání souvisejících eventů
    - Hledejte napříč dalšími alerty/eventy `SubjectLogonId`, které je v původním alertu. Např.:
        - EventID 4769 (Kerberos ticket requested) a pole `ipAddress` obsahující IP adresu kompromitovaného stroje.
        - V EventID 4624 (Network Logon) koreluje pole `TargetLogonId` se `SubjectLogonId`.

### 2. Izolace
Ve spolupráci s příslušnými administrátory:
1. Deaktivace účtu spouštějícího DCSync.
    - Pole `SubjectUserName` v eventu s ID 4662 udává účet, který DCSync spustil.
2. Izolace stanice, ze které běžel DCSync.
    - Nastavení firewall pravidel pro úplnou izolaci napadeného stroje - přerušení veškerých spojení.

### 3. Reset přihlšovacích údajů
Ve spolupráci s příslušnými administrátory:
1. [Reset hesla](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-reset-the-krbtgt-password) pro účet `krbtgt`:
    1. *Start* -> *Control Panel* -> *Administrative Tools* -> *Active Directory Users and Computers*
    2. Vyberte *View* -> *Advanced Features* -> dvojité kliknutí na domain controler -> vyberte *Users* -> klikněte pravým talčítkem na účet `krbtgt` -> zvolte *Reset Password*
    * Po vyřešení útoku proveďte změnu hesla ještě jednou. (Opětovný reset je možný nejdříve po 10 hodinách.)
2. Reset hesel privilegovaných účtů a dalších služeb.
3. Reset hesla kompromitovaného účtu.

### 4. Zjištění rozsahu útoku 
1. Prozkoumání napadeného stroje - nástroje *mimikatz*, *Impacket*. 
2. Review ostatních DC, jejich komunikace a změn (EventID 5136) v inkriminovaném období.
3. Review uživatelů a udělených privilegovaných práv. 
