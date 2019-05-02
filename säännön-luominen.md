# Uuden säännön luominen

Säännöt muuttavat chatin toimintaa validoimalla chat-viestejä vaikuttamalla palvelimen ja asiakkaan väliseen viestintään.

Säännöt ovat luokkia, jotka toteuttavat `Rule`-rajapinnan. Yleensä säännön on hyvä periä `RuleBase`-luokka. Tärkeimmät metodit, joita säännöllä voi olla, ovat:

1. `applyTextMessage`: Muuttaa palvelimen vastaanottamaa TextMessage-viestiä ennen sen välittämistä takaisin asiakasohjelmille.
2. `applyRoomStateMessage`: Muuttaa palvelimen luomaa RoomStateMessage-viestiä ennen sen lähettämistä asiakasohjelmille.
3. `isValidMessage`: Palauttaa boolean-arvon, joka kertoo noudattaako annettu TextMessage-olio sääntöä.

Näiden metodien lisäksi säännöllä voi olla `ruleEnabled` ja `ruleDisabled`-metodit, joita kutsutaan aina, kun sääntö tulee voimaan ja kun se kumotaan.

Luokasta luodaan vain yksi instanssi, joten sääntöolion tila on koko palvelimen laajuinen, **ei** huonekohtainen. Jos huonekohtaista tilaa tarvitaan, voi sääntöolio sisältää esimerkiksi hajautustaulun, jossa on oma tilaa jokaista huoneen tunnistetta kohden.

Alla on listattu vaiheet, jotka on tehtävä, kun uutta sääntöä lisätään. Esimerkkinä luomme säännön, joka sallii viestissä olevan vain tietyn määrä huutomerkkejä.

## 1. Sääntöluokan kirjoittaminen

`src/rules/em-limit-rule.ts`:

```typescript
import {Rule, RuleBase} from './rule';
import {TextMessage, RuleParameters} from 'fluxxchat-protokolla';
import {Connection} from '../connection';

export class ExclamationMarkLimitRule extends RuleBase implements Rule {
	public title = 'rule.emLimit.title';
	public description = 'rule.emLimit.description';
	public ruleName = 'em_limit';
	public parameterTypes = {limit: 'number'} as RuleParameterTypes;

	public isValidMessage(parameters: RuleParameters, message: TextMessage, _sender: Connection) {
		return (message.textContent.match(/\!/g) || []).length <= parameters.limit;
	}
}
```

Sääntöluokassa määritellään metodien lisäksi myös **säännön nimi**, **säännön kuvaus**, **säännön tunniste** ja **säännön parametrit ja niiden tyypit**.

> **Huom!** Jos sääntöluokka yliajaa `ruleEnabled`-metodin, **muista kutsua `super.ruleEnabled()`-metodia**.
>
> RuleBase-luokan ruleEnabled-metodi kumoaa voimassa olevat saman tyyppiset säännöt. Katso esimerkit:
>
> 1. `statistics-rule.ts`: Alustaa huonene tilan ruleEnabled-metodissa ja kutsuu asianmukaisesti yläluokan metodia.
> 2. `message-length-rule.ts`: Kutsuu yläluokan metodia ja kumoaa sen jälkeen lisäksi myös muita sääntöjä.
> 3. `mute-rule.ts`: Ei kutsu yläluokan metodia vaan toteuttaa kumoamisen kokonaan itse.

## 2. Lisää sääntö `active-rules.ts`-tiedostoon

Lisää sääntö `src/rules/active-rules.ts`-tiedostoon:

```typescript
// ...
import {ExclamationMarkLimitRule} from './em-limit-rule';

// ...
const EM_LIMIT = new ExclamationMarkLimitRule();
// ...

export const RULES: {[key: string]: Rule} = {
	// ...
	em_limit: EM_LIMIT,
	no_em_limit: new DisablingRule([EM_LIMIT], 'no_em_limit', 'rule.noEmLimit.title'),
	// ...
};
```

Samalla määritellään säännölle kumoamissääntö. Kumoamissäännölle voi antaa nimen lisäksi myös kuvauksen tarvittaessa.

## 3. Lisää käännösmerkkijonot

Lisää `src/i18n/data.json`-tiedostoon käännösmerkkijonot kaikilla kielillä:

```json
{
	"en": {
		...
		"rule.emLimit.title": "Exclamation Mark Limit",
		"rule.emLimit.description": "Sets the maximum number of allowed exclamation marks",
		"rule.noEmLimit.title": "Disable Exclamation Mark Limit",
		...
	},
	"fi": {
		...
		"rule.emLimit.title": "Huutomerkkirajoitus",
		"rule.emLimit.description": "Asettaa rajan maksimimäärälle huutomerkkejä viestissä",
		"rule.noEmLimit.title": "Kumoa huutomerkkirajoitus",
		...
	}
}
```