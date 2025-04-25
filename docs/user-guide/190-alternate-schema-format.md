# Alternate KuboCD Schema Format

In our previous examples, we defined `schema.parameters` and `schema.context` using a standard OpenAPI/JSON schema. 
While fully supported, it can be quite verbose.

KuboCD also supports a more concise schema syntax, specifically designed for this use case.

Here is one of our `podinfo` `Package` example from a previous chapter (Only the `schema` section) 

???+ abstract "podinfo-p02.yaml"

    ```
    apiVersion: v1alpha1
    ........
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
    modules:
    .........
    ```

Here’s an identical version of this `Package` using the simplified KuboCD schema format:


???+ abstract "podinfo-p02-alt.yaml"

    ```
    apiVersion: v1alpha1
    ......
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
    modules:
    .......
    ```

As you can see, the format is significantly more compact.

!!! info
    This schema format is **NOT** standard OpenAPI

The presence or absence of the `$schema:` key is what KuboCD uses to distinguish between standard and KuboCD schema formats.

The difference are the following:

- No `$schema:` key for the KuboCD format
- The `required` fields are not in an array anymore, but are now an attribute of the node.
- The flag `additionalProperties: false` is set on all object and sub-object of a `parameters` schema.
- The flag `additionalProperties: true` is set on all object and sub-object of a `context` schema.
- the `type: object` is now optional. It is deduced from the presence of the `properties` attribute.
- the `type: array` is now optional. It is deduced from the presence of the `items` attribute.
- All others attributes (`default`, `enums`, `pattern`, ....) are left unchanged.

When a `Package` is build using this form, both schemas are converted to their standard OpenAPI/JSON and stored in this last form. 

This normalized version will be accessible using `kubocd dump package...` or `kubocd render....` CLI tools.

!!! warning

    If you still use the standard form, setting `additionalProperties: false` on all objects of `schema.parameters` is important. 
    If not, any typo in a variable name will not be trapped and may lead tricky errors.

For the remaining of this manuel, we will use the KuboCD schema. 

