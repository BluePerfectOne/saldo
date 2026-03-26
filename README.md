# Saldo

> Henkilökohtainen ja perhekohtainen talouden seurantasovellus — suomenkielinen, selainpohjainen, itse hostattava.

![Versio](https://img.shields.io/badge/versio-0.1.0-blue) ![Lisenssi](https://img.shields.io/badge/lisenssi-MIT-green)

---

## Miksi tämä projekti?

Olemassa olevat sovellukset (mm. Firefly III) ovat hyviä kirjanpitotyökaluja, mutta niissä on puutteita:

- Ylätason tilannekuva puuttuu tai on kömpelö
- Maksuaikakortin laskutusjakson seuranta ei onnistu
- Ylläpito rasittaa (Docker, päivitykset, monimutkaisuus)
- Suomenkielinen käyttöliittymä puuttuu
- Automaattinen pankkiintegraatio suomalaisille pankeille on vaikea toteuttaa kolmannen osapuolen sovelluksissa

Tämä projekti rakentaa kevyen, itse hostattavan dashboardin joka ratkaisee juuri nämä ongelmat.

---

## Tehdyt päätökset

### Teknologia

| Komponentti | Valinta | Perustelu |
|---|---|---|
| Frontend | HTML + CSS + JS (vanilla) | Ei build-tooleja, suoraan selaimessa, helppo ylläpitää |
| Kuvaajat | Chart.js | Kevyt, selainpohjainen, ei riippuvuuksia |
| Backend | Python (Flask tai FastAPI) | Kevyt, tuttu, hyvät kirjastot |
| Tietokanta | SQLite | Ei erillistä palvelinta, riittää kotikäyttöön |
| Pankkiintegraatio | GoCardless Bank Account Data API | Ainoa realistinen vaihtoehto yksityishenkilölle PSD2-ympäristössä; tukee OP:ta, S-Pankkia, Nordeaa ym. |
| Hosting | Paikallinen (localhost tai kotiverkko) | Ei pilvipalvelua, data pysyy omassa hallinnassa |

### UI-arkkitehtuuri

- **Dashboard-ensin**: ylätason tilannekuva on oletussivu — ei kirjanpitosovellus
- **Slide-in sivupaneeli**: kortteja klikkaamalla aukeaa detaljinäkymä oikealta ilman sivunvaihtoa
- **Kieli**: suomi oletuksena, kielitiedostot eriytetään lokalisointia varten

### Integraatiot ja tietolähteet

| Lähde | Metodi | Automaatio |
|---|---|---|
| S-Pankki, OP, Nordea | GoCardless API | Automaattinen, OAuth uusittava 90 pv välein |
| BASE | GoCardless API | Automaattinen |
| Nordnet | CSV-export | Manuaalinen tai puoliautomaattinen |
| PayPal | PayPal REST API tai CSV | TBD |
| Mastercard (maksuaikakortti) | GoCardless API | Automaattinen |
| Lainat | Manuaalinen syöttö | Manuaalinen (saldo + korko + lyhennys) |

---

## Arkkitehtuuri

```
┌─────────────────────────────────────────────┐
│              Selain (Frontend)               │
│   dashboard.html + chart.js + panel UI      │
└────────────────────┬────────────────────────┘
                     │ REST API
┌────────────────────▼────────────────────────┐
│             Python Backend                   │
│   Flask/FastAPI · Auth · Scheduler          │
└──────┬───────────────────────┬──────────────┘
       │                       │
┌──────▼──────┐        ┌───────▼──────────────┐
│   SQLite    │        │  GoCardless API       │
│  (data.db)  │        │  + Nordnet CSV        │
└─────────────┘        └──────────────────────┘
```

### Tietokantarakenne (alustava)

```
users           — käyttäjät (id, nimi, salasana_hash, rooli)
accounts        — tilit (id, user_id, pankki, nimi, tyyppi, gocard_id)
transactions    — tapahtumat (id, account_id, pvm, summa, kuvaus, kategoria_id)
categories      — kategoriat (id, nimi, väri, säännöt_json)
loans           — lainat (id, user_id, nimi, jäljellä, lyhennys, korko, eräpäivä)
card_cycles     — maksuaikakorttijaksot (id, account_id, alku, loppu, maksupäivä, budjetti_pv)
```

---

## Maksuaikakortti-ominaisuus

Tämä on ominaisuus jota muut sovellukset eivät tue:

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
- Teknisesti yksinkertainen: yksi SQLite + käyttäjätaulu + user_id viiteavaimen kaikkialla

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
- [ ] Python-projektirakenne, Flask/FastAPI
- [ ] SQLite-skeema ja migraatiot
- [ ] REST API datalle (tilit, tapahtumat, saldot)
- [ ] Dashboard hakee datan API:sta JSONin kautta

### Vaihe 3 — GoCardless-integraatio
- [ ] GoCardless developer-tili ja API-avaimet
- [ ] OAuth-flow pankin yhdistämiseen (S-Pankki, OP, Nordea, BASE)
- [ ] Tapahtumien haku ja tallennus SQLiteen
- [ ] Scheduler (cron) päivittäistä hakua varten

### Vaihe 4 — Autentikointi
- [ ] Käyttäjätilit ja kirjautuminen
- [ ] Session-hallinta
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
rahatilanne/
├── frontend/
│   ├── index.html          ← dashboard (tästä aloitettu)
│   ├── static/
│   │   ├── style.css
│   │   └── app.js
│   └── components/
│       └── panel.js
├── backend/
│   ├── app.py              ← Flask/FastAPI entry point
│   ├── models.py           ← tietokantamallit
│   ├── routes/
│   │   ├── accounts.py
│   │   ├── transactions.py
│   │   └── auth.py
│   └── integrations/
│       ├── gocardless.py
│       ├── nordnet.py
│       └── paypal.py
├── data/
│   └── rahatilanne.db      ← SQLite (gitignoressa)
├── config.example.yaml     ← konfiguraatiopohja
├── requirements.txt
└── README.md
```

---

## Tietoturva ja yksityisyys

- Sovellus on tarkoitettu **paikalliseen käyttöön** (localhost tai kotiverkko)
- Pankkidata tallennetaan ainoastaan omalle koneelle
- GoCardless-API-avaimet tallennetaan ympäristömuuttujiin, ei koodiin
- Julkinen GitHub-repository ei sisällä henkilökohtaista dataa (`.gitignore`)
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
