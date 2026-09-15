# Windows Server 2025 - Active Directory -laboratorio

Rakensin Windows Server 2025 -toimialueen (Active Directory) ja liitin siihen Windows 11
-työaseman. Sen jälkeen määritin DNS:n, DHCP:n, tiedostojaot, kotikansiot, liikkuvat profiilit,
levykiintiöt, Group Policyn, IIS:n ja tulostuspalvelut. Käyttäjien luonti on automatisoitu
PowerShellillä, ja Windows 11 -asennus tehdään automaattisesti (unattended).

Tutkinnon osa: Järjestelmätuessa toimiminen (106425, 45 osp).

## Ympäristö

VMware ESXi 6.7 (hallinta 192.168.1.50, vSwitch0). Sisäverkkona on `LAB-Internal` -port group.

| Kone | Käyttöjärjestelmä | Rooli |
| --- | --- | --- |
| DC01 | Windows Server 2025 | Toimialueen ohjauskone, DNS, DHCP, tiedosto-/IIS-/tulostuspalvelin |
| WIN-N3D85T4RMA0 | Windows 11 Pro | Toimialuetyöasema |

## Verkko

Palvelimessa kaksi verkkokorttia: NIC1 ulkoiseen labran verkkoon, NIC2 eristettyyn
`LAB-Internal` -toimialueverkkoon. Sisäisellä rajapinnalla ei ole yhdyskäytävää, joten
toimialueverkko pysyy eristettynä (työasemalla ei ole internetiä sen kautta - tarkoituksella).

| Laite | Osoite |
| --- | --- |
| DC01 (sisäinen) | 192.168.100.1/24, DNS 192.168.100.1 |
| Windows 11 | 192.168.100.0/24 DHCP:llä |
| Toimialue | lab.local |

Yhteys testattiin molempiin suuntiin `ping`-komennolla.

## Palvelin, AD DS ja DNS

Asensin Windows Server 2025:n, nimesin sen DC01:ksi, määritin kiinteän sisäisen IP:n ja
päivitykset, minkä jälkeen lisäsin AD DS -roolin ja ylensin koneen toimialueen ohjauskoneeksi.

- Toimialue lab.local, NetBIOS LAB, ohjauskone DC01.lab.local
- Ylennys loi SYSVOL:n, NETLOGON:n ja AD-integroidun DNS-vyöhykkeen automaattisesti.
- Windows 11 käyttää DC01:tä DNS-palvelimena.

```cmd
nslookup lab.local
nslookup DC01.lab.local
ping DC01.lab.local
```

Otin tilannevedokset (snapshot) molemmista virtuaalikoneista ennen jatkamista.

## OU:t, käyttäjät ja ryhmät

OU:t käyttäjille, ryhmille, työasemille ja palvelimille, ja siirsin objektit niihin.

- user1 - luotu käsin, käytetty testaukseen
- student1, student2 - luotu PowerShellillä
- Turvaryhmä IT käyttöoikeuksien keskitettyyn hallintaan - varmennettu komennolla
  `whoami /groups` → `LAB\IT`

Create-LabUsers.ps1:

```powershell
Import-Module ActiveDirectory

$Users = @(
    @{ Username="student1"; FirstName="Student"; LastName="One"; Password="P@ssw0rd123!" },
    @{ Username="student2"; FirstName="Student"; LastName="Two"; Password="P@ssw0rd123!" }
)
foreach ($u in $Users) {
    if (-not (Get-ADUser -Filter "SamAccountName -eq '$($u.Username)'" -ErrorAction SilentlyContinue)) {
        $pw = ConvertTo-SecureString $u.Password -AsPlainText -Force
        New-ADUser -Name "$($u.FirstName) $($u.LastName)" -GivenName $u.FirstName -Surname $u.LastName `
            -SamAccountName $u.Username -UserPrincipalName "$($u.Username)@lab.local" `
            -Path "OU=UserAccounts,DC=lab,DC=local" -AccountPassword $pw -Enabled $true
    }
}
if (-not (Get-ADGroup -Filter "Name -eq 'IT'" -ErrorAction SilentlyContinue)) {
    New-ADGroup -Name "IT" -GroupScope Global -GroupCategory Security -Path "OU=Groups,DC=lab,DC=local"
}
Add-ADGroupMember -Identity "IT" -Members "student1","student2"
```

