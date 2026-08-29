# Met Objects

::[mappedBy](https://pkg.semvia.io/semvia/folio/mapping#mappedBy) RmlMappingMap

Projects the Metropolitan Museum of Art Open Access catalogue — `public.objects`, 484,956
rows and 54 columns in the `met` database, every column typed `text` — onto `met:Object`.
One TriplesMap over the whole relation, subject-keyed on the Met's own object number.

Nothing here copies data. The mapping is a description the federation engine registers as
a virtual table; every read is answered by Postgres at query time.

The sibling [artists](artists) document maps the SAME relation a second time, keyed on
`constituent_id`, and mints the artist IRIs this document's `dct:creator` points at. Two
TriplesMaps over one physical table is deliberate and supported: the federation engine
keys a registered table on the TriplesMap's local name rather than on the relation, and the
two maps share a logical source, so a query joining an object to its maker is answered by
one scan of one table with two subject-key columns — no `rml:joinCondition` needed, and
none is authored.

The iterator below is `public.objects`, schema-qualified. `met` is the DATABASE name, not
a schema — `met.objects` resolves to no relation, and a relation reference that resolves
to nothing registers with an empty Arrow schema, which the federation engine refuses.

> **The predicates are not the columns.** A `###` heading is a term in
> [the ontology](ontology), and the `::reference` line under it is the column that term is
> lifted from. Where the two happen to agree — `country`, `department` — that is the
> column being well named, not the model deferring to it. Three headings carry an explicit
> `::predicate` because the right term is already published elsewhere: a title is
> `dct:title`, a maker is `dct:creator`, and an external identifier asserting identity is
> `owl:sameAs`.

**What this mapping cannot say.** Three limits are structural, already filed, and stated here rather than worked around.

**Pipe-packed cells are one literal, not many values.**
`classification`, `tags`, `artist_display_name`, `artist_nationality`, `artist_ulan_url`
and `artist_wikidata_url` all pack multiple values into a single cell with a `|`
separator. There is no construct in the RML subset this engine admits that fans one cell
out into N triples — `svf:split_part` extracts exactly one field at a fixed position and
cannot iterate — so every packed column here yields the packed string verbatim. That is
[SemviaIO/SemviaIO#5719](https://github.com/SemviaIO/SemviaIO/issues/5719). The
consequence a query author must hold: an equality filter on `met:classification` matches
only objects whose cell holds that term ALONE, and a substring filter over-matches. Both
are wrong, in opposite directions.

**Only the first credited maker is reachable.** `artist_display_name` packs co-attributed
makers, but `constituent_id` never does — it is unpacked on all 484,956 rows. So there is
no per-artist key to split against even if splitting existed, and the mapping mints
exactly one `met:Artist` per row. An object credited to `"Paulding Farnham|Tiffany & Co."`
reaches one of the two through `dct:creator`, and the second maker is invisible to the
graph. Same issue, #5719.

**The subject tags cannot become concept links.** `tags` is the sharpest loss in the whole
relation, because the source already carries what a consumer wants: `tags_aat_url` and
`tags_wikidata_url` hold a Getty AAT IRI and a Wikidata IRI per tag, positionally aligned
with `tags` and packed the same way. Reaching them needs the per-cell split above AND a
three-way positional zip that aligns index *i* of three independently packed columns —
a primitive this engine does not have and #5719 does not promise. So `met:tagsRaw` carries
the packed string, the two IRI columns are not mapped at all, and the package claims
nothing more than that.

**Two decisions that look like omissions.**

**`is_highlight` stays a string.** The column holds `"True"` and `"False"`, which are not
in the lexical space of `xsd:boolean` (`"true"` and `"false"` are). Typed-literal minting
stamps the datatype without validating the lexical form
([SemviaIO/SemviaIO#5720](https://github.com/SemviaIO/SemviaIO/issues/5720)), so declaring
`::datatype boolean` here would silently create 484,956 malformed literals rather than
refusing. `met:isHighlight` is an `xsd:string` and a query asks for `"True"`.

**`owl:sameAs` is stored and never followed.** `met:Object`'s `owl:sameAs` carries the
object's Wikidata entity, and the artist map carries two more. All three are outbound
external identifiers — values to project, not edges to walk. `owl:sameAs` is not in the
production rule set and nothing rewrites a query across it, so a pattern that tries to
reach an object's facts THROUGH its Wikidata IRI returns no rows and no error. Cross-source
identity in this stack is settled by the IRI templates inside the RML — two mappings that
mint the same subject IRI are talking about the same thing — and never by an inference
asked afterwards.

## Objects

One catalogued work per row, subject-keyed on `object_id`, the Met's own accession-ordered
object number. Present and unique on every row, so the map mints a subject for all 484,956.

::[source](http://w3id.org/rml/source) [met-source](connection#met-source)
::[referenceFormulation](http://w3id.org/rml/referenceFormulation) [SQL2008Table](http://w3id.org/rml/SQL2008Table)
::[iterator](http://w3id.org/rml/iterator) "public.objects"
::[template](http://w3id.org/rml/template) "https://data.metmuseum.org/object/{object_id}"
::[class](http://w3id.org/rml/class) [Object](ontology#Object)

### title

The work's title, as the Met publishes it. A clean single-valued column — never packed,
which is what makes it the one text column a substring filter can be trusted on.

::[predicate](http://w3id.org/rml/predicate) [title](http://purl.org/dc/terms/title)
::[reference](http://w3id.org/rml/reference) "title"

### objectName

The Met's descriptive name for the kind of thing this is — `Vase`, `Photograph`, `Dress`.
Not the title: an untitled photograph has an object name and nothing else.

::[reference](http://w3id.org/rml/reference) "object_name"

### classification

The collection-management classification. Packed on a large minority of rows, so the value
is a cell rather than a term — see the limits above before filtering on it.

::[reference](http://w3id.org/rml/reference) "classification"

### culture

The culture or people the work is attributed to. 7,313 distinct values, open free text.

::[reference](http://w3id.org/rml/reference) "culture"

### medium

Materials and technique, written for a wall label. 65,908 distinct values — prose.

::[reference](http://w3id.org/rml/reference) "medium"

### country

The modern country the work is geographically attributed to. Recorded on 76,007 rows and
absent on the rest, which is why no shape requires it and every query reads it optionally.

It is also a packed column on 1,118 of those rows, so the three renderings of "how many
Egyptian objects" disagree: 31,296 for `= "Egypt"`, 31,385 once pipe-delimited tokens are
counted, and 33,461 for a substring match that also catches `Egyptian Sudan`. Only the
first is expressible here, and it is the number the corpus asserts. See #5719.

::[reference](http://w3id.org/rml/reference) "country"

### isHighlight

`"True"` or `"False"`, as the source spells them. Deliberately an `xsd:string` — see the
note above on #5720.

::[reference](http://w3id.org/rml/reference) "is_highlight"

### galleryNumber

The gallery the object is currently on view in. Absent on 435,415 of 484,956 rows, and the
absence is the signal: no gallery means in storage. Every query over it belongs under
`OPTIONAL`, and unbound is an answer rather than a gap.

::[reference](http://w3id.org/rml/reference) "gallery_number"

### tagsRaw

The Met's subject tags as one pipe-packed literal. `Raw` is the honest part of the name —
see the limits above.

::[reference](http://w3id.org/rml/reference) "tags"

### creationDateEarliest

The earliest year of the creation range, minted as an `xsd:integer`. Negative for BCE:
`"-332"` is a faithful integer literal for 332 BCE and needs no repair. The source column
is `text` like every other one, so this datatype is a minting decision the ontology makes
rather than one the database carries — and the cost of that is real: a numeric comparison
on this path cannot be delegated to SQL, because pushdown requires the delivered column to
already be non-textual. A range query here is answered above the scan, not inside it.

::[reference](http://w3id.org/rml/reference) "object_begin_date"
::[datatype](http://w3id.org/rml/datatype) [integer](http://www.w3.org/2001/XMLSchema#integer)

### creationDateLatest

The latest year of the creation range. Equal to the earliest for a work dated to a single
year; the pair is a closed range, not a point.

::[reference](http://w3id.org/rml/reference) "object_end_date"
::[datatype](http://w3id.org/rml/datatype) [integer](http://www.w3.org/2001/XMLSchema#integer)

### department

The curatorial department, as a concept IRI rather than a literal. The template mints the
same IRI the nineteen concepts in [departments](departments) are asserted under, so the
two meet on identity without either deriving the other — and because template minting is
invertible, a query naming one department IRI as a constant inverts back to an equality on
the source cell and pushes into SQL.

The percent-encoding is not cosmetic: `Egyptian Art` mints
`https://data.metmuseum.org/department/Egyptian%20Art`, and the concepts are spelled the
same way for the same reason.

::[template](http://w3id.org/rml/template) "https://data.metmuseum.org/department/{department}"
::[termType](http://w3id.org/rml/termType) [IRI](http://w3id.org/rml/IRI)

### creator

The primary recorded maker, as the artist IRI the sibling [artists](artists) map mints from
the same column. A row with no `constituent_id` — 202,443 of them — mints no triple here,
which is RML's ordinary null semantics and is correct: the catalogue records no maker, so
the graph asserts none.

::[predicate](http://w3id.org/rml/predicate) [creator](http://purl.org/dc/terms/creator)
::[template](http://w3id.org/rml/template) "https://data.metmuseum.org/artist/{constituent_id}"
::[termType](http://w3id.org/rml/termType) [IRI](http://w3id.org/rml/IRI)

### wikidata identity

The object's own Wikidata entity, where the Met records one — 69,154 of 484,956 rows, 14%
coverage. The column already holds a full absolute URL, so it is referenced directly with
an IRI term type rather than re-templated, and it is the one external-identifier column in
the relation that is never packed (0 rows), which is what makes the direct reference safe.

Projected, never traversed — see the note above.

::[predicate](http://w3id.org/rml/predicate) [sameAs](http://www.w3.org/2002/07/owl#sameAs)
::[reference](http://w3id.org/rml/reference) "object_wikidata_url"
::[termType](http://w3id.org/rml/termType) [IRI](http://w3id.org/rml/IRI)
