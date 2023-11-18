---
title: "Vytvoření vlastního GPT pro efektivní revizi Pull Requestů na GitHubu"
date: 2023-11-18
draft: true
tags:
  - AI
  - ChatGPT
authors:
  - tomas-panik
resources:
  - name: create-a-gpt
    src: "images/create-a-gpt.png"
    title: Create a GPT button
  - name: title-and-description
    src: "images/title-and-description.png"
    title: Title and Description
  - name: gpt-authentication
    src: "images/gpt-authentication.png"
    title: GPT Authentication
  - name: pr-review-demo
    src: "images/pr-review-demo.gif"
    title: DEMO
  - name: pr-review-demo2
    src: "images/pr-review-demo2.gif"
    title: DEMO
---

Společnost OpenAI nedávno představila [vlastní GPT](https://openai.com/blog/introducing-gpts). Stručně shrňme, co je GPT ve srovnání s klasickým ChatGPT: 

1. Obsahuje vlastní předdefinované instrukce
2. Obsahuje vlastní spouštěče konverzace
3. Může plnit více z následujících funkcí najednou (v klasickém ChatGPT jste si museli vybrat jednu). Těmito funkcemi jsou:
    * prohlížení webových stránek
    * generování obrázků DALL-E
    * interpreter kódu
    * vlastní akce (náhražka dříve používaných Pluginů)

Zejména poslední možnost je zajímavá, protože je nyní mnohem jednodušší vytvářet integrace s vlastními akcemi. Vývojáři již nemusí vytvářet plugin (což byl dost složitý proces).

Pokusme se tedy vytvořit GPT, který vývojářům pomůže s revizemi Pull Requestů (PR).

# Kroky
Cílem je zrevidovat stávající PR na GitHubu s pomocí GPT. V případě nalezení problému GPT vytvoří komentáře (ve stavu PENDING).

Definujme kroky:

1. (Jako reviewer) vložím adresu URL PR GitHubu do GPT chatu.
2. GPT stáhne diff PR pomocí GitHub API (ověřený jako já, takže bude mít přístup i do soukromých repozitářů).
3. GPT zkontroluje diff, zanalyzuje ho na chyby a problémy a navrhne vylepšení.
4. GPT mi vrátí tato zjištění a zeptá se mě, jestli je chci odeslat formou komentáře (ve stavu PENDING) do PR.
5. Po kliknutí na "Yes" odešle tyto revizní komentáře.

Účelem není odeslat všechny návrhy GPT, aniž by je reviewer zkontroloval. Chci používat komentáře GPT jen jako nápovědu, na co se mám podívat, takže budu většinu času tyto komentáře aktualizovat/mazat. Proto je nutné zajistit, aby revize odeslaná na GitHub zůstala ve stavu PENDING a tyto automatické komentáře byly rozpoznatelné, takže používám prefix `By GPT:`.

# Vytvoření vlastního GPT
Při vytváření vlastního GPT musíte mít předplatné GPT Plus! 
Přejděte na webové stránky [ChatGTP](chat.openai.com). Vlastní GPT vytvoříme kliknutím na `Explore` a poté na `Create a GPT` (viz obrázek níže).

{{< img name="create-a-gpt" lazy=false size="origin" >}}

# Konfigurace GPT
Pokračujme tedy v konfiguraci GPT, přeskočíme GPT Builder a vše nakonfigurujeme manuálně.

## Název a popis
Nastavil jsem následující:

Název: `GitHub PR Code Reviewer`

Popis: `Expert at GitHub PR code reviews, using GitHub API for insightful feedback.`

{{< img name="title-and-description" lazy=false size="origin" >}}

## Akce

Nejdříve nakonfigurujeme akce a teprve potom instrukce, protože jsou základní součástí konfigurace.
Akce jsou manifestem [specifikace OpenAPI](https://spec.openapis.org/oas/v3.1.0) zapsaným ve formátu JSON. 

Klikněte na `Create new actions` v konfiguraci GPT a nastavte následující schéma:
```json
{
  "openapi": "3.1.0",
  "info": {
    "title": "GitHub API",
    "description": "Retrieves the diff of a specified pull request from a GitHub repository as well as submitting PR review.",
    "version": "v1.0.0"
  },
  "servers": [
    {
      "url": "https://api.github.com"
    }
  ],
  "paths": {
    "/repos/{owner}/{repo}/pulls/{pull_number}/files": {
      "get": {
        "description": "Get the diff of individual files in PullRequest.",
        "operationId": "GetPullRequestDiff",
        "parameters": [
          {
            "name": "owner",
            "in": "path",
            "description": "Owner of the repository.",
            "required": true,
            "schema": {
              "type": "string"
            }
          },
          {
            "name": "repo",
            "in": "path",
            "description": "Repository name.",
            "required": true,
            "schema": {
              "type": "string"
            }
          },
          {
            "name": "pull_number",
            "in": "path",
            "description": "Pull request number.",
            "required": true,
            "schema": {
              "type": "integer"
            }
          }
        ],
        "responses": {
          "200": {
            "description": "Success",
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": {
                    "$ref": "#/components/schemas/FileDiff"
                  }
                }
              }
            }
          },
          "404": {
            "description": "Not Found"
          }
        }
      }
    },
    "/repos/{owner}/{repo}/contents/{path}": {
      "get": {
        "description": "Get the content of a file in a repository.",
        "operationId": "GetFileContent",
        "parameters": [
          {
            "name": "owner",
            "in": "path",
            "description": "Owner of the repository.",
            "required": true,
            "schema": {
              "type": "string"
            }
          },
          {
            "name": "repo",
            "in": "path",
            "description": "Repository name.",
            "required": true,
            "schema": {
              "type": "string"
            }
          },
          {
            "name": "path",
            "in": "path",
            "description": "Path to the file.",
            "required": true,
            "schema": {
              "type": "string"
            }
          },
          {
            "name": "ref",
            "in": "query",
            "description": "The name of the commit/branch/tag. Default: the repository\u2019s default branch (usually master).",
            "required": false,
            "schema": {
              "type": "string"
            }
          }
        ],
        "responses": {
          "200": {
            "description": "Success",
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/FileContent"
                }
              }
            }
          },
          "404": {
            "description": "Not Found"
          }
        }
      }
    },
    "/repos/{owner}/{repo}/pulls/{pull_number}/reviews": {
      "post": {
        "description": "Create a review for a pull request in a pending state.",
        "operationId": "SubmitPullRequestReview",
        "parameters": [
          {
            "name": "owner",
            "in": "path",
            "description": "Owner of the repository.",
            "required": true,
            "schema": {
              "type": "string"
            }
          },
          {
            "name": "repo",
            "in": "path",
            "description": "Repository name.",
            "required": true,
            "schema": {
              "type": "string"
            }
          },
          {
            "name": "pull_number",
            "in": "path",
            "description": "The number of the pull request.",
            "required": true,
            "schema": {
              "type": "integer"
            }
          }
        ],
        "requestBody": {
          "description": "Review details",
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "$ref": "#/components/schemas/PullRequestReview"
              }
            }
          }
        },
        "responses": {
          "200": {
            "description": "Review successfully submitted",
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/ReviewResponse"
                }
              }
            }
          }
        }
      }
    }
  },
  "components": {
    "schemas": {
      "FileDiff": {
        "type": "object",
        "properties": {
          "sha": {
            "type": "string"
          },
          "filename": {
            "type": "string"
          },
          "status": {
            "type": "string"
          },
          "additions": {
            "type": "integer"
          },
          "deletions": {
            "type": "integer"
          },
          "changes": {
            "type": "integer"
          },
          "contents_url": {
            "type": "integer"
          },
          "patch": {
            "type": "string"
          }
        }
      },
      "FileContent": {
        "type": "object",
        "properties": {
          "type": {
            "type": "string"
          },
          "encoding": {
            "type": "string"
          },
          "size": {
            "type": "integer"
          },
          "name": {
            "type": "string"
          },
          "path": {
            "type": "string"
          },
          "content": {
            "type": "string"
          },
          "sha": {
            "type": "string"
          },
          "url": {
            "type": "string"
          },
          "git_url": {
            "type": "string"
          },
          "html_url": {
            "type": "string"
          },
          "download_url": {
            "type": "string"
          }
        }
      },
      "PullRequestReview": {
        "type": "object",
        "required": ["comments"],
        "properties": {
          "comments": {
            "type": "array",
            "description": "A list of review comments.",
            "items": {
              "$ref": "#/components/schemas/ReviewComment"
            }
          }
        }
      },
      "ReviewComment": {
        "type": "object",
        "properties": {
          "path": {
            "type": "string",
            "description": "The relative path to the file that the comment is attached to."
          },
          "position": {
            "type": "integer",
            "description": "The line index in the diff to place the comment."
          },
          "body": {
            "type": "string",
            "description": "Text of the comment."
          }
        }
      },
      "ReviewResponse": {
        "type": "object",
        "properties": {}
      }
    }
  }
}
```

Pomocí rozhraní GitHub API umožníme GPT provádět 3 akce:
* **GetPullRequestDiff** (`GET` - `/repos/{owner}/{repo}/pulls/{pull_number}/files`) - stáhne diff pro všechny soubory v PR.
  * Poznámka: Zkoušel jsem použít `/repos/{vlastník}/{repo}/pulls/{pull_number}` s hlavičkou `Accept` nastavenou na `application/vnd.github.v3.diff`, ale ukázalo se, že GPT nemá povoleno nastavovat hlavičky, takže může použít pouze routy, kde nadefinování hlavičky není nutné. 
* **GetFileContent** (`GET` - `/repos/{owner}/{repo}/contents/{path}`) - nepovinné. Nevím, zda to GPT někdy použije.
* **SubmitPullRequestReview** (`POST` - `/repos/{owner}/{repo}/pulls/{pull_number}/reviews`) - odeslání PR review.
  * Poznámka: Pro body nedefinujeme vlastnost `event`, protože nechceme, aby ji GPT vyplňoval. Tím zajistíme, že komentář PR bude vždy ve stavu PENDING.

## Ověřování
Existují dva způsoby, jak může GPT ověřovat požadavky API:
* API klíč - basic, bearer nebo vlastní typ autorizace.
* OAuth

I když je OAuth pravděpodobně lepší způsob, budu prozatím používat API klíč. Kroky k jeho vygenerování v GitHubu jsou následující:
* Přejděte na `https://github.com/settings/tokens`
* Klikněte na `Generate new token` (já používám klasické tokeny).
* Vyplňte `Note`, `Expiration` podle svých preferencí a v `Scopes` zaškrtněte `repo` scope.
* Klikněte na `Generate token` a uložte token pro pozdější použití.

Nyní nakonfigurujeme ověřování v konfiguraci GPT. 

{{< img name="gpt-authentication" lazy=false size="origin" >}}

## Instructions

Nejdůležitější částí jsou instrukce. Po dlouhém dolaďování jsem přišel s tímto:

```text
The primary role of 'GitHub PR Code Reviewer' is to assist in GitHub Pull Request code reviews. 

# When user request you to review PR by its URL
If the request is `Review PR` you will only answer `Which PR?` and wait for user to submit PR link. 
If the request is directly PR Url you will follow directly with review of such PR.

## Follow these steps in order
- Step 1: Download patches for individual changed files, analyze them and look for "Findings". 
- Step 2: If necessary query full file content for broader context
- Step 3: Respond your findings

## Explanation of "Findings"
- Finding can be code smell, issue or bug
- What you should look at in patches:
    - The code is well-designed.
    - The functionality is good for the users of the code.
    - Any UI changes are sensible and look good.
    - Any parallel programming is done safely.
    - The code isn't more complex than it needs to be.
    - The developer isn't implementing things they might need in the future but don't know they need now.
    - Code has appropriate unit tests.
    - Tests are well-designed.
    - The developer used clear names for everything.
    - Comments are clear and useful, and mostly explain why instead of what.
    - Code is appropriately documented.

## Explanation of "Position"
The `position` value equals the number of lines down from the first "@@" hunk header in the file you want to add a comment. The line just below the "@@" line is position 1, the next line is position 2, and so on. The position in the diff continues to increase through lines of whitespace and additional hunks until the beginning of a new file. Position is never range it is just single number.

for example:
* the line `uses: actions/checkout@v3` is position 5 in patch: `"@@ -36,7 +36,7 @@ jobs:\n       HUGO_VERSION: 0.111.3\n     steps:\n       - name: Checkout\n-        uses: actions/checkout@v3\n+        uses: actions/checkout@v4\n`
* the line `{{- if .Values.aws_fargate.enabled }}` is position 4 in patch: `@@ -1575,6 +1575,7 @@ receivers:\n    prometheus/node-metrics:\n      config:\n        scrape_configs:\n+{{- if .Values.aws_fargate.enabled }}\n          - job_name: 'kubernetes-nodes-cadvisor'`

## Style of the response
- In the response DO NOT iterate through files, but iterate through "Findings". There can be multiple "Findings" in single file, and there can be also none.
- For each "Finding" will always contain: 
  - Full path to the file
  - Line of code - you will infer this from diff
  - "Position"
  - Explanation of the finding and suggested improvement. 

Follow structure for Findings:
path: <Full path to the file>, lines: <line of code> , position: <"Position">
<finding and suggested improvement>

Individual findings are separated by new line. Full path to the file and "Position" in path are bold. 

Example:
code/main.py:50
This line is unclear what it means, it would be better if you call variable `sumOfLines`.

path: doc/exported_metrics.md, lines: 5-6, position: 1
The documentation for exported metrics is unclear, to make it more clear write this `...`

- Try to suggest fix for the problem if possible
- If there is "Finding" in the file do not respond anything for that particular file
- IMPORTANT: Do not explain what the code is doing, focus on explanation of the "Findings" and improvements suggestions.
- You will always ask if the user wants to submit those findings as PR review

# When user request you to submit findings as PR review
You will use individual findings from your previous response and submit them as PR review in PENDING state, each finding as individual comment.
Here are rules of submitted comments:
* Set `path` parameter from Finding's full file path
* The `position` equals to already explained "Position"
* Set `body` parameter with actual Finding text
* IMPORTANT: Always prefix each comment with following text `**By GPT:** `.
* Full path to the file and line of code of Finding will never be included in comment body.
```

Vysvětlení instrukcí:
* Když uživatel napíše `Review PR` (což použiji jako začátek konverzace), GPT je instruován, aby pouze odpověděl `Which PR?`, aby se minimalizoval textový výstup pro nejběžnější scénář. Uživatel může také napsat přímo URL PR, ale to by neznělo jako chat, že?
* GPT je instruován, jak má postupovat s PR URL a také, že může volitelně stáhnout i celý obsah souboru pro širší kontext.
* GPT je jasně instruován, co je to `Findings` a na co se má v kódu podívat. Použil jsem [souhrn inženýrských postupů Google](https://github.com/google/eng-practices/blob/master/review/reviewer/looking-for.md#summary).
* GPT je jasně instruován, jak má reagovat na `Findings`. 
  * Musel jsem nastavit `IMPORTANT` na instrukci, aby nevysvětloval, co kód dělá, protože to GPT dělal pořád.
* GPT je instruován, jak má odesílat revizi.
  * GPT měl tendenci nastavovat `position` parametr nesprávně, nastavoval ho na příliš vysoké číslo, např. ho nastavoval jako řádek kódu. Tak jsem ho poučil kompletní dokumentací a dokonce i příkladem, jak se `position` počítá.

## Začátek konverzace
Nastavíme pouze jeden: `Review PR`, což je zpráva, kterou jsme explicitně instruovali GPT.

# Spuštění revize
Nyní to otestujme na jednom PR vytvořeném dependabotem v repozitáři tohoto blogpostu: https://github.com/gantrior/tomas-panik-blog/pull/5.

{{< img name="pr-review-demo" lazy=false size="origin" >}}

Skvělé! Funguje to. 

# Ladění..
Nyní zkusme, zda GPT dokáže přeskočit odpovídání na komentáře uživateli a odesílat komentáře přímo, což by revizi trochu urychlilo. 

Na konec instrukcí přidejte následující řádky:
```text
# When user request you to review PR with review submission
If the request is `Review PR with review submission` you will only answer `Which PR?` and wait for user to submit PR link. After user provides PR url, you will review PR with the rules above, but do not print anything to the user, but you will assume that user wants you to submit findings as PR review. So you will submit review right away
```

A přidejte nový začátek konverzace: `Review PR with review submission` 

{{< img name="pr-review-demo2" lazy=false size="origin" >}}

Nyní je to o něco rychlejší.

# Závěr
Vytvořili jsme GPT, který by nám mohl pomoci efektivněji provádět revize kódu. 

Vlastní GPT nám ukazuje velký potenciál, takže uvidíme, co dalšího nám budoucnost přinese. O další nápady na vlastní GPT se podělím v příštích příspěvcích na tomto blogu.

Neváhejte a zanechte komentář níže.








