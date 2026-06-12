# Diagram Gate

A relative local image (CRITICAL regression: must render, not 404):

![a red box](./diagram-assets/red-box.png)

## First diagram

```mermaid title="Gate pipeline"
graph LR
  GATEALPHA[gatealphanode] --> GATEBETA{gatebetanode}
  GATEBETA -->|yes| GATEGAMMA[gategammanode]
```

## Second diagram (id-collision check)

```mermaid
graph TD
  GATEDELTA[gatedeltanode] --> GATEEPSILON[gateepsilonnode]
```

## Kept as source

```mermaid render=false
graph LR
  RAWKEPT --> ASCODE
```

## Deliberately broken

```mermaid
graph LR
  A -->
  (((
```

Done.
