# Fabricius

Fabricius is an Anki plugin that bidirectionally syncs between Roam and Anki. The goal is to have robust, fast syncing for the most common use-cases.

Connecting the east side of the city to Tiber Island since 62 BC, the Pons Fabricius (Fabricius Bridge) is the oldest bridge in Rome to survive to the present day. - Wikipedia

**You are responsible for your data.** While we have tested this library as far as possible, there may still be bugs. Ideally, you should keep backups of both Roam and Anki data.

## Getting started
You need to define a `config.json` as well as custom note type(s) that will be used by the plugin. As a recommended starter configuration, we recommend that you use cloze notes:

```json
{
    "api_key": "...",
    "roam_db_name": "my-db",
    "cards": [
        {
            "model" : "ClozeRoam",
            "deck" : "default",
            "tagMap": {"srs":"Text"}
        }
    ]
}
```

create the following Anki *note type* (also known as *model*) [instructions in anki docs](https://docs.ankiweb.net/#/getting-started?id=note-types):
```text
ClozeRoam
- Text
- TextUID
```

## Syncing cloze notes

When the plugin sees the cloze syntax in Roam text, it will automatically convert it to Anki's cloze syntax. The Anki model should support clozes (but the plugin won't complain if it doesn't).

anki model:
```text
ClozeRoam
- Text
- TextUID
```

text in Roam (*note the modified cloze syntax, since Roam reserves `{{}}` and `::` for internal use*)
```text
    - "In [[C]], {c2:indirect references} to variables are done via {c1:[[pointers]]} #srs"
```

plugin config maps tags to fields in the Anki model:
```json
{
    "cards": [
        {
            "model" : "ClozeRoam",
            "deck" : "default",
            "tagMap": {"srs":"Text"}
        }
    ]
}
```

resulting card in anki 
```
In [[C]], {{c2::indirect references}} to variables are done via {{c1::[[pointers]]}} #srs
```

## Limitations
- If cloze deletions are removed from Roam, the corresponding cloze card needs to be emptied manually in Anki after syncing (Tools -> Empty Cards...)
- The identity of a note in Anki is based on the block ID in roam. If the block ID changes, you will lose the review schedule of the card. Consequently, this means you need to be careful not to delete blocks with SR prompts in them.
- Sync when deck browser is open may cause failure to sync from anki to roam.

## Design Notes

### Sync model (simplified)
For a given config:

```json
{
    "cards": [
        {
            "model" : "MyAnkiModel",
            "deck" : "default",
            "tagMap": {"srs/f":"Front", "srs/e":"Back", "srs/info": "Info"}
        }
    ]
}
```

we will expect all the fields (specified as values in `tagMap`) to exist in the given model. In addition `{field}UID` must also exist as a field. This allows the plugin to track which field in the Anki note maps to which Roam block.

Roam is the source of truth for which notes should exist. If the block uid disappears from Roam, the corresponding note in Anki will be orphaned. This will have to be deleted via `Tools -> Fabricus: Clear Orphans`.

1. Plugin pulls all relevant (block, modification date) based on configured tags from Roam.
2. If the card does not exist in anki, create a new card
3. Load the corresponding notes in Anki and read their modification date
4. If they are unequal, resolve with the newer one (either pushing back to Roam, or adding a card to Anki)

### Ideas for other note models

#### Using Nesting to represent custom cards
Blocks nested under one another are treated as a single note. Not all fields in a note need to be specified, and order of the fields does not matter.

anki model:
```text
MyAnkiModel
- Front
- FrontUID
- Back
- BackUID
- Info
- InfoUID
```

plugin config maps tags to fields in the Anki model:
```json
{
    "cards": [
        {
            "model" : "MyAnkiModel",
            "deck" : "default",
            "tagMap": {"srs/f":"Front", "srs/e":"Back", "srs/info": "Info"}
        }
    ]
}
```

Pros:
- Very flexible

Cons:
- Each model would need to have unique field names
- More complicated to implement since we don't know which field will be a parent block.

#### Using "header" tags for custom models
```text
    - #srs/country <----- everything under this block will be considered as one note in Anki.
        - What is the [[srs/countrycapital]] of Italy? <----- we support this notation too
        - Rome #srs/countrycapitalans
        - Pizza came from Italy. <----- anything without relevant tags will be ignored. 
```

Simpler to implement and also quite intuitive.

## Testing Notes

These are some scenarios to test before releases:

- block changed in roam
- block deleted in roam
- note deleted in Anki
- note changed in Anki
- `config.json`: multiple items in `cards` field