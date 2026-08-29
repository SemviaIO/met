# met

A Semvia semantic package over the Metropolitan Museum of Art's Open Access
catalogue — 484,956 objects, and the artists they are attributed to.

It ships no data. What it ships is a description: a small ontology, the
nineteen curatorial departments as a SKOS scheme, a Postgres connection
descriptor carrying no credential, and two RML TriplesMaps that tell the Semvia
federation engine how to read one wide relation as RDF. Install it and the
catalogue becomes queryable over SPARQL, answered by the database at query time.
Nothing is copied and nothing is materialized.

## Install

```json
{
  "requires": {
    "https://github.com/SemviaIO/met": "SemviaIO/met#v0.1.0"
  }
}
```

Any git ref works in place of the tag — a branch or a commit SHA, in the
`owner/repo#ref` form.

The connection descriptor names the database but carries **no password**. The
installer binds it out of band: `svdb:password` is a secret-marked field, and the
binding is keyed to the descriptor's own subject IRI,
`https://github.com/SemviaIO/met/schema/connection#met-source`, so the credential
is released only for the destination you consented to. No credential appears
anywhere in this repository, and none ever will.

## What it maps

The Met's Open Access export is **one** table. Everything below is a reading of
`public.objects`, not a join across relations.

| Source relation | TriplesMap | Yields |
| --- | --- | --- |
| `public.objects` | `Objects` | one `met:Object` per catalogue row |
| `public.objects` | `Artists` | one `met:Artist` per constituent identifier, from the artist columns of the same row |

Objects are subject-keyed on their object identifier, at
`https://data.metmuseum.org/object/{object_id}`; artists on their constituent
identifier, at `https://data.metmuseum.org/artist/{constituent_id}`. Departments
are minted from the department name, which is why the SKOS concepts in
`departments.ttl` carry exactly the IRIs the template produces — including the
percent-encoding.

```
schema/
  ontology.ttl      the classes and predicates
  connection.ttl    where the database is (no credential)
  objects.md        the object-side mapping
  artists.ttl       the artist-side mapping
  departments.ttl   the nineteen curatorial departments as a SKOS scheme
```

The object mapping is authored as Markdown rather than Turtle. That is not a
convenience — a mapping is a document a human reads and argues with, and reading
it should not require reading RDF. `sem build` materializes it into the Turtle
the engine consumes.

`artists.ttl` is the exception, and the file says why in its header: every one of
its object maps is a three-deep RML-FNML function execution, and the Markdown
surface has one line per glue node and no spelling for a function *tree*. Rather
than author the wrong mapping in the nicer format, that map drops to Turtle.

## The model

Two classes. `met:Object` is a catalogue row. `met:Artist` is a constituent
identifier, and what that identifier actually identifies is the single most
important thing to know before querying this package — see below.

Predicates are named for what they mean rather than for the column they read.
The column is `artist_display_bio`; there is no `met:artistDisplayBio`, because a
question answered through a predicate named after a column is only SQL spelled in
SPARQL. Where a column's content resisted being given a meaning, the predicate
says so in its name — `met:tagsRaw` is the cell, not a tag.

## Caveats worth knowing before you query

**A `constituent_id` is a per-row attribution token, not an artist key.** Of its
48,365 distinct values, 9,361 identify one maker and **39,004 are fused
multi-maker tokens** — `1070527632` is the two-engraver pair
`Joos de Bosscher|Abraham de Bruyn`, and `7632610705` is the same pair listed the
other way round. The two sets are disjoint: no identifier is ever used both ways.
184,946 objects are attributed through a real identifier, 97,567 through a fused
one, and 202,443 through none at all. The package maps all of them and declares
this rather than filtering to the clean 9,361, because filtering would silently
drop the creator edge from 97,567 objects — and a silent loss is worse than a
declared one. A count of `met:Artist` is a count of attribution tokens.

**Multi-valued cells are projected to their first field.** The same 97,567 rows
pipe-pack all six artist columns, and the mapping keeps field one of each. The
field counts match across all six columns on every packed row, so what you get is
a coherent tuple for the first maker — and nothing at all for the co-attributed
ones. That is
[semvia#5719](https://github.com/SemviaIO/SemviaIO/issues/5719); when the engine
grows per-cell fan-out, this package gets the rest of the makers without a
modelling change.

**`met:tagsRaw` and `met:country` are packed too, and are not repaired.** `tags`
is one literal per row: filtering `= "Birds"` finds 2,895 rows and misses the
5,430 where `Birds` shares its cell. `country` is packed on 1,118 rows, so "how
many Egyptian objects" has three defensible answers — 31,296 for an exact match,
31,385 counting pipe-delimited tokens, 33,461 for a substring match that also
catches `Egyptian Sudan`. Only the first is expressible here.

**Artist dates are strings; object dates are integers.** `object_begin_date` and
`object_end_date` hold an integer on all 484,956 rows (BCE as a negative), so
they are stamped `xsd:integer` and can be ordered and compared. The artist date
columns cannot: after trimming, 1,899 begin dates are full ISO dates like
`1895-01-21` and 429 are negative for BCE. Stamping them anyway would mint
`"1895-01-21"^^xsd:integer` without complaint, because the term constructor does
not validate a lexical form against its datatype
([semvia#5720](https://github.com/SemviaIO/SemviaIO/issues/5720)). They stay
`xsd:string`, and the shapes say why.

**`met:isHighlight` is a string, for the same reason.** The column holds `True`
and `False`, which are not `xsd:boolean` lexical forms. Filter on the string.

**`owl:sameAs` is projected, never traversed.** An artist's Wikidata and ULAN
identifiers and an object's Wikidata identifier ride `owl:sameAs`. Semvia stores
that edge and does not entail across it, so a query that reads the identifier as
a value works and a query that tries to *join through* it returns nothing —
without erroring. Read it; do not route through it. One identifier reaches up to
ten distinct Wikidata IRIs across its rows, which is another reason not to.

**No `sh:minCount` on any mapped property.** These shapes describe a virtual
graph over a source we do not control, and that source has nulls — 202,443 rows
carry no artist at all, and 435,415 carry no gallery number. A shape is
documentation here, not a gate the source could satisfy.

## Licence and attribution

Everything Semvia authored here is CC0 1.0 Universal — see `LICENSE`.

The data is the Met's and is not redistributed by this package. See `NOTICE` for
the source, the Met's terms of use, and what this package does and does not
claim.
