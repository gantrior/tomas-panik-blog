---
title: "Základy ChatGPT API pro vývojáře (v Pythonu)"
date: 2023-10-22
draft: false
tags:
  - AI
  - Python
  - ChatGPT
authors:
  - tomas-panik
---

Tento článek je pro ty vývojáře, kteří si ještě nevyzkoušeli ChatGPT API. Já si to nedávno vyzkoušel a musím konstatovat, že to skýtá velký potenciál. Zde je tedy podrobnější návod, jak na to.

{{< toc >}}

---

Než začneme, tak ...

## Co je to "token" a kolik stojí dotazy na OpenAI API?

V kontextu OpenAI modelů je token jednotkou textu, kterou model zpracovává. Může být znakem, slovem nebo složeným slovem, v závislosti na jazyce a specifickém textu.

Kodování tokenů je proces, kdy je text rozdělen do jednotlivých tokenů, které model může zpracovat. OpenAI používá Byte-Pair-Encoding (BPE) algoritmus pro tokenizaci textu.

Pro práci s tokeny OpenAI budete potřebovat knihovnu `tiktoken`.

```bash
pip install tiktoken
```

Použijte knihovnu `tiktoken` k zakódování textu do tokenů.

```python
import tiktoken

tokenizer = tiktoken.encoding_for_model("gpt-3.5-turbo")
tokens = tokenizer.encode("Text který je třeba tokenizovat: !@#$%^&*()")

print(tokens)

for token in tokens:
  print(tokenizer.decode_single_token_bytes(token))
```
Výstup:
```bash
[1199, 100228, 20195, 4864, 259, 29432, 71853, 4037, 450, 869, 266, 25, 758, 31, 49177, 46999, 5, 9, 368]
b'Text'
b' kter'
b'\xc3\xbd'
b' je'
b' t'
b'\xc5\x99'
b'eba'
b' token'
b'iz'
b'ov'
b'at'
b':'
b' !'
b'@'
b'#$'
b'%^'
b'&'
b'*'
b'()'
```

Z výstupu je vidět, jakým způsobem se text převádí na tokeny. Např. `Text` jako slovo je jeden token. Naopak české slovo `třeba` se rozdělilo na 3 tokeny: `t`, `ř`, `eba`. Z čehož vyplývá, že nejoptimálněji se tokenizuje text bez háčků a čárek. 

### Kolik stojí dotazy na OpenAI API?
Dotazy na OpenAI API nejsou zadarmo, ale noví uživatelé mají počáteční budget, který je dostatečný na prvotní experimentování. Obecně je cena za vstupní/výstupní tokeny.

Model `gpt-3.5-turbo`, který budeme dále používat, je velmi cenově příznivý. Jeho cena je pro vstupní data $0.0015 / 1000 tokenů a výstupní data $0.002 / 1000 tokenů.

