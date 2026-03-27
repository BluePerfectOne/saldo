# Saldo

> Henkilökohtainen ja perhekohtainen talouden seurantasovellus — suomenkielinen, selainpohjainen, itse hostattava.

![Versio](https://img.shields.io/badge/versio-0.1.0-blue) ![Lisenssi](https://img.shields.io/badge/lisenssi-MIT-green)

---

## Miksi tämä projekti?

Olemassa olevat sovellukset kuten Firefly III ovat erinomaisia omiin tarkoituksiinsa rakennettuja kirjanpitotyökaluja — tämä projekti ei pyri kilpailemaan niiden kanssa. Lähtökohta on puhtaasti henkilökohtainen: mikä työkalu sopii juuri minulle ja kotitalouteeni.

Projektin prioriteetit ovat:

- **Ylätason tilannekuva ensin** — missä mennään, ei kirjanpitolomakkeita
- **Suomalaiset pankit** — GoCardless PSD2-integraation kautta, automaattisesti
- **Maksuaikakortin laskutusjaksoseuranta** — ominaisuus jota en ole muualta löytänyt
- **Perhenäkymä** — kotitalouden yhteinen tilannekuva yhden asennuksen sisällä
- **Täysin itse hostattava** — data pysyy omassa hallinnassa, ei pilviriippuvuuksia
- **Suomenkielinen käyttöliittymä** — koska se on luontevinta

Tämä on henkilökohtainen projekti. Se ei pyri olemaan yleiskäyttöinen — se pyrkii olemaan oikea työkalu omaan käyttöön.

---

## Tehdyt päätökset

### Teknologia

| Komponentti | Valinta | Perustelu |
| --- | --- |--- |
| Frontend | HTML + CSS + JS (vanilla) | Ei build-tooleja, suoraan selaimessa, helppo ylläpitää |
| Kuvaajat | Chart.js | Kevyt, selainpohjainen, ei riippuvuuksia |
| Backend | Python + FastAPI | Async-natiivi, Pydantic-validointi, automaattiset API-dokumentit — sopii GoCardless-scheduleriin |
| Tietokanta | SQLite | Ei erillistä palvelinta, riittää kotikäyttöön |
| Pankki-integraatio | GoCardless Bank Account Data API | Ainoa realistinen vaihtoehto yksityishenkilölle PSD2-ympäristössä; tukee OP:ta, S-Pankkia, Nordeaa ym. |
| Hosting | Paikallinen (localhost tai kotiverkko) | Ei pilvipalvelua, data pysyy omassa hallinnassa |

### UI-arkkitehtuuri

- **Dashboard-ensin**: ylätason tilannekuva on oletussivu — ei kirjanpitosovellus
- **Slide-in sivupaneeli**: kortteja klikkaamalla aukeaa detaljinäkymä oikealta ilman sivunvaihtoa
- **Kieli**: suomi oletuksena, kielitiedostot eriytetään lokalisointia varten

### Integraatiot ja tietolähteet

| Lähde | Integraatiomoduuli | Automaatio |
| --- | --- | --- |
| S-Pankki, OP, Nordea | `gocardless` | Automaattinen, OAuth uusittava 90 pv välein |
| BASE | `gocardless` | Automaattinen |
| Nordnet | `nordnet_csv` | Manuaalinen tai puoliautomaattinen |
| Mastercard (maksuaikakortti) | `gocardless` | Automaattinen |
| Lainat | `manual` | Manuaalinen syöttö (saldo + korko + lyhennys) |
| PayPal | — | Ei toteuteta toistaiseksi — katso Avoimet kysymykset |

### Integrointiarkkitehtuuri

Integroinnit ovat omia moduulejaan jotka toteuttavat yhteisen `DataSourceIntegration`-protokollan. Saldo-ydin ei tiedä mitään yksittäisten integraatioiden toteutuksesta — se kutsuu vain sovittua rajapintaa.

**Protokolla** (`backend/integrations/base.py`):

```python
class DataSourceIntegration(Protocol):
    def get_config_schema(self) -> dict: ...           # lomakekenttien määritys
    def list_credentials(self) -> list[dict]: ...      # tallennettavien tunnistetietojen kuvaukset
    def get_alert_schedule(self) -> list[dict]: ...    # millä aikataululla check_alerts() ajetaan
    async def check_alerts(self) -> list[Alert]: ...   # palauttaa avoinna olevat hälytykset
    async def test_connection(self) -> bool: ...       # "Testaa yhteys" -nappi
    async def fetch_accounts(self) -> list: ...        # tilit Saldon formaatissa
    async def fetch_transactions(self, account_id, since) -> list: ...
```

`get_alert_schedule()` palauttaa listan tarkistusaikatauluja (esim. `[{"interval_hours": 24}]`). Saldo rekisteröi nämä scheduleriin yhteystä alustettaessa — scheduler itse ei tiedä mitä tarkistaa, se vain kutsuu `check_alerts()` sovituin väliajoin. `Alert`-rakenne sisältää otsikon, kuvauksen, vakavuustason ja mahdollisen toimintakehotteen.

`list_credentials()` palauttaa listan sanakirjoja muodossa `{"nimi": ..., "tarkoitus": ..., "sijainti": ..., "arvo": ...}`. Backend käy läpi kaikki aktiiviset tietolähteet ja aggregoi vastaukset yhdelle päätepisteelle. Arvot puretaan salauksesta vasta käyttäjän eksplisiittisellä pyynnöllä.

Jokainen integraatio (`gocardless.py`, `nordnet_csv.py`, `manual.py`) toteuttaa protokollan omalla tavallaan. Rekisteri (`backend/integrations/registry.py`) yhdistää tunnisteen (`"gocardless"`) toteutusluokkaan.

**Tietolähteen lisäystyönkulku:**
1. Käyttäjä avaa "Lisää tietolähde" — näkee listan rekisteröidyistä integraatioista
2. Valitsee integraation — backend palauttaa sen `config_schema` — frontend renderöi lomakkeen
3. Käyttäjä täyttää kentät, painaa "Testaa yhteys" — backend kutsuu `test_connection()`
4. Onnistumisen jälkeen käyttäjä vahvistaa — tunnistetiedot tallennetaan salattuina tietokantaan

Uuden integraation lisääminen = yksi uusi tiedosto `integrations/`-kansioon + yksi rivi rekisteriin.

---

## Hälytykset ja ilmoitukset

Integrointimoduulit voivat lähettää Saldon käyttöliittymään tärkeitä ilmoituksia ilman, että Saldo-ydin tietää mitään niiden sisällöstä. Vastuu vuokra-aikojen seurannasta, OAuth-vanhenemisista ja muista integraatiokohtaisista tiloista on aina integraatiomoduulilla itsellään.

### Vastuunjako

| Osapuoli | Vastuu |
| --- | --- |
| **Integraatiomoduuli** | Tietää oman vuokra-aikansa, valtuutuksensa tilan ja missä tilanteessa käyttäjän toimenpiteet ovat tarpeen |
| **Integraatiomoduuli** | Määrittelee kuinka usein `check_alerts()` kutsutaan (`get_alert_schedule()`) |
| **Integraatiomoduuli** | Palauttaa `Alert`-objektilistan kun tarkistus ajetaan |
| **Scheduler** | Kutsuu `check_alerts()` integraation määrittämällä aikataululla — ei tiedä mikä tarkistus on |
| **Saldo-ydin** | Tallentaa hälytykset tietokantaan ja toimittaa ne käyttöliittymälle |
| **Frontend** | Näyttää hälytykset (ilmoitusalue, piste kuvakkeessa tms.) — toteutustapa auki |

### `Alert`-rakenne

```python
@dataclass
class Alert:
    integration_type: str   # "gocardless"
    account_id: int | None  # liittyykö tiettyyn tiliin vai koko integraatioon
    severity: str           # "info" | "warning" | "error"
    title: str              # "GoCardless-valtuutus vanhenee pian"
    body: str               # "Yhteys S-Pankkiin vanhenee 5 päivän kuluttua. Uusi valtuutus tarvitaan."
    action_url: str | None  # linkki toimenpideohjeeseen tai re-auth-virtaukseen
    expires_at: datetime | None  # milloin hälytys ei enää ole ajankohtainen
```

### Esimerkki: GoCardless OAuth-vanheneminen

`gocardless.py` tallentaa yhteyden luomisajan ja tietokannasta luettavan vanhenemisajan. `get_alert_schedule()` palauttaa `[{"interval_hours": 24}]`. Kun scheduler kutsuu `check_alerts()`, moduuli laskee jäljellä olevan ajan:
- > 14 pv: ei hälytystä
- 7–14 pv: `severity: "warning"` — ilmoitus näkyviin
- < 7 pv: `severity: "error"` — korostettu ilmoitus
- Vanhentunut: `severity: "error"` — datasource merkitty epäkurantiksi

Yhteys uusitaan käyttäjän toimesta ilmoituksen kautta — ei automaattisesti.

---

## Avoimet kysymykset ja alustavat päätökset

Seuraavat asiat on tunnistettu mutta ei vielä päätetty — tai niihin on tehty alustava päätös, joka saattaa muuttua arkkitehtuurin tarkentuessa. Nämä kannattaa ratkaista ennen kyseistä vaihetta.

### ~~¹ Flask vai FastAPI~~ ✅ Päätetty: FastAPI

**Tila:** Päätetty.

FastAPI valittu seuraavilla perusteilla:
- Async-natiivi — GoCardless-scheduler ja ulkoiset API-kutsut toimivat luontevasti
- Pydantic-skeemat toimivat samalla tietokontrakteina frontendin ja backendin välillä
- Automaattinen Swagger-dokumentaatio (`/docs`) helpottaa kehitystä
- Dependency injection tekee autentikoinnista siistimpää kuin Flaskissa

**Huomio SQLite:stä:** Standardikirjaston `sqlite3` on synkroninen. FastAPI-async-reitittimissä DB-kutsut ajetaan joko `aiosqlite`-kirjastolla tai `asyncio.to_thread()`:llä. Tämä päätös tehdään Vaiheen 2 yhteydessä.

### ² PayPal-integraatio

**Tila:** Lykätty — ei toteuteta toistaiseksi.

Integrointiarkkitehtuurin ansiosta PayPal-integraatio on helppo lisätä myöhemmin omana moduulinaan ilman muutoksia ytimeen. Ei estä muuta kehitystä.

### ~~GoCardless OAuth:n uusiminen käytännössä~~ ✅ Päätetty: integraatiomoduuli omistaa hälytyslogiikan

**Tila:** Päätetty.

Vuokra-aikojen seuranta ja vanhenemisilmoitukset ovat integraatiomoduulin vastuulla, eivät schedulerin. Scheduler on sokea suorittaja — se kutsuu `check_alerts()` integraation itsensä määrittämällä aikataululla. Katso [Hälytykset ja ilmoitukset](#h%C3%A4lytykset-ja-ilmoitukset) -luku.

### Maksuaikakortin budjetti-granulaarisuus

**Tila:** Alustava — `budjetti_pv` on tällä hetkellä yksittäinen arvo per laskutusjakso.

Avoimia alakysymyksiä ennen tietokannan lukitsemista:
- Onko budjetti korttikohtainen vai käyttäjäkohtainen?
- Säilytetäänkö historiallinen budjetti eri jaksoille erikseen?
- Voidaanko jaksokohtaista budjettia muuttaa jälkikäteen?

### Projektin nimi ja kansiorakenne

**Tila:** Epäjohdonmukaisuus — hakemistorakenne käyttää nimeä `rahatilanne/`, repositorio on `Saldo`.

Ennen koodipohjan skaffoldausta kannattaa päättää lopullinen nimi ja kansiorakenne.

### Versiokartan aukot

**Tila:** Alustava — versiokartta hyppää `0.6.0`→`0.9.0` ilman väliversioita.

Vaiheet 7+ (lokalisaatio, mobiili, dokumentaatio) ovat vielä nimeämättä. Kannattaa joko nimetä väliversiot tai hyväksyä, että ne täsmentyvät myöhemmin.

---

## Tunnistetietojen hallintanäkymä

Kaikki sovelluksen tallentamat arkaluontoiset tiedot ovat nähtävillä yhdessä paikassa. Tarkoitus on antaa käyttäjälle täysi läpinäkyvyys siitä, mitä järjestelmä tietää ja mihin se on tallennettu.

### Mitä näkymä näyttää

Jokaisesta tallennetusta tunnistetiedosta esitetään:

| Kenttä | Kuvaus |
|---|---|
| **Nimi** | Tunnisteen nimi (esim. "GoCardless access token") |
| **Tarkoitus** | Mitä varten tietoa käytetään (esim. "Pankkiyhteyden OAuth-valtuutus, S-Pankki") |
| **Sijainti** | Missä tieto säilytetään (esim. `accounts.config_json`, `.env`) |
| **Arvo** | Salattu oletuksena — käyttäjä paljastaa yksitellen klikkaamalla |

Jokainen integraatiomoduuli toteuttaa `list_credentials()` joka palauttaa tämän rakenteen. Backend aggregoi kaikkien aktiivisten tietolähteiden tiedot yhdelle endpointille.

### Suojaukset

- Näkymä vaatii **uudelleentodentamisen** (salasana syötetään uudelleen) vaikka sessio olisi aktiivinen
- Arvot näytetään **piilotettuna** oletuksena, paljastetaan yksitellen eksplisiittisellä klikkauksella
- Näkymä on saatavilla **vain adminille**

### Pyyhi kaikki -toiminto

"Pyyhi kaikki tunnistetiedot" -nappi:
1. Pyytää erillisen vahvistuksen (ei pelkkä "Oletko varma?" — vaatii esim. salasanan tai tekstin kirjoittamisen)
2. Poistaa kaikkien tilien `config_json`-arvojen sisällön
3. Peruuttaa voimassa olevat OAuth-istunnot ulkoisiin palveluihin (integraatiokohtainen `revoke()`-metodi, jos saatavilla)
4. Tyühjää käyttäjän sessiotunnukset
5. Kirjaa toimenpiteen lokiin

Tämä on sovelluksen "tehdasnollaus" tunnistetietojen osalta. Tallennettu finanssidata (tapahtumat, saldot) *ei* poistu — sen poistaminen on erillinen toimi.

---

## Arkkitehtuuri

```
┌─────────────────────────────────────────────┐
│              Selain (Frontend)               │
│   dashboard.html + chart.js + panel UI      │
│   + tietolähteen lisäysvelho                │
└────────────────────┬────────────────────────┘
                     │ REST API
┌────────────────────▼────────────────────────┐
│             Python + FastAPI                 │
│   Auth · Scheduler · Integration Registry   │
└──────┬───────────────────────┬──────────────┘
       │                       │
┌──────▼──────┐        ┌───────▼──────────────────────────┐
│   SQLite    │        │  Integration Registry             │
│  (data.db)  │        │  gocardless · nordnet_csv · ...  │
└─────────────┘        └──────────────────────────────────┘
```

### Tietokantarakenne (alustava)

```
users           — käyttäjät (id, nimi, salasana_hash, rooli)
accounts        — tilit (id, user_id, nimi, tyyppi, integration_type, config_json)
transactions    — tapahtumat (id, account_id, pvm, summa, kuvaus, kategoria_id)
categories      — kategoriat (id, nimi, väri, säännöt_json)
loans           — lainat (id, user_id, nimi, jäljellä, lyhennys, korko, eräpäivä)
card_cycles     — maksuaikakorttijaksot (id, account_id, alku, loppu, maksupäivä, budjetti_pv)
```

`integration_type` on rekisteriavain (esim. `"gocardless"`). `config_json` on salattu JSON-blob joka sisältää integraatiokohtaiset tunnistetiedot — ei GoCardless-spesifisiä sarakkeita ytimessä.

---

## Maksuaikakortti-ominaisuus

- Korttijaksolla on **alkupäivä, loppupäivä ja maksupäivä** (esim. 1.–31.3., maksu 1.4.)
- Dashboard näyttää **kertymän** ja **edistymispalkin** jakson sisällä
- **Päivittäinen vauhti** lasketaan kertymästä ja jäljellä olevista päivistä
- **Budjetti per päivä** on käyttäjän asettama; ylitys näkyy varoituksena
- **Ennuste jakson lopulle** perustuu nykyiseen vauhtiin

---

## Monikäyttäjä / perheominaisuus

Sovellus tukee useampaa käyttäjää yhden asennuksen sisällä:

- Jokainen kirjautuu omilla tunnuksillaan
- Tilit ja tapahtumat liitetty käyttäjäkohtaisesti
- **Perhenäkymä** (valinnainen): aggregoitu dashboard joka yhdistää kaikkien tilit
  - Voidaan rajoittaa tietyille käyttäjille (esim. vain admin näkee kaikkien tiedot)
- Teknisesti yksinkertainen: yksi SQLite + käyttäjätaulu + user_id viiteavain kaikkialla

> Esimerkki: palkansaajat näkevät omat tilinsä, perhenäkymä näyttää yhteisen tilanteen.

---

## Kategorisointi

Kolmitasoinen malli:

1. **Automaattinen sääntöpohjainen**: avainsanat tapahtuman kuvauksessa → kategoria
   - Esim. "K-Market" → Ruoka, "Telia" → Laskut
2. **Puoliautomaattinen**: tunnistamattomat tapahtumat flagataan → käyttäjä hyväksyy kategorian
3. **Manuaalinen**: käyttäjä voi aina ylikirjoittaa

Säännöt tallennetaan JSON-rakenteena tietokantaan, helposti laajennettavissa.

---

## Jatkosuunnitelma

### Vaihe 1 — Staattinen prototyyppi ✅ (tehty)
- Dashboard HTML mockup esimerkkidatalla
- Kaikki kortit klikattavissa, sivupaneeli toimii
- Maksuaikakortti-widget

### Vaihe 2 — Backend ja tietokanta
- [ ] Python-projektirakenne, FastAPI
- [ ] `DataSourceIntegration`-protokollan määrittely ja `gocardless`-moduulin runko
- [ ] Integration Registry
- [ ] SQLite-skeema ja migraatiot (`aiosqlite` vs `asyncio.to_thread()` päätetään tässä)
- [ ] REST API datalle (tilit, tapahtumat, saldot)
- [ ] Tietolähteen lisäysvelho: lomake, "Testaa yhteys", tallennus
- [ ] Dashboard hakee datan API:sta JSONin kautta

### Vaihe 3 — GoCardless-integraatio
- [ ] GoCardless developer-tili ja API-avaimet
- [ ] OAuth-flow pankin yhdistämiseen (S-Pankki, OP, Nordea, BASE)
- [ ] Tapahtumien haku ja tallennus SQLiteen
- [ ] Scheduler: rekisteröi `get_alert_schedule()` kaikkien aktiivisten integraatioiden osalta
- [ ] Hälytysten tallennus tietokantaan ja toimitus frontendille
- [ ] `gocardless.py`: `check_alerts()` toteuttaa vanhenemislogiikan (14/7 pv -kynnykset)
- [ ] Re-auth-virtaus ilmoituksesta käsin

### Vaihe 4 — Autentikointi ja hallinta
- [ ] Käyttäjätilit ja kirjautuminen
- [ ] Session-hallinta
- [ ] Uudelleentodentamisvirtaus (tunnistetietojen suojaus)
- [ ] Tunnistetietojen hallintanäkymä: listaus, peitto/paljastus, pyyhi kaikki
- [ ] Perhenäkymä

### Vaihe 5 — Kategorisointi
- [ ] Sääntöpohjainen automaattiluokittelu
- [ ] Manuaalinen tarkistus tuntemattomille
- [ ] Kategoriaeditori UI:ssa

### Vaihe 6 — Lisäintegraatiot
- [ ] Nordnet CSV-tuonti
- [ ] PayPal-integraatio
- [ ] Laina-syöttölomake

### Vaihe 7 — Viimeistely
- [ ] Lokalisaatio (i18n-rakenne, FI/EN)
- [ ] Mobiiliystävällinen layout
- [ ] Asennus-/konfiguraatio-ohje

---

## Versiointi

Projekti noudattaa [semanttista versiointia](https://semver.org/lang/fi/) (`MAJOR.MINOR.PATCH`).

| Versio | Tila | Sisältö |
|---|---|---|
| `0.1.0` | ✅ Valmis | Staattinen dashboard-HTML, mockup-data, sivupaneeli, maksuaikakortti-widget |
| `0.2.0` | 🔲 Seuraava | Python backend + SQLite + REST API |
| `0.3.0` | 🔲 | GoCardless-integraatio, oikea pankkidata |
| `0.4.0` | 🔲 | Autentikointi, monikäyttäjä/perhenäkymä |
| `0.5.0` | 🔲 | Kategorisointi (sääntöpohjainen + manuaalinen tarkistus) |
| `0.6.0` | 🔲 | Nordnet CSV-tuonti, PayPal, lainasyöttö |
| `0.9.0` | 🔲 | Feature freeze — vain bugifiksit |
| `1.0.0` | 🔲 | Vakaa julkaisu, täysi dokumentaatio |

Patch-versiot (`0.1.1` jne.) bugifixeille ja pienille korjauksille. Muutokset kirjataan [CHANGELOG.md](CHANGELOG.md)-tiedostoon.

---

## Tiedostorakenne (tavoite)

```
saldo/
├── frontend/
│   ├── index.html              ← dashboard (tästä aloitettu)
│   ├── credentials.html        ← tunnistetietojen hallintanäkymä
│   ├── static/
│   │   ├── style.css
│   │   └── app.js
│   └── components/
│       └── panel.js
├── backend/
│   ├── app.py                  ← FastAPI entry point
│   ├── models.py               ← tietokantamallit
│   ├── routes/
│   │   ├── accounts.py
│   │   ├── transactions.py
│   │   ├── auth.py
│   │   └── credentials.py          ← tunnistetietojen aggregointi ja pyyhintä
│   └── integrations/
│       ├── base.py                 ← DataSourceIntegration Protocol
│       ├── registry.py             ← integraatiorekisteri
│       ├── gocardless.py
│       ├── nordnet_csv.py
│       └── manual.py
├── data/
│   └── saldo.db                ← SQLite (gitignoressa)
├── config.example.yaml
├── requirements.txt
└── README.md
```

---

## Tietoturva ja yksityisyys

- Sovellus on tarkoitettu **paikalliseen käyttöön** (localhost tai kotiverkko)
- Pankkidata tallennetaan ainoastaan omalle koneelle
- Integraatiokohtaiset tunnistetiedot tallennetaan **salattuina** `config_json`-kenttään, ei selkokielisesti
- GoCardless-API-avaimet tallennetaan ympäristömuuttujiin, ei koodiin
- Julkinen GitHub-repository ei sisällä henkilökohtaista dataa (`.gitignore`)
- **Tunnistetietojen hallintanäkymä** antaa käyttäjälle täyden läpinäkyvyyden kaikesta tallennetusta arkaluonteisesta tiedosta
- Tuotantokäyttöön kotiverkossa: HTTPS + vahva salasana suositeltava

---

## Kehitysympäristö

```bash
# Kloonaa
git clone https://github.com/[käyttäjä]/rahatilanne.git
cd rahatilanne

# Python-ympäristö
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt

# Käynnistys
python backend/app.py
# → http://localhost:5000
```

---

## Lisenssi

MIT — vapaa käyttää, muokata ja jakaa.
