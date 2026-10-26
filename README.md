# UniFi UDM Pro / Max - Konfiguracja IPTV (Multicast, VLAN 438) z wkładki GPON lub zewnętrznego terminala ONT (np. ZTE 601) na WAN

Poradnik jak przepuścić oparty na multicastach ruch telewizyjny (VLAN 438) z wkładki GPON (np. Leox) lub zewnętrznego terminala ONT (np. ZTE 601) działającego w trybie bridge. Rozwiązanie sprawdza się w scenariuszu, gdzie sprzęt operatora jest wpięty prosto w port WAN routera UDM Pro / Max, a sesja PPPoE zestawiana jest bezpośrednio na urządzeniu UniFi. System UniFi domyślnie blokuje zmostkowanie L2 portu WAN z siecią LAN, co rozwiązujemy poniższym skryptem startowym.

## Wymagania
* Router: UDM Pro, UDM SE lub UDM Pro Max (skrypt operuje na interfejsie WAN).
* Urządzenie dostępowe GPON lub ONT (np. wkładka Leox LXT-010H-D lub ZTE 601) pracujące w trybie bridge.
* Włączony dostęp SSH w konsoli UniFi OS (OS Settings -> Console Settings -> SSH).

## Krok 1: Przygotowanie sieci w panelu UniFi
Zanim przejdziesz do terminala, musisz przygotować sieć lokalną:
1. Przejdź do **Settings -> Networks** i utwórz nową sieć.
2. Nazwij ją np. `IPTV`.
3. Jako router/typ wybierz **Third-Party Gateway** (lub *VLAN Only*).
4. Wpisz odpowiedni **VLAN ID** (np. `438`).
5. Przypisz utworzoną sieć jako **Native / Primary Network** do portu w switchu lub Access Poincie (np. U7 In-Wall), do którego wepniesz dekoder.

## Krok 2: Instalacja obsługi skryptów startowych (On-Boot)
Połącz się z routerem przez SSH (np. `ssh root@192.168.10.1`) i zainstaluj mechanizm UDM Boot Script, który pozwoli skryptom przetrwać restart maszyny:

```bash
curl -fsL "[https://raw.githubusercontent.com/unifi-utilities/unifios-utilities/main/on-boot-script/remote_install.sh](https://raw.githubusercontent.com/unifi-utilities/unifios-utilities/main/on-boot-script/remote_install.sh)" | /bin/sh
```

## Krok 3: Ręczne utworzenie folderu i skryptu
Aby uniknąć błędu E212: Can't open file for writing, najpierw utwórz wymagany katalog, a następnie stwórz plik za pomocą wbudowanego edytora vi.
```bash
mkdir -p /data/on_boot.d
vi /data/on_boot.d/438-iptv-vlan.sh
```

Gdy otworzy się edytor:
Wciśnij klawisz i, aby wejść w tryb edycji (na dole pojawi się -- INSERT --).
Wklej poniższy kod (jeśli Twoja wkładka jest w porcie SFP+ (eth9), a VLAN to 438):
```bash
#!/bin/sh
# Mostek L2 dla IPTV (VLAN 438) z portu WAN SFP+ do LAN
ip link add link eth9 name eth9.438 type vlan id 438
ip link set dev eth9.438 up
ip link set dev eth9.438 master br438
```

Wciśnij klawisz Esc, aby wyjść z trybu wprowadzania.

Wpisz :wq i wciśnij Enter, aby zapisać plik i zamknąć edytor.

## Krok 4: Nadanie uprawnień i sprawdzenie
Nadaj skryptowi prawa do uruchamiania:
```bash
chmod +x /data/on_boot.d/438-iptv-vlan.sh
```

Możesz zweryfikować poprawne zapisanie i uprawnienia poleceniami:
```bash
cat /data/on_boot.d/438-iptv-vlan.sh
ls -l /data/on_boot.d/438-iptv-vlan.sh
```

(W wyniku ls -l powinieneś zobaczyć flagę -rwxr-xr-x, co oznacza gotowość do uruchomienia).

## Krok 5: Ręczne wywołanie skryptu (bez restartowania routera)
Aby telewizja ruszyła od razu w bieżącej sesji, wpisz:
```bash
sh /data/on_boot.d/438-iptv-vlan.sh
```

Sprawdzenie, czy wirtualny mostek połączył interfejsy:
```bash
brctl show br438
```

Jeśli obok interfejsów portów lokalnych widzisz na liście eth9.438, to konfiguracja działa. Od teraz po każdym restarcie routera telewizja będzie uruchamiać się automatycznie.