Kompletní ceník je dostupný zde: [OpenAI Pricing](https://openai.com/pricing)

---

## Nastavení prostředí pro práci s API OpenAI GPT-3.5

### Instalace potřebných knihoven
Pro interakci s API OpenAI budeme potřebovat knihovnu `openai`. Můžete ji nainstalovat pomocí následujícího příkazu:

```bash
pip install openai
```

### Vytvoření API klíče pro autentizaci našich požadavků na API OpenAI
Postup je následující:

* Přejděte na stránku https://platform.openai.com/account/api-keys.
* Pokud nemáte účet OpenAI, vytvořte si jej.
* Po přihlášení klikněte na tlačítko pro vytvoření nového API klíče.
* Zadejte název pro svůj klíč a klikněte na "Create new secret key".

### Nastavení přístupových údajů pro API OpenAI
Abyste nastavili svůj API klíč zadejte následující kód:

```python
import openai

# Nahraďte 'sk-abcdefgh1234567890' vaším API klíčem
openai.api_key = 'sk-abcdefgh1234567890'
```

Nyní, když máme připravené prostředí, můžeme začít s odesíláním našich prvních požadavků na ChatGPT prostřednictvím API OpenAI GPT-3.5!

## Odeslání základního požadavku
Požadavek pošleme pomocí `openai.ChatCompletion.create` následovně:

```python
response = openai.ChatCompletion.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "system", "content": "Jsi můj asistent."},
        {"role": "user", "content": "Přelož následující text z čestiny do Francouzštiny: 'Ahoj, jak se máš?'"},
    ]
)
print(response['choices'][0]['message']['content'].strip())
```

```bash
Bonjour, comment ça va ?
```

## Porozumění odpovědi
Požadavek na API vrací odpověď ve formátu JSON, který obsahuje několik užitečných informací, ale nás zajímá pouze klíč `choices`, který obsahuje pole s odpověďmi, v našem případě je pouze jedna odpověď, obsahující textový řetězec, který je výstupem modelu. Používáme metodu `strip()` k odstranění přebytečných bílých znaků z výstupu.

## Vysvětlení parametru "role"
Role sděluje modelu, jak by měl obsah zpracovat. Zde je popis rolí:
* **user** - informuje model, že následující obsah pochází od uživatele – tj. od osoby, která položila otázku,
* **asistent** - informuje model, že obsah byl vygenerován jako reakce na uživatele,
* **systém** - zpráva specifikovaná vývojářem k "nasměrování" odpovědi modelu. V závislosti na použitém modelu může mít systémová zpráva větší či menší dopad na skutečné odpovědi modelu.

## Práce s kontextem konverzace
Při práci s ChatGPT je důležité chápat, jak API zachází s kontextem konverzace. Pokud chcete, aby API mělo v kontextu předchozí odpovědi, musíte poslat celou historii konverzace. Tímto způsobem může model vidět celou konverzaci a odpovědět konzistentně s předchozími zprávami.

```python
# Příklad pokračování v konverzaci
conversation_history = [
    {"role": "system", "content": "Jsi můj asistent."},
    {"role": "user", "content": "Přelož následující text z čestiny do Francouzštiny: 'Ahoj, jak se máš?'"},
    {"role": "assistant", "content": "Bonjour, comment ça va ?"},
    {"role": "user", "content": "Přelož to samé do španělštiny"}
]

response = openai.ChatCompletion.create(
    model="gpt-3.5-turbo",
    messages=conversation_history
)

print(response['choices'][0]['message']['content'].strip())
```

```bash
¡Hola, cómo estás?
```

V tomto příkladu jsme odeslali čtyři zprávy jako součást požadavku na API: jednu zprávu systému, dvě zprávy uživatele a jednu zprávu asistenta. Poslední zpráva uživatele je nová otázka, na kterou chceme, aby asistent odpověděl. Protože jsme poslali celou historii konverzace, asistent může vidět předchozí otázky a odpovědi a tak lépe odpovědět na novou otázku.

## Některé pokročilejší funkce API

### Maximální délka odpovědi
Omezte délku odpovědi nastavením parametru `max_tokens`. To je užitečné pro kontrolu délky výstupního textu.

```python
response = openai.ChatCompletion.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "system", "content": "Jsi můj asistent."},
        {"role": "user", "content": "Napiš krátkou pohádku na spaní pro mého syna (maximálně 50 slov)."},
    ],
    max_tokens=50
)
print(response['choices'][0]['message']['content'].strip())
```

```text
Byl jednou malý kluk, který nemohl usnout. Tak přišlo na pomoc kouzelné pavoučí pletivo. Když ho pověsil nad postel, obj
```

### "Teplota"
Parametr `temperature` ovlivňuje kreativitu odpovědi. Nižší hodnota (např. 0.2) generuje konzistentnější, méně kreativní text, zatímco vyšší hodnota (např. 0.8) generuje více kreativní, ale méně předvídatelný text.

```python
response = openai.ChatCompletion.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "system", "content": "Jsi můj asistent."},
        {"role": "user", "content": "Napiš krátkou pohádku na spaní pro mého syna (maximálně 50 slov)."},
    ],
    temperature=1
)
print(response['choices'][0]['message']['content'].strip())
```

```text
Byl jednou malý kluk, který se nemohl v noci dobře uklidnit. Jednoho večera mu maminka přišla s kouzelným příběhem. "Když zavřeš oči, zazní jemné šepotání vlnkami a hvězdy tě zahladí do snů. Usni, ať tě noční můry nepobíhají, zavři oči a dej šanci sladkým snům." Kluk se usmál, zavřel oči a rychle se ponořil do krásného spánku.
```

### Další parametry
Další parametry můžete prozkoumat v dokumentaci: [OpenAI API Documentation](https://platform.openai.com/docs/api-reference/chat/create)

## Limitování tokenů v dotazech

Vzhledem k tomu, že při volání OpenAI API se platí za tokeny a modely mívají předem definovaný počet tokenů, se kterými dokáží naráz pracovat, tak je vhodné vědět, jak je v dotazech na API limitovat. V momentě, kdy je počet tokenů větší, než je limit, API vrátí error a nevykoná se.

Je víc možností, jak řešit větší zprávy, je možné je rozdělit na víc zpráv, já jsem ořízl přebytek:

```python
import tiktoken

def send_chatcompletion(
    system_prompt="Jsi můj asistent",
    user_prompt=None,
    chat_model="gpt-3.5-turbo",
    completions_token_limit=500,
):
    response = openai.ChatCompletion.create(
        model=chat_model,
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt},
        ],
        temperature=1,
        max_tokens=completions_token_limit
    )

    return response

def send(
    system_prompt="Jsi můj asistent",
    user_prompt=None,
    chat_model="gpt-3.5-turbo",
    model_token_limit=4097,
    completions_token_limit=500,
):
    tokenizer = tiktoken.encoding_for_model(chat_model)
    token_integers_system = tokenizer.encode(system_prompt)
    token_integers_user = tokenizer.encode(user_prompt)

    wrapped_content_token_count = 11 # magická hodnota která vyjádří konstatní navýšení za request. Je to odvozeno od množství zpráv a parametrů, které se v každém requestu posílají
    user_token_limit = model_token_limit - len(token_integers_system) - wrapped_content_token_count - completions_token_limit

    user_prompt_trimmed = tokenizer.decode(token_integers_user[0:user_token_limit])

    return send_chatcompletion(
        system_prompt=system_prompt,
        user_prompt=user_prompt_trimmed,
        chat_model=chat_model,
        completions_token_limit=completions_token_limit,
    )

velmi_dlouhy_dotaz = "Velmi dlouhy dotaz" * 800

tokenizer = tiktoken.encoding_for_model("gpt-3.5-turbo")
tokens = tokenizer.encode(velmi_dlouhy_dotaz)
print(len(tokens))

send(user_prompt=velmi_dlouhy_dotaz)
```

```text
5600
<OpenAIObject chat.completion id=chatcmpl-88qc17YAJIdTkruVZKhj9MoQkQh7K at 0x78c4f281c950> JSON: {
  "id": "chatcmpl-88qc17YAJIdTkruVZKhj9MoQkQh7K",
  "object": "chat.completion",
  "created": 1697119557,
  "model": "gpt-3.5-turbo-0613",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Omlouv\u00e1m se, ale v\u00e1\u0161 dotaz je p\u0159\u00edli\u0161 dlouh\u00fd a nejsem schopen na n\u011bj odpov\u011bd\u011bt. Mohli byste ho, pros\u00edm, zkr\u00e1tit nebo sd\u011blit konkr\u00e9tn\u011bj\u0161\u00ed informace?"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 3597,
    "completion_tokens": 65,
    "total_tokens": 3662
  }
}
```

## Ukázkový projekt: Automatická Čtečka RSS
V tomto projektu si vytvoříme automatizovanou čtečku RSS, která nám umožní sledovat témata, která nás zajímají, pomocí personalizovaných filtrů. Tato čtečka bude načítat články z RSS feedu, analyzovat je a doporučovat nám články na základě našich zájmů.

### Příprava
Nejdříve budeme potřebovat knihovnu `feedparser` pro načítání RSS feedů a `tiktoken` pro práci s tokeny.

```bash
pip install feedparser tiktoken
```

### Načtení RSS Feedu
Použijeme knihovnu `feedparser` k načtení RSS feedu z vybraného zdroje.

```python
import feedparser

# Nahraďte 'rss_url' URL vašeho RSS feedu
rss_url = 'https://servis.idnes.cz/rss.aspx?c=technet'
feed = feedparser.parse(rss_url)

print(feed)
```

### Vytvoření vlastního profilu
Vlastní profil je možné vytvořit v libovolné srozumitelné formě.

```python
profile = """
Obsah: Zajímá mě jedno z následujících témat:
* umělá inteligence
* programování v jednom z jazyků Python, Golang, C# nebo frontendový vývoj v Typescript - React a Angular
* hardwarové technologické novinky
* softwarové novinky, případně novinky v nových verzích existujícího softwaru
""".strip()
```

### Analýza článků
A teď už výsledný kód analýzy článků:

```python
import tiktoken
import json

def send(
    system_prompt=None,
    user_prompt=None,
    chat_model="gpt-3.5-turbo",
    model_token_limit=4097,
):
    tokenizer = tiktoken.encoding_for_model(chat_model)
    token_integers_system = tokenizer.encode(system_prompt)
    token_integers_user = tokenizer.encode(user_prompt)

    completions_token_limit = 500
    wrapped_content_token_count = 11 # magická hodnota která vyjádří konstatní navýšení za request. Je to odvozeno od množství zpráv a parametrů, které se v každém requestu posílají
    user_token_limit = model_token_limit - len(token_integers_system) - wrapped_content_token_count - completions_token_limit

    user_prompt_trimmed = tokenizer.decode(token_integers_user[0:user_token_limit])

    response = openai.ChatCompletion.create(
        model=chat_model,
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt_trimmed},
        ],
        temperature=1,
        max_tokens=completions_token_limit
    )

    return response

def is_article_relevant(article_title, article_description, profile):
    prompt = """
Dám ti můj profil a článek a ty vyhodnotíš, zda mě ten článek bude zajímat. Odpovíš v JSON formátu s následující strukturou: `{{ "odpoved": "<ano nebo ne>", "duvod": "<krátké vysvětlení, proč mě to má nebo nemá zajímat>"}}`.
Můj profil:
'''
{profile}
'''

Titulek článku: `{title}`
""".format(profile=profile, title=article_title, description=article_description)

    response = send(
        system_prompt="Jsi moje automatická čtečka RSS článků, který vyhodnotí zda je pro mě článek zajímavý.",
        user_prompt=prompt,
    )

    result = json.loads(response['choices'][0]['message']['content'].strip().lower())
    print(result)

    if result["odpoved"] == 'ano':
      return True

    return False

# Projděte prvních 10 článků z feedu
for entry in feed.entries[0:10]:
    article_title = entry.title
    article_description = entry.description
    if is_article_relevant(article_title, article_description, profile):
        print(f"DOPORUČENÝ ČLÁNEK: {entry.link}")
    else:
        print(f"nedoporučený článek: {article_title}")
```

Kde výstup je následující:

```text
{'odpoved': 'ne', 'duvod': 'tento článek se neodkazuje na žádné z tvých zájmů. hovoří o využití počítačového rozhraní, což je hlavně ovládnutí uživatelského rozhraní a nezabývá se žádným konkrétním tématem, které tě zajímá.'}
nedoporučený článke: KVÍZ: Umíte využít počítačové rozhraní na maximum?
{'odpoved': 'ne', 'duvod': 'článek se týká historické události a nemá souvislost s umělou inteligencí, programováním, hardwarovými nebo softwarovými novinkami, které tě zajímají.'}
nedoporučený článke: První vzdušná vítězství izraelského letectva vybojovala československá stíhačka
{'odpoved': 'ne', 'duvod': 'článek se zabývá tématem leteckého závodu, které není v souladu s tvými zájmy v programování a technologickém pokroku.'}
nedoporučený článke: Nejstarší letecký závod světa má vítěze. Vyhráli francouzští vzduchoplavci
{'odpoved': 'ne', 'duvod': 'článek se zabývá výzkumem kosmických vzorků z planetky bennu, což se nijak nesouvisej s tvými zájmy o umělou inteligenci, programování nebo technologické novinky.'}
nedoporučený článke: Jako černé uhlí s hojným obsahem vody. NASA odhalila vzorky z planetky Bennu
{'odpoved': 'ne', 'duvod': 'článek se zaměřuje na aktualizaci operačního systému windows 11, což není v mé oblasti zájmu.'}
nedoporučený článke: Nový update Windows 11 přinese mimo jiné žádanou funkci hlavního panelu
{'odpoved': 'ne', 'duvod': 'článek se zaměřuje na otázku získání pilotního diplomu číňanem a ekvádorcem v české republice, což se vztažuje k pilotnímu výcviku a celkovému leteckému průmyslu, což neodpovídá zájmům uvedeným ve tvém profilu.'}
nedoporučený článke: „Exoti“ v rejstříku letců. Pilotní diplom získali v ČSR i Číňan a Ekvádorec
{'odpoved': 'ne', 'duvod': 'článek se zabývá jiným tématem než umělou inteligencí, programováním, hardwarovými nebo softwarovými novinkami.'}
nedoporučený článke: Polský vodíkový balon vlétl do drátů vysokého napětí. Slavný závod pokračuje
{'odpoved': 'ne', 'duvod': 'tento článek se nezabývá žádným z mých zájmových témat. je to spíše politický článek, který se zaměřuje na jednu konkrétní událost.'}
nedoporučený článke: Musk doporučil při konfliktu v Izraeli sledovat toxické účty. Pak to smazal
{'odpoved': 'ano', 'duvod': 'tento článek by tě mohl zajímat, jelikož se jedná o hardwarovou technologickou novinku - brýle od applu, které umějí mnoho zajímavých věcí.'}
DOPORUČENÝ ČLÁNEK: https://www.idnes.cz/technet/pc-mac/htc-vive-xr-elite-virtualni-smisena-mixovana-realita-bryle-vr-mr-xr-ar.A231009_122457_hardware_vse#utm_source=rss&utm_medium=feed&utm_campaign=technet&utm_content=main
{'odpoved': 'ne', 'duvod': 'článek se zaměřuje na vojenskou technologii, která se nehodí k žádnému z tvých zájmů.'}
nedoporučený článke: VIDEO: Naposledy v zahraničí. Co předvedl vrtulník Mi-24 AČR na Slovensku
```

ChatGPT mi doporučil 1 článek k přečtení na základě mého profilu.

## Závěr
OpenAI API pro automatizaci osobních workflow má nepřeberné množství použití a má potenciál ušetřit velké množství času.