## Windows 11:n liittäminen toimialueeseen

Asetin DNS:ksi 192.168.100.1, liitin WIN-N3D85T4RMA0:n toimialueeseen lab.local, käynnistin
uudelleen ja kirjauduin toimialuetunnuksella.

```cmd
systeminfo | findstr Domain
whoami
whoami /groups
```

## DHCP

Asensin ja valtuutin DHCP-roolin, loin IPv4-scopen verkolle 192.168.100.0/24 ja määritin DNS-
(192.168.100.1) ja toimialue- (lab.local) optiot. Windows 11 sai osoitteen automaattisesti
(`ipconfig`). DC01 pysyy kiinteästi osoitteessa 192.168.100.1.

## Tiedostopalvelut (jaot, kotikansiot, login-skripti)

SMB-tiedostopalvelin, jossa IT-ryhmän jako ja Home-jako kotikansioille. Jako- ja
NTFS-oikeudet kohdistetaan IT-ryhmän kautta. (Alkuvaiheen "tarvitaan käyttöoikeus" -virhe
johtui jakoasetuksista - korjattu.)

- Kotikansio `\\DC01\Home\user1`, yhdistetty levyksi H:, käytettävissä automaattisesti
  kirjautumisen jälkeen.
- SYSVOL:ssa oleva login-skripti yhdistää verkkolevyt kirjautuessa:

```cmd
net use H: \\DC01\Home\%USERNAME%
```

## Liikkuvat profiilit

Liikkuva profiili tallennettuna sijaintiin `\\DC01\Profiles\%USERNAME%`. Kirjautuessa
palvelimelle muodostui profiilikansio (user1.v6), joten profiili seuraa käyttäjää koneiden
välillä. Varmennettu sisään- ja uloskirjautumisella.

## Levykiintiöt

Otin käyttöön levykiintiöiden hallinnan kotikansioiden levyllä (levyn Ominaisuudet → Kiintiö):
päälle kiintiöiden hallinta, asetettu raja ja varoitustaso, sekä "estä levytila käyttäjiltä,
jotka ylittävät kiintiönsä". Tarkistin käytön Kiintiömerkinnöistä (Quota Entries) ja testasin
käyttäjän H:-levyllä - käyttö seurataan ja rajataan palvelimella.

## Group Policy

GPO:t luotu ja linkitetty OU-rakenteeseen. Sovellettu ja varmennettu komennoilla
`gpupdate /force` + `gpresult /r`:

- Password Policy (salasanapolitiikka)
- Käyttäjä-/työasemarajoitukset - estetty Ohjauspaneeli, komentokehote (CMD) ja Asetukset
  (varmennettu: Asetukset ei avautunut työasemalla)
- Työpöytäasetukset
- Taustakuva - määritetty, mutta kuvan näyttö vaati lisäselvitystä (polku/muoto); osittain

## IIS-web-palvelin

Asensin IIS:n, otin Default Web Siten käyttöön ja lisäsin `index.html`-testisivun. Latautui
onnistuneesti Windows 11:ltä osoitteesta http://192.168.100.1.

## Tulostuspalvelut

Asensin Print and Document Services -roolin, lisäsin jaetun tulostimen, julkaisin sen
toimialueeseen ja yhdistin sen Windows 11 -työasemalta.

## Windows 11:n automaattinen asennus

Tein `autounattend.xml`-vastaustiedoston (Windows SIM / ADK) ja loin siitä ISO-levyn:

```cmd
oscdimg -m -o -u2 -udfver102 C:\Autounattend C:\autounattend.iso
```

ESXi:ssä liitin virtuaalikoneeseen kaksi CD/DVD-asemaa - toisessa Windows 11 -ISO, toisessa
`autounattend.iso`. Windows Setup tunnisti vastaustiedoston automaattisesti ja asennus eteni
itsestään.

## Testaus

- Verkko - ping molempiin suuntiin, DNS toimii, DHCP jakaa osoitteen
- AD - toimialuekirjautuminen toimii, `LAB\IT` -jäsenyys varmennettu, toimialueliitos varmennettu
- Tiedostot - jaot noudattavat oikeuksia, H: ja liikkuva profiili toimivat
- Roolit - IIS avautuu työasemalta, GPO-rajoitukset voimassa, jaettu tulostin toimii
