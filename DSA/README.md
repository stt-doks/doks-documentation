# DOKS Sanctions API
DOKS Sanctions API (later referred as DSA), is a HTTP API for finding company and private person names from official sanctions lists.

Read more about DSA:
- [In English](DSA_short_en.pdf)
- [In Finnish #1](DSA_short_fi.pdf)
- [In Finnish #2](DSA_fi.pdf)

## Index
- [Basics](#basics)
- [Authentication](#authentication)
- [Making API calls](#making-api-calls)
- [HTTP response](#http-response)
- [Search by keyword](#search-by-keyword)
- [Search parameters](#search-parameters)
- [Sanction lists](#sanction-lists)
- [Match score](#match-score)
- [Internal reference](#internal-reference)
- [API response](#api-response)
- [Detail types](#detail-types)
- [Downloading target data](#downloading-target-data)
- [Examples](#examples)
- [Implementation strategies](#implementation-strategies)
- [Support](#support)

## Basics
DSA is used by making API calls as HTTP requests. The host for API calls is:
```
https://dsa.doks.fi/
```

The host is followed by subdirectories:
```
https://dsa.doks.fi/api/current/keyword/
```

SSL/TLS is used to encrypts communications between a client and server.

## Authentication
DSA is using Bearer token authentication and valid API key must be send in the Authorization header:
```http
Authorization: Bearer place-your-apikey-here
```

## Making API calls
API call is made with HTTP method `POST` and payload is sent as JSON in request body. For example:

```http
POST /api/current/keyword/
Authorization: Bearer place-your-apikey-here
Content-Type: application/json

[
    {
        "entityType": "PERSON",
         "keyword": "Saddam Hussein"
    }
]
```

```sh
curl -X POST https://dsa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "PERSON", "keyword": "Saddam Hussein"}]'
```

To protect your customer data, sent payload is never saved or logged in DSA.

## HTTP response
Successfull API call always returns with HTTP response code:
```
HTTP/2 200
```

If any else response code is returned, API call should be considered as failed.

Response payload is returned in JSON-format. Gracefully failed API calls returns error messages, for example:
```json
{
    "errors":["Invalid API key"]
}
```

## Search by keyword
DSA is used for finding company and private person names from official sanctions lists. The search is made by sending objects as array. Basic use case is to send a single object, in other words search for a single name:

```HTTP
POST /api/current/keyword/
Authorization: Bearer place-your-apikey-here
Content-Type: application/json

[
    {
        "keyword": "Saddam Hussein"
        "entityType": "PERSON"
    }
]
```

It is possible to add multiple objects in a single API call. This reduces the number of round-trips in server but involves more complex results handling for client:

```http
POST /api/current/keyword/
Authorization: Bearer place-your-apikey-here
Content-Type: application/json

[
    {
        "keyword": "Saddam Hussein"
        "entityType": "PERSON"
    },

    {
        "keyword": "Lukoil"
        "entityType": "ORGANIZATION"
    }
]
```

Maximum number of objects in a single API call is `100`.

## Search parameters
Search can be fine-tuned by using parameters:

```json
{
    "keyword": "Saddam Hussein",
    "entityType": "PERSON",
    "ref": "your-internal-reference",
    "sources": ["EUFINANCIAL", "UN"],
    "minScore": 0.9321
}
```

| Parameter  | Type             | Mandatory | Explanation                                                                                 |
| ---------- | ---------------- | --------- | ------------------------------------------------------------------------------------------- |
| keyword    | string           | yes       | A keyword that is used in search.                                                           |
| entityType | string           | yes       | Search context, `PERSON` or `ORGANIZATION`.                                                 |
| ref        | string           | no        | Your internal reference for this search. Can be used to help handling the response.         |
| sources    | array of strings | no        | List of used sanction list sources. Defaults to all possible sanction lists.                |
| minScore   | number           | no        | Applied matching score for search from value between `0.50` and `1.00`. Defaults to `0.75`. |

## Entity types
Every search must be made in the context of company or individual person. This is due the nature of the original format of official sanctions lists. It also significantly reduces false-positives. There is two possible values for entity type:

- `PERSON` for searching private persons
- `ORGANIZATION` for searching companies, associations and similar.

Entity type is is defined as mandatory parameter `entityType` in search object.

## Sanction lists
DSA includes data from multiple sanctions lists. As default, the search covers all these lists. It is possible to adjust this, by entering wanted lists (sources) in search object as array. If the `source` -parameter is not set or it is empty, the search is made with all available sources.

| Target        | Explanation                                                                                        |
| ------------- | -------------------------------------------------------------------------------------------------- |
| `OFACSDN`     | Office of Foreign Asset Control (OFAC) Specially Designated Nationals and Blocked Persons list     |
| `OFACCONS`    | Office of Foreign Asset Control (OFAC) Consolidated Sanctions List (Non-SDN)                       |
| `KRP`         | Asset freeze decisions made by the Finnish National Bureau of Investigation                        |
| `EUFINANCIAL` | Consolidated list of persons, groups and entities subject to EU financial sanctions                |
| `UN`          | UN Security Council's sanctions list                                                               |
| `UK`          | Office of Financial Sanctions Implementation (OFSI) HM Treasury The UK Sanctions List              |
| `UKCONS`      | Office of Financial Sanctions Implementation (OFSI) HM Treasury Consolidated List                  |
| `BIS`         | International Trade Administration (ITA) / Bureau of Industry and Security (BIS) consolidated list |
| `EUEXTENDED`  | EU sanctions related to certain sectors or events.                                                 |

## Match score
DSA searches given keywords from sanctions lists with high-end name matching algorithm. The algorithm can match with multiple strategies, including phonetic similarity and transliteration spelling differences. It also takes care for example misspellings, aliases, nicknames, initials and names in different languages.

The match score is a percentage calculated by DSA name matching algorithm between two names: used keyword and name in sanction lists.

By introducing the `minScore` -parameter in search object, the results are being filtered accordingly. If the value is low, it will lead to more hits but also with more false-positive hits, usually causing need for manual work for reviewing. If the value is high, it will lead to fewer hits (reducing the need for manual review work), but there is a possibility for missing the true-positive hit.

DSA uses default of `75%` (0.75) for searches. This can be considered as a low but safe value, resulting large number of hits but also with false-positives. Commonly used best-practice for sanctions list search is `85%` (0.85).

Read more about the match score from [here](DSA_yleiskatsaus_nimien_vertailuun.pdf) (in Finnish).

## Internal reference
An internal reference can be placed in search object as string. Using this `ref` -parameter does not affect the search results in any way. It is passed as-is in results and can be used to help handling possible hits when search is made with multiple search objects in a single API call. 

## API response
DSA responses for keyword search by giving a list of hits, list of hit targets and few helpfull utilities:

```json
{
  "maxScore": 0.99,
  "numHits": 3,
  "numTargets": 1,
  "hits": [
    {
      "id": "dsa-b9a1da58",
      "keyword": "Saddam Hussein",
      "match": "Saddam HUSSEIN",
      "score": 0.99,
      "ref": "your-internal-reference"
    },
    {
      "id": "dsa-b9a1da58",
      "keyword": "Saddam Hussein",
      "match": "Saddam HUSSAIN",
      "score": 0.95310426,
      "ref": "your-internal-reference"
    },
    {
      "id": "dsa-b9a1da58",
      "keyword": "Saddam Hussein",
      "match": "Saddam HUSAYN",
      "score": 0.85336906,
      "ref": "your-internal-reference"
    }
  ],
  "targets": {
    "dsa-b9a1da58": {
      "url": "https://dsa.doks.fi/api/current/download/?id=dsa-b9a1da58",
      "source": "OFACSDN",
      "details": [
        {
          "nameType": "PRIMARY",
          "type": "NAME",
          "value": "Saddam Hussein AL-TIKRITI"
        },
        {
          "type": "DESCRIPTION",
          "value": "named in UNSCR 1483; President since 1979"
        },
        {
          "type": "NATIONALITY",
          "value": "Iraq"
        },
        {
          "type": "BIRTHDATE",
          "value": "28 Apr 1937"
        },
        {
          "type": "BIRTHPLACE",
          "value": "al-Awja, near Tikrit, Iraq"
        },
        {
          "nameType": "AKA",
          "type": "NAME",
          "value": "Saddam HUSAYN"
        },
        {
          "nameType": "AKA",
          "type": "NAME",
          "value": "Saddam HUSSAIN"
        },
        {
          "nameType": "AKA",
          "type": "NAME",
          "value": "Saddam HUSSEIN"
        }
      ]
    }
  }
}
```

### Response:
| Field      | Type             | Explanation                                                                                               |
| ---------- | ---------------- | --------------------------------------------------------------------------------------------------------- |
| maxScore   | number           | Maximum score from all the found hits.                                                                    |
| numHits    | integer          | Number of hits. Hit is a match for given keyword against name detail in single sanction list entry.       |
| numTargets | integer          | Number of targets. Target is a single sanction list entry. There can be several hits for a single target. |
| hits       | array of objects | List of hit objects as array.                                                                             |
| targets    | object           | Object containing all unique targets, eg. sanction list entries. Object key is a target id.               |

### Hits:
| Field   | Type   | Explanation                                                      |
| ------- | ------ | ---------------------------------------------------------------- |
| id      | string | Target id for a hit. Represents a single sanction list entry.    |
| keyword | string | Used keyword for a search.                                       |
| match   | string | Name that has been matched for a keyword in sanction list entry. |
| score   | number | Match score for this hit.                                        |
| ref     | string | Your internal reference, passed as-is from the initial API call. |

### Hit targets:
| Field   | Type             | Explanation                                                                                                |
| ------- | ---------------- | ---------------------------------------------------------------------------------------------------------- |
| url     | string           | Url for downloading details for this target. Can be used to create UI for false-positive match processing. |
| source  | string           | Sanction list for this target.                                                                             |
| details | array of objects | List of all known details for this target.                                                                 |

### Hit target details:
| Field    | Type   | Explanation                                              |
| -------- | ------ | -------------------------------------------------------- |
| type     | string | Detail type.                                             |
| value    | string | Value for a single detail.                               |
| nameType | string | Type of name. Present only if the detail type is `NAME`. |

## Detail types
DSA consolidates the data from different sanction lists and output's results in unified format. This is very helpfull for handling the results. Details for a single target (sanction list entry) can contain following detail types:

- `NATIONALITY`
- `ADDRESS`
- `COUNTRY`
- `DESCRIPTION`
- `BIRTHDATE`
- `BIRTHPLACE`
- `TITLE`
- `GENDER`
- `NAME`
- `IDENTIFICATION` for known passport or residence card information

There can be multiple entries of same detail type present in a single target. In other words, one target can have multiple names and even multiple known birthdates.

### Genders
For a detail type of `GENDER`, there is two possible values:
- `MALE`
- `FEMALE`

### Name type
If the detail type is `NAME`, an additional field, `nameType` is filled for a detail. There is following name types available:
- `PRIMARY` for a primary name
- `AKA` for an alias (Also Known As)
- `FKA` for a formerly known name
- `ALT` for a alternative name variation

## Downloading target data
Target url can be used to fetch to fetch ready-prepared html or pdf version that includes all the details ready to be viewed on UI.

Default output type is html:
```
https://dsa.doks.fi/api/current/download/?id=dsa-b9a1da58
```

Output type can be changed to pdf with a query parameter:
```
https://dsa.doks.fi/api/current/download/?id=dsa-b9a1da58&type=pdf
```

Please note, that similar to other API calls to DSA, also this call needs an Bearer token authentication and valid API key must be send in the Authorization header:

```sh
curl -X GET 'https://dsa.doks.fi/api/current/download/?id=dsa-b9a1da58'
-H 'Authorization: Bearer place-your-apikey-here'
```

## Examples

Search by a single private person name:
```sh
curl -X POST https://dsa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "PERSON", "keyword": "Saddam Hussein"}]'
```

Search by a single company name:
```sh
curl -X POST https://dsa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "ORGANIZATION", "keyword": "Lukoil"}]'
```

Search by a multiple private person names:
```sh
curl -X POST https://dsa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "PERSON", "keyword": "Saddam Hussein"}, {"entityType": "PERSON", "keyword": "Vladimir Putin"}]'
```

Search by a multiple names, mixing entity types:
```sh
curl -X POST https://dsa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "PERSON", "keyword": "Saddam Hussein"}, {"entityType": "ORGANIZATION", "keyword": "Lukoil"}]'
```

Search only for EU financial sanctions:
```sh
curl -X POST https://dsa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "PERSON", "keyword": "Vladimir Putin", "sources": ["EUFINANCIAL"]}]'
```

Search with adjusted (very strict) matching requirements:
```sh
curl -X POST https://dsa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "PERSON", "keyword": "Saddam Hussein", "minScore": 0.98}]'
```

Search by a multiple names and placing your internal reference:
```sh
curl -X POST https://dsa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "PERSON", "keyword": "Saddam Hussein", "ref": "abc-123"}, {"entityType": "ORGANIZATION", "keyword": "Lukoil", "ref": "def-456"}]'
```

## Implementation strategies
See [this pdf](Implementing_DSA.pdf) for ideas and insights.

## Support
Please contact tech@doks.fi for support.
