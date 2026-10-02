---
template:
  id: https://www.modelware.io/sierra/system-analysis/interfaces
  name: "Interface Analysis"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Interface Analysis

Check that component ports are wired, and see which items each component exchanges.

## Unconnected Components

Components that own ports, none of which is connected.

```table
PREFIX component: <https://www.modelware.io/sierra/component#>

SELECT ?Component ?Port ?Direction
WHERE {
  ?Component (component:hasPort|^component:portOf) ?Port .
  OPTIONAL { ?Port component:direction ?Direction }
  FILTER NOT EXISTS {
    ?Component (component:hasPort|^component:portOf) ?p .
    ?p (component:connectedTo|^component:connectedTo) ?other .
  }
}
ORDER BY ?Component ?Port
```

## Items per Component

The number of connections at a component's ports that transfer each item.

```matrix
---
rowColumnLabel: Component / Item
stylesheet:
  - selector: cell [Number(value) > 0]
    style:
      background-color: lightgreen
---
PREFIX oml: <http://opencaesar.io/oml#>
PREFIX component: <https://www.modelware.io/sierra/component#>

SELECT ?row ?column (COALESCE(?n, 0) AS ?value)
WHERE {
  { SELECT DISTINCT ?row WHERE { ?row (component:hasPort|^component:portOf) ?p } }
  { SELECT DISTINCT ?column WHERE { ?c a component:Connection ; component:transfers ?column } }
  OPTIONAL {
    SELECT ?row ?column (COUNT(DISTINCT ?c) AS ?n)
    WHERE {
      ?row (component:hasPort|^component:portOf) ?p .
      ?c a component:Connection ;
         component:transfers ?column ;
         (oml:hasSource|oml:hasTarget) ?p .
    }
    GROUP BY ?row ?column
  }
}
ORDER BY ?row ?column
```

## Interface Graph

Components linked by the items their connections transfer.

```graph
---
layout:
  mode: force
  fit: true
group:
  byPredicate: true
---
PREFIX oml: <http://opencaesar.io/oml#>
PREFIX component: <https://www.modelware.io/sierra/component#>

CONSTRUCT {
  ?from ?item ?to .
}
WHERE {
  ?c a component:Connection ;
     oml:hasSource ?fromPort ;
     oml:hasTarget ?toPort ;
     component:transfers ?item .
  ?from (component:hasPort|^component:portOf) ?fromPort .
  ?to (component:hasPort|^component:portOf) ?toPort .
}
```
