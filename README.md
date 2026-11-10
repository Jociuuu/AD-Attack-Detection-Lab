# Active Directory Attack & Detection Lab (MITRE ATT&CK)

Homelab typu **blue team**, w którym przeprowadzam pełny łańcuch ataku na domenę Active Directory i pokazuję, **jak każdy etap wygląda w logach i w SIEM-ie**. Każda technika jest zmapowana na [MITRE ATT&CK](https://attack.mitre.org/), opisana od strony ataku **i** detekcji.

> ⚠️ **To środowisko testowe (homelab).** Wszystkie adresy IP, nazwy kont i hasła (`vagrant123!`, `Password1` itd.) należą do odizolowanej sieci wirtualnej i nie mają żadnego związku z systemami produkcyjnymi.

---

## Po co ten projekt

Celem nie jest „włamać się”, tylko **zrozumieć, jak atak wygląda z perspektywy obrońcy**. Dla każdej techniki:

1. przeprowadzam atak z maszyny Kali,
2. sprawdzam, co zobaczył sieciowy IDS (Suricata) i hostowy IDS / SIEM (Wazuh),
3. opisuję, **dlaczego** detekcja zadziałała (albo nie) i jak ją poprawić.

To jest dokładnie praca analityka SOC, tylko widziana z obu stron naraz.

---

## Architektura labu

| Maszyna | Rola | System | IP |
|---|---|---|---|
| **DC01** | Kontroler domeny (Active Directory) | Windows Server 2022 | `192.168.56.10` |
| **WS01** | Stacja robocza w domenie | Windows 10 | `192.168.56.11` |
| **Kali** | Maszyna atakująca | Kali Linux | `192.168.56.112` |
| **Suricata** | Sieciowy IDS (NIDS) | Debian 13 | `192.168.56.113` |
| **Wazuh** | SIEM + hostowy IDS (HIDS) | — | `192.168.56.111` |

- **Domena:** `corplab.local` (NetBIOS: `CORPLAB`)
- **Sieć:** `192.168.56.0/24` (VirtualBox Host-Only)
- Suricata nasłuchuje na interfejsie w trybie promiscuous i przesyła alerty (`eve.json`) do Wazuha, więc **ruch sieciowy i logi hostów oglądam w jednym panelu**.

**NIDS vs HIDS:** Suricata widzi *ruch w sieci* (skany, połączenia), a agent Wazuha widzi *wnętrze maszyny* (logi, logowania, procesy). Dopiero oba razem dają pełny obraz — to motyw przewijający się przez cały projekt.

---

## Łańcuch ataku (kill chain)

Techniki tworzą jedną, realistyczną ścieżkę napastnika: od rozpoznania, przez kradzież poświadczeń, aż po przejęcie kontrolera domeny i obejście zabezpieczeń stacji roboczej.

| # | Taktyka | Technika | Opis |
|---|---|---|---|
| 1 | Reconnaissance | [T1595 Active Scanning](https://attack.mitre.org/techniques/T1595/) | Skan portów DC → **własna reguła Suricaty** |
| 2 | Credential Access | [T1110.001 Brute Force: Password Guessing](https://attack.mitre.org/techniques/T1110/001/) | Zgadywanie hasła SMB → korelacja w Wazuhu |
| 3 | Discovery | [T1087 Account Discovery](https://attack.mitre.org/techniques/T1087/) | Wylistowanie kont domeny |
| 4 | Credential Access | [T1558.004 AS-REP Roasting](https://attack.mitre.org/techniques/T1558/004/) | Kradzież i łamanie hasha Kerberos offline |
| 5 | Lateral Movement | [T1021 Remote Services](https://attack.mitre.org/techniques/T1021/) | Ruch boczny na DC przez WinRM |
| 6 | Defense Impairment | [T1686.003 Disable or Modify System Firewall](https://attack.mitre.org/) | Wyłączenie firewalla WS01 przez GPO → wejście na stację |

> Każda technika ma własny katalog z opisem (`README.md`) i zrzutami ekranu.

---

## Użyte narzędzia

- **Atak:** nmap, NetExec (nxc), Impacket (GetNPUsers), hashcat, evil-winrm
- **Detekcja:** Suricata (reguły ET Open + własne), Wazuh (HIDS, SIEM, korelacja, SCA)
- **Infrastruktura:** VirtualBox, Vagrant, Active Directory

---

## Najważniejsze wnioski (blue team)

- **Brak logu = brak detekcji.** Zanim cokolwiek wykryjesz, źródło musi to rejestrować — kilka technik wymagało najpierw włączenia audytu na Windowsie.
- **Pojedyncze zdarzenie vs korelacja.** Jedno nieudane logowanie to szum; dopiero *nagromadzenie* (reguła progowa) to alert o ataku.
- **Atak z ważnym hasłem jest cichy.** Discovery i ruch boczny wyglądają jak legalna praca admina — łapie się je po *kontekście* (kto, skąd, czym), nie po samym fakcie logowania.
- **Detekcja ma luki.** Część technik (skan, AS-REP Roasting) domyślny zestaw reguł przepuszcza — dlatego piszę **własne reguły** w Suricacie i Wazuhu.
