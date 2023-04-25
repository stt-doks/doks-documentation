# DOKS PEP API
DOKS PEP API (later referred as DPA), is a HTTP API for finding company and private person names from PEP lists.

## Index
- [Basics](#basics)
- [Authentication](#authentication)
- [Making API calls](#making-api-calls)
- [HTTP response](#http-response)
- [Search by keyword](#search-by-keyword)
- [Search parameters](#search-parameters)
- [Match score](#match-score)
- [Internal reference](#internal-reference)
- [API response](#api-response)
- [Detail types](#detail-types)
- [Downloading target data](#downloading-target-data)
- [Examples](#examples)
- [Implementation strategies](#implementation-strategies)
- [Support](#support)

## Basics
DPA is used by making API calls as HTTP requests. The host for API calls is:
```
https://dpa.doks.fi/
```

The host is followed by subdirectories:
```
https://dpa.doks.fi/api/current/keyword/
```

SSL/TLS is used to encrypts communications between a client and server.

## Authentication
DPA is using Bearer token authentication and valid API key must be send in the Authorization header:
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
        "keyword": "Sauli Niinistö"
    }
]
```

```sh
curl -X POST https://dpa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "PERSON", "keyword": "Sauli Niinistö"}]'
```

To protect your customer data, sent payload is never saved or logged in DPA.

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
DPA is used for finding company and private person names from PEP (politically exposed person) lists. The search is made by sending objects as array. Basic use case is to send a single object, in other words search for a single name:

```HTTP
POST /api/current/keyword/
Authorization: Bearer place-your-apikey-here
Content-Type: application/json

[
    {
        "keyword": "Sauli Niinistö"
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
        "keyword": "Sauli Niinistö"
        "entityType": "PERSON"
    },

    {
        "keyword": "Sanna Marin"
        "entityType": "PERSON"
    }
]
```

Maximum number of objects in a single API call is `100`.

## Search parameters
Search can be fine-tuned by using parameters:

```json
{
    "keyword": "Sauli Niinistö",
    "entityType": "PERSON",
    "ref": "your-internal-reference",
    "minScore": 0.9321
}
```

| Parameter  | Type             | Mandatory | Explanation                                                                                 |
| ---------- | ---------------- | --------- | ------------------------------------------------------------------------------------------- |
| keyword    | string           | yes       | A keyword that is used in search.                                                           |
| entityType | string           | yes       | Search context, `PERSON` or `ORGANIZATION`.                                                 |
| ref        | string           | no        | Your internal reference for this search. Can be used to help handling the response.         |
| minScore   | number           | no        | Applied matching score for search from value between `0.50` and `1.00`. Defaults to `0.75`. |

## Entity types
Every search must be made in the context of company or individual person. It also significantly reduces false-positives. There is two possible values for entity type:

- `PERSON` for searching private persons
- `ORGANIZATION` for searching companies, associations and similar.

Entity type is is defined as mandatory parameter `entityType` in search object.

## Match score
DPA searches given keywords from PEP lists with high-end name matching algorithm. The algorithm can match with multiple strategies, including phonetic similarity and transliteration spelling differences. It also takes care for example misspellings, aliases, nicknames, initials and names in different languages.

The match score is a percentage calculated by DPA name matching algorithm between two names: used keyword and name in PEP lists.

By introducing the `minScore` -parameter in search object, the results are being filtered accordingly. If the value is low, it will lead to more hits but also with more false-positive hits, usually causing need for manual work for reviewing. If the value is high, it will lead to fewer hits (reducing the need for manual review work), but there is a possibility for missing the true-positive hit.

DPA uses default of `75%` (0.75) for searches. This can be considered as a low but safe value, resulting large number of hits but also with false-positives. Commonly used best-practice for PEP list search is `85%` (0.85).

Read more about the match score from [here](DSA_yleiskatsaus_nimien_vertailuun.pdf) (in Finnish).

## Internal reference
An internal reference can be placed in search object as string. Using this `ref` -parameter does not affect the search results in any way. It is passed as-is in results and can be used to help handling possible hits when search is made with multiple search objects in a single API call. 

## API response
DPA responses for keyword search by giving a list of hits, list of hit targets and few helpfull utilities:

```json
{
    "maxScore": 1,
    "numHits": 2,
    "numTargets": 2,
    "hits": [
        {
            "id": "dpa-09c1f48d",
            "keyword": "Sauli Niinistö",
            "match": "Sauli NIINISTO",
            "score": 0.99,
            "ref": "123456"
        },
        {
            "id": "dpa-eba0ffdd",
            "keyword": "Sauli Niinistö",
            "match": "Sauli Niinistö",
            "score": 1,
            "ref": "123456"
        }
    ],
    "targets": {
        "dpa-09c1f48d": {
            "url": "https://dpa.doks.fi/api/current/download/?id=dpa-09c1f48d",
            "details": [
                {
                    "nameType": "PRIMARY",
                    "type": "NAME",
                    "value": "Sauli NIINISTO"
                },
                {
                    "type": "GENDER",
                    "value": "UNKNOWN"
                },
                {
                    "type": "CITIZENSHIP",
                    "value": "FI"
                },
                {
                    "type": "POSITIONS",
                    "value": "Pres."
                }
            ]
        },
        "dpa-eba0ffdd": {
            "url": "https://dpa.doks.fi/api/current/download/?id=dpa-eba0ffdd",
            "details": [
                {
                    "nameType": "PRIMARY",
                    "type": "NAME",
                    "value": "Sauli Niinistö"
                },
                {
                    "type": "BIRTHDATE",
                    "value": "24/08/1948"
                },
                {
                    "type": "GENDER",
                    "value": "MALE"
                },
                {
                    "type": "BIRTHPLACE",
                    "value": "Salo"
                },
                {
                    "type": "CITIZENSHIP",
                    "value": "FI"
                },
                {
                    "type": "OCCUPATIONS",
                    "value": "politician"
                },
                {
                    "type": "POSITIONS",
                    "value": "1974-01-01 - 1975-01-01  lensmann|1995-04-13 - 1996-02-01  Minister of Justice of Finland|1987-03-21 - 1999-03-23  member of the Parliament of Finland|1995-04-13 - 2001-08-30  Deputy Prime Minister of Finland|1996-02-02 - 1999-04-15  Minister of Finance|1999-04-15 - 2003-04-16  Minister of Finance|2007-03-21 - 2011-04-19  member of the Parliament of Finland|2009-01-01 - 2012-01-01  chairperson|1994-01-01 - 2001-01-01  Chairperson of the National Coalition Party|1999-03-24 - 2003-03-18  member of the Parliament of Finland|2007-04-24 - 2011-04-27  Speaker of the Parliament of Finland|Since 2012-03-01  President of Finland"
                },
                {
                    "type": "DESCRIPTION",
                    "value": "President of Finland since 2012"
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
| numHits    | integer          | Number of hits. Hit is a match for given keyword against name detail in single pep list entry.       |
| numTargets | integer          | Number of targets. Target is a single pep list entry. There can be several hits for a single target. |
| hits       | array of objects | List of hit objects as array.                                                                             |
| targets    | object           | Object containing all unique targets, eg. pep list entries. Object key is a target id.               |

### Hits:
| Field   | Type   | Explanation                                                      |
| ------- | ------ | ---------------------------------------------------------------- |
| id      | string | Target id for a hit. Represents a single pep list entry.    |
| keyword | string | Used keyword for a search.                                       |
| match   | string | Name that has been matched for a keyword in pep list entry. |
| score   | number | Match score for this hit.                                        |
| ref     | string | Your internal reference, passed as-is from the initial API call. |

### Hit targets:
| Field   | Type             | Explanation                                                                                                |
| ------- | ---------------- | ---------------------------------------------------------------------------------------------------------- |
| url     | string           | Url for downloading details for this target. Can be used to create UI for false-positive match processing. |
| details | array of objects | List of all known details for this target.                                                                 |

### Hit target details:
| Field    | Type   | Explanation                                              |
| -------- | ------ | -------------------------------------------------------- |
| type     | string | Detail type.                                             |
| value    | string | Value for a single detail.                               |
| nameType | string | Type of name. Present only if the detail type is `NAME`. |

## Detail types
DPA consolidates the data from pep lists and output's results in unified format. This is very helpfull for handling the results. Details for a single target (pep list entry) can contain following detail types:

- `CITIZENSHIP`
- `DESCRIPTION`
- `BIRTHDATE`
- `BIRTHPLACE`
- `GENDER`
- `NAME`
- `OCCUPATIONS`
- `POSITIONS`

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
https://dpa.doks.fi/api/current/download/?id=dpa-09c1f48d
```

Output type can be changed to pdf with a query parameter:
```
https://dpa.doks.fi/api/current/download/?id=dpa-09c1f48d&type=pdf
```

Please note, that similar to other API calls to DPA, also this call needs an Bearer token authentication and valid API key must be send in the Authorization header:

```sh
curl -X GET 'https://dpa.doks.fi/api/current/download/?id=dpa-09c1f48d'
-H 'Authorization: Bearer place-your-apikey-here'
```

## Examples

Search by a single private person name:
```sh
curl -X POST https://dpa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "PERSON", "keyword": "Sauli Niinistö"}]'
```

Search by a single company name:
```sh
curl -X POST https://dpa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "ORGANIZATION", "keyword": "Schenker"}]'
```

Search by a multiple private person names:
```sh
curl -X POST https://dsa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "PERSON", "keyword": "Sauli Niinistö"}, {"entityType": "PERSON", "keyword": "Sanna Marin"}]'
```

Search by a multiple names, mixing entity types:
```sh
curl -X POST https://dsa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "PERSON", "keyword": "Sauli Niinistö"}, {"entityType": "ORGANIZATION", "keyword": "Schenker"}]'
```

Search with adjusted (very strict) matching requirements:
```sh
curl -X POST https://dsa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "PERSON", "keyword": "Sauli Niinistö", "minScore": 0.98}]'
```

Search by a multiple names and placing your internal reference:
```sh
curl -X POST https://dsa.doks.fi/api/current/keyword/
-H 'Authorization: Bearer place-your-apikey-here'
-H 'Content-Type: application/json'
-d '[{"entityType": "PERSON", "keyword": "Sauli Niinistö", "ref": "abc-123"}, {"entityType": "ORGANIZATION", "keyword": "Schenker", "ref": "def-456"}]'
```

## Support
Please contact tech@doks.fi for support.
