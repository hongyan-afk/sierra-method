---
template:
  id: https://www.modelware.io/sierra/system-analysis/ports
  name: "Ports"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Ports

Define component ports, identify the component each port belongs to, and specify whether the port is an input or output.

## Current Port Ownership

```table
---
columns:
  Port: { label: "Port" }
  Component: { label: "Component" }
---
PREFIX component: <https://www.modelware.io/sierra/component#>

SELECT DISTINCT ?Port ?Component
WHERE {
    ?Port a component:Port .

    {
        ?Component component:hasPort ?Port .
    }
    UNION
    {
        ?Port component:portOf ?Component .
    }
}
ORDER BY ?Port
```

## Port Editor

```table-editor
---
columns: { this: { label: "Port" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <https://www.modelware.io/sierra/base#> .
@prefix component: <https://www.modelware.io/sierra/component#> .

component:PortShape
    a sh:NodeShape ;
    sh:targetClass component:Port ;

    sh:sparql [
        sh:message "Every port must belong to exactly one component." ;
        sh:select """
            PREFIX component: <https://www.modelware.io/sierra/component#>

            SELECT $this
            WHERE {
                FILTER NOT EXISTS {
                    {
                        ?component component:hasPort $this .
                    }
                    UNION
                    {
                        $this component:portOf ?component .
                    }
                }
            }
        """ ;
    ] ;

    sh:sparql [
        sh:message "Every port must belong to exactly one component." ;
        sh:select """
            PREFIX component: <https://www.modelware.io/sierra/component#>

            SELECT $this
            WHERE {
                {
                    {
                        ?component1 component:hasPort $this .
                    }
                    UNION
                    {
                        $this component:portOf ?component1 .
                    }
                }

                {
                    {
                        ?component2 component:hasPort $this .
                    }
                    UNION
                    {
                        $this component:portOf ?component2 .
                    }
                }

                FILTER (?component1 != ?component2)
            }
        """ ;
    ] ;

    sh:property [
        sh:path component:portOf ;
        sh:name "Component" ;
        sh:class component:Component ;
        sh:maxCount 1 ;
    ] ;

    sh:property [
        sh:path component:direction ;
        sh:name "Direction" ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:in ( "In" "Out" ) ;
        sh:message "Every port must have exactly one direction: In or Out." ;
    ] ;

    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:maxCount 1 ;
    ] ;

    .
```