# FluxxChat-protokolla

## Yleistä

FluxxChat on asiakas-palvelin-malliin pohjautuva pikaviestisovellus. Asiakkaat viestivät palvelimen kanssa WebSocket-yhteyden kautta ja palvelin välittää asiakkaiden viestit toisille asiakkaille.

Asiakkaan voi istunnon aikana joko olla huoneessa tai olla olematta huoneessa, ja voi tästä riippuen lähettää erilaisia viestejä (paketteja). Viestit voidaan jakaa kahteen kategoriaan: niihin, joita voi lähettää aina (*yleiset viestit*), ja niihin, joita voi lähettää vain huoneeseen liittymisen jälkeen (*huoneviestit*). Viestit ovat JSON-muotoisia tekstejä, ja yksi viesti lähetetään aina yhdessä WebSocket-kehyksessä.

Tyypillisesti istunnon alussa tapahtuu seuraavia asioita:

1. Asiakas ja palvelin muodostavat WebSocket-yhteyden.
2. Palvelin lähettää asiakkaalle SERVER_STATE-viestin, joka sisältää mm. palvelimen käännösmerkkijonot palvelimen tukemilla kielillä ja listan palvelimen tukemista korteista.
3. (*valinnainen*) Asiakas lähettää CREATE_ROOM-viestin, johon palvelin vastaa ROOM_CREATED-viestillä, joka sisältää huoneen tunnisteen.
4. Asiakas lähettää JOIN_ROOM-viestin, jonka mukana on huoneen tunniste.
5. Palvelin lähettää asiakkaalle ROOM_STATE-viestin, joka sisältää tiedot huoneessa olijoista ja voimassa olevista säännöistä.

Kun huoneeseen on näin liitytty, asiakas voi lähettää palvelimelle TEXT- ja NEW_RULE-viestejä. Palvelin käsittelee nämä viestit ja lähettää yleensä jokaisen vastaanottamansa viestin jälkeen kaikille asiakkaille uuden ROOM_STATE-viestin, joka sisältää tiedon muutoksista, sekä mahdollisesti TEXT-, SYSTEM- ja VALIDATE_TEXT_RESPONSE-viestejä.

Istunnon päättyessä asiakas lähettää LEAVE_ROOM-viestin, jonka jälkeen yhteys katkaistaan.

Koska tahansa istunnon aikana asiakas voi lähettää PROFILE_IMG_CHANGE-viestejä muuttaakseen profiilikuvaansa ja palvelin KEEP_ALIVE-viestejä, jotka eivät tee mitään muuta, kuin pitävät yhteyden olemassa.

**Tämä dokumentti on yleisen tason kuvaus protokollasta. Katso protokollan [lähdekoodi] ja sen kommentointi saadaksesi yksityiskohtaista tietoa.**

[lähdekoodi]: https://github.com/FluxxChat/FluxxChat-protokolla/blob/master/lib/index.ts

## Yleiset viestit

Näitä viestejä voi lähettää koska tahansa istunnon aikana.

### Asiakkaalta palvelimelle

#### CREATE_ROOM

Pyytää palvelinta luomaan uuden huoneen. Palvelimen odotetaan vastaavan tähän ROOM_CREATED-viestillä.

#### JOIN_ROOM

Ilmoittaa palvelimelle, että asiakas haluaa liittyä huoneeseen. Viestin mukana on huoneen tunniste.

Jos asiakas on jo huoneessa, hänet poistetaan siitä. Asiakkaan ei pidä lähettää erikseen LEAVE_ROOM-viestiä (jota käytetään vain kun yhteys halutaan katkaista).

### Palvelimelta asiakkaalle

#### LANGUAGE_DATA

Palvelin lähettää tämän viestin heti yhteyden muodostamisen jälkeen. Se sisältää palvelimen lähettämien viestien käännökset palvelimen tukemilla kielillä.

#### ROOM_CREATED

Sisältää palvelimen luoman uuden huoneen tunnisteen. Palvelin lähettää tämän viestin vastauksena CREATE_ROOM-viestiin.

#### ERROR

Sisältää virheviestin. Palvelin voi lähettää tämän koska tahansa virheen sattuessa.

#### KEEP_ALIVE

Tämä paketti ei tee mitään. Sen tarkoitus on pitää yhteys voimassa.

## Huoneviestit

### Asiakkaalta palvelimelle ja palvelimelta asiakkaalle

#### TEXT

Asiakkaan lähettämä pikaviesti muille huoneen asiakkaille. Nimestään huolimatta voi sisältää sekä tekstiä, kuvia että ääntä.

Jos `validateOnly`-kentän arvo on `true`, palvelin vastaa tähän viestiin VALIDATE_TEXT_RESPONSE-viestillä, jolla ilmoitetaan, rikkooko viesti sääntöjä. Jos kentän arvo on `false` ja viesti ei riko sääntöjä, palvelin lisää viestiin metatietoja (`markdown`, `senderId`, `senderNickname`, `timestamp`) ja välittää sen tämän jälkeen sellaisenaan kaikille asiakkaille lähettäjä mukaan lukien.

Palvelin säilyttää kaikki viestin kentät, joten uusien sisältötyyppien lisääminen ei vaadi muutoksia palvelimeen.

### Asiakkaalta palvelimelle

#### NEW_RULE

Pyytää palvelinta ottamaan käyttöön uuden säännön.

### Palvelimelta asiakkaalle

#### SYSTEM

Palvelimelta lähtöisin oleva pikaviesti. Sisältää tyypillisesti ilmoituksen uudesta huoneeseen liittyjästä, huoneesta lähtijästä tai uudesta säännöstä.

#### VALIDATE_TEXT_RESPONSE

Vastaus TEXT-viestiin, jonka `validateOnly`-kenttä on `true`. Sisältää listan säännöistä, joita tarkasteltu viesti rikkoo.

#### WORD_PREDICTION

Palvelin lähettää tämän viestin VALIDATE_TEXT_RESPONSE-viestin yhteydessä, jos ennakoiva tekstinsyöttö on otettu säännöllä käyttöön.

## Käännösmerkkijonot

Palvemen lähettämät SYSTEM-, ERROR- ja ROOM_STATE-viestit eivät sisällä luonnollista kieltä, vaan käännöskoodeja. Asiakkaan oletetaan korvaavan koodit LANGUAGE_DATA-viestissä annetuilla käännösmerkkijonoilla ennen niiden näyttämistä käyttäjälle.

Käännösmerkkijonot saattavat sisältää kohtia, joihin on täydennettävä arvoja. Täydennettävät arvot on annettu viestien `values`-kentissä.
