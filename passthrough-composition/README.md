# Passthrough Compositions

Sometimes, you just want to expose the underlying api directly. Copying the schema from the MR to the XRD is kind of annoying. While code generation could possibly make this friendlier, this repo demonstrates another possibility.

1. in `definition.yaml`, add a free-form config block to capture direct MR properties.
```yaml
openAPIV3Schema:
  type: object
  properties:
    spec:
      type: object
      properties: 
        forProvider: 
          type: object 
          additionalProperties: 
            x-kubernetes-preserve-unknown-fields: true
```

2. in `composition.yaml` patch the config block over the MR properties, merging with existing config.
```yaml
patches:
- fromFieldPath: spec.forProvider
  toFieldPath: spec.forProvider
  policy:
    mergeOptions:
      keepMapValues: true
```

### Pros
- Forward compatible: new fields don't require re-creating the XRD
- Simple to build and maintain

### Cons
- No schema validation