 

# SpaCy Tricks and Examples

Way more coming..!


Unpossess entities -- remove 's and ' at end of entity:

```
@Language.component("unpossess_entities")
def unpossess_entities(doc):
    new_ents = []
    for ent in doc.ents:
        if doc[ent.end -1].text == "'s" or doc[ent.end -1].text == "'":
            new_ent = Span(doc, ent.start, ent.end-1, label=ent.label)
            new_ents.append(new_ent)
        else:
            new_ents.append(ent)
    doc.ents = new_ents
    return doc
```

