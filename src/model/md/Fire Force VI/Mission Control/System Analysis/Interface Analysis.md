---
ontology: https://fireforce6.github.io/mission-control/bundle
---

# Interface Analysis

How the Fire Force VI components are wired through their ports, and how mass is spread over the three segments.

## Connection Direction

A connection between two peer components should go from an Out port to an In port. A connection into a child component goes from In to In, and a connection out of a child goes from Out to Out.

```table
PREFIX oml: <http://opencaesar.io/oml#>
PREFIX base: <https://www.modelware.io/sierra/base#>
PREFIX component: <https://www.modelware.io/sierra/component#>

SELECT ?Connection ?From ?FromDirection ?To ?ToDirection ?Kind ?Result
WHERE {
  ?Connection a component:Connection ;
              oml:hasSource ?From ;
              oml:hasTarget ?To .
  ?fromComponent (component:hasPort|^component:portOf) ?From .
  ?toComponent (component:hasPort|^component:portOf) ?To .
  OPTIONAL { ?From component:direction ?FromDirection }
  OPTIONAL { ?To component:direction ?ToDirection }
  BIND(IF(EXISTS { ?fromComponent (base:contains|^base:isContainedBy)+ ?toComponent }, "into child",
       IF(EXISTS { ?toComponent (base:contains|^base:isContainedBy)+ ?fromComponent }, "out of child", "between peers")) AS ?Kind)
  BIND(STR(?FromDirection) AS ?f)
  BIND(STR(?ToDirection) AS ?t)
  BIND(IF((?Kind = "between peers" && ?f = "Out" && ?t = "In") ||
          (?Kind = "into child" && ?f = "In" && ?t = "In") ||
          (?Kind = "out of child" && ?f = "Out" && ?t = "Out"), "conforms", "violates") AS ?Result)
}
ORDER BY DESC(?Result) ?Connection
```

All seven connections conform. A connection drawn the wrong way between two peers would show up here as "violates".

## Partly Wired Components

Components that already have a connected port but still have one that is not connected.

```table
PREFIX component: <https://www.modelware.io/sierra/component#>

SELECT ?Component ?UnconnectedPort
WHERE {
  ?Component (component:hasPort|^component:portOf) ?UnconnectedPort .
  FILTER NOT EXISTS { ?UnconnectedPort (component:connectedTo|^component:connectedTo) ?x }
  FILTER EXISTS {
    ?Component (component:hasPort|^component:portOf) ?p .
    ?p (component:connectedTo|^component:connectedTo) ?y .
  }
}
ORDER BY ?Component
```

## Are the Components Wired to Each Other?

**Question.** Which components have ports that no connection uses, and which items does each component actually exchange?

**Evidence.**

```compose
template: https://www.modelware.io/sierra/system-analysis/interfaces
```

**Interpretation.** The data storage, the on-board computer and the payload processor each own a port, but nothing is connected to any of them, so the electronics segment is not yet wired to the rest of the satellite. Together with the propulsion segment's command input above, that leaves no path for commands to reach propulsion. The emergency communication unit and its transceiver exchange commands and telemetry with each other but have no power connection, and the graph shows them as an island apart from FireSat: the unit is modeled, but how it is powered and how it reaches the platform are not. These are gaps in what has been recorded, not in the vocabulary, since the component vocabulary can already state every missing connection.

## Mass by Segment

```python
result = await query("""
PREFIX oml: <http://opencaesar.io/oml#>
PREFIX base: <https://www.modelware.io/sierra/base#>
PREFIX component: <https://www.modelware.io/sierra/component#>
SELECT DISTINCT ?segment ?part ?value ?unit
WHERE {
  ?root (base:contains|^base:isContainedBy) ?segment .
  FILTER NOT EXISTS { ?top (base:contains|^base:isContainedBy) ?root }
  FILTER NOT EXISTS {
    ?root (base:contains|^base:isContainedBy) ?mid .
    ?mid (base:contains|^base:isContainedBy) ?segment .
  }
  ?segment (base:contains|^base:isContainedBy)+ ?part .
  ?part component:mass ?q .
  ?q oml:value ?value .
  OPTIONAL { ?q oml:unit ?unit }
}
""")

to_kg = {"kg": 1.0, "g": 0.001}
totals, raw, parts = {}, {}, {}
converted = 0
for r in result["rows"]:
    segment = r["segment"].split("#")[-1]
    unit = (r.get("unit") or "kg").rsplit("/", 1)[-1]
    value = float(r["value"])
    totals[segment] = totals.get(segment, 0) + value * to_kg[unit]
    raw[segment] = raw.get(segment, 0) + value
    parts[segment] = parts.get(segment, 0) + 1
    if unit != "kg":
        converted += 1

total = sum(totals.values())
rows = "".join(
    f"<tr><td>{s}</td><td>{parts[s]}</td><td>{totals[s]:.2f}</td>"
    f"<td>{(100 * totals[s] / total if total else 0):.1f}%</td></tr>"
    for s in sorted(totals, key=totals.get, reverse=True)
)
display(
    '<table class="oml-md-table">'
    '<thead><tr><th>Segment</th><th>Parts</th><th>Mass (kg)</th><th>Share</th></tr></thead>'
    f'<tbody>{rows}</tbody></table>'
    f'<p>Total {total:.2f} kg over {sum(parts.values())} parts. '
    f'{converted} masses are recorded in grams and were converted. '
    f'Adding the raw values without converting gives {sum(raw.values()):.2f}.</p>'
)
```

Three payload parts are recorded in grams. Adding the recorded values as they are would put the payload above a tonne, almost the whole satellite, so the masses are converted in the script before they are added.
