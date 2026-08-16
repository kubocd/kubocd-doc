# Alternate KuboCD Schema Format

In previous examples, we defined `schema.parameters` and `schema.context` using standard OpenAPI/JSON schema. While fully supported, this syntax can be verbose.

KuboCD provides a concise schema syntax specifically designed to simplify this definition.

Comparing the `schema` section of the `podinfo` Package:

**Standard OpenAPI:**

```yaml
schema:
  parameters:
    $schema: http://json-schema.org/schema#
    type: object
    additionalProperties: false
    properties:
      host: { type: string }
    required:
      - host
  context:
    $schema: http://json-schema.org/schema#
    additionalProperties: true
    type: object
    properties:
      ingress:
        type: object
        additionalProperties: true
        properties:
          className: { type: string }
          domain: { type: string }
        required:
          - domain
          - className
    required:
      - ingress
```

**KuboCD Simplified Format:**

```yaml
schema:
  parameters:
    properties:
      host: { type: string, required: true }
  context:
    properties:
      ingress:
        required: true
        properties:
          className: { type: string, required: true }
          domain: { type: string, required: true }
```

The simplified format is significantly more compact.

!!! info
    This format is **NOT** standard OpenAPI.

KuboCD distinguishes between the two formats by checking for the presence of the `$schema:` key.

**Key Differences:**

- **No `$schema:` key**.
- **Required fields** are defined as boolean attributes on the node (`required: true`) rather than in a separate list.
- **`additionalProperties: false`** is automatically implied for all objects in `schema.parameters`.
- **`additionalProperties: true`** is automatically implied for all objects in `schema.context`.
- **`type: object`** is inferred if `properties` is present.
- **`type: array`** is inferred if `items` is present.
- Other attributes (`default`, `enum`, `pattern`, etc.) remain unchanged.

When building a Package with this simplified format, KuboCD automatically converts it to standard OpenAPI/JSON schema. This normalized version is stored in the artifact and is viewable via `kubocd dump package` or `kubocd render`.

!!! warning
    If you choose to use the standard form, remember to set `additionalProperties: false` on `schema.parameters` objects. Without it, typos in variable names will be ignored, leading to potential deployment issues.

For the remainder of this manual, we will use the simplified KuboCD schema format.
