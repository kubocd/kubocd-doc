# Template functions

Package templates (see [Templating](./500-package.md#templating)) and `Release.spec.parameters` are Go templates.
Starting with v0.4.0, they can call a fixed list of functions, and nothing else.

## The rules

1. A template only reads the data passed to it: maps, lists, strings, numbers and booleans. It reads no file, makes no
   network call, reads no environment variable and queries nothing in the cluster. The only exception is
   [`resolveDNS`](#resolvedns), in package templates, when the platform operator allows it.
2. A manifest is read as written. Neither the CLI nor the controller replaces a `${...}` sequence in a package or
   `Release` manifest, when packaging or when loading. See
   [Environment variables in manifests](#environment-variables-in-manifests).
3. For a given KuboCD version, the same data renders the same text, character for character: no random value, no
   clock, no order that changes. The only exception is `resolveDNS` when allowed, whose result also depends on the DNS
   answers. A map with two keys that are the same once converted to strings, such as `1` and `"1"`, is an error, in a
   manifest as in a rendered text.
4. A template execution changes neither its data nor the data seen by another execution, of the same `Release` or of
   another one.
5. A render error says where and why, without quoting the data or the rendered text, in the `Release` status, its
   events and the controller log. The messages of the KuboCD functions, and those written with `fail` and `required`,
   are kept as written. The errors of the steps that use the rendered values, from building the HelmReleases to the
   requests sent to the API server, follow the same rule. See [Errors](#errors).

### Where the rules stop

These rules apply to the templates KuboCD renders. The values they produce go to the Helm charts of the package, whose
own templates follow the rules of Helm and use its functions. Flux renders the charts in the cluster, and
`kubocd render` runs `helm template` on them after the KuboCD rendering. The `Release` status and events show the
messages of its HelmReleases as Flux and Helm write them.

## The data model

Package templates get every root element below. `Release.spec.parameters` gets some of them.

| Root element | Content | In `Release.spec.parameters` |
| :--- | :--- | :--- |
| `.Release` | The `Release` object. | Yes |
| `.Context` | The merged context. See [The Context Resource](../user-guide/160-the-context.md). | Yes |
| `.Parameters` | The release parameters, merged with the package defaults. | No |
| `.ImageRedirects` | The image redirects of the `Config` resources, read by [`imageRedirect`](#imageredirect). | Yes |
| `.Inputs` | For each input of the package, by alias: the values of the connection it resolves to. | No |
| `.InputLists` | For each input allowing several connections, by alias: the values of every ready matching connection, highest `priority` first. | No |

`.Inputs` and `.InputLists` belong to the EXPERIMENTAL connections feature. They are not set yet when the `inputs` of a
package are rendered.

The data is plain data: maps, lists, strings, numbers and booleans. A number read from a manifest, such as a parameter,
can be a floating point number: convert it with `int` where an integer is expected, as with
[`slice`](#go-texttemplate-built-ins).

Each template execution works on its own copy of the data. `set`, which changes a dict, and `sortAlpha`, which sorts a
list in place, only change this copy: no other template, and no other `Release` of the same package, sees the change.

## Available functions

Package templates and `Release.spec.parameters` get the same functions, except [`resolveDNS`](#resolvedns), which only
package templates get.

### Sprig functions

These 45 functions come from [Sprig](https://masterminds.github.io/sprig/) v3.3.0 and behave as documented there:

| Group | Functions |
| :--- | :--- |
| Strings | `indent`, `lower`, `nindent`, `nospace`, `quote`, `regexReplaceAll`, `replace`, `split`, `splitList`, `title`, `trim`, `trimAll`, `trimPrefix`, `trimSuffix`, `trunc`, `upper` |
| Lists | `append`, `compact`, `concat`, `first`, `has`, `join`, `last`, `list`, `prepend`, `sortAlpha`, `uniq` |
| Dictionaries | `dict`, `dig`, `get`, `hasKey`, `omit`, `set` |
| Logic, defaults and types | `coalesce`, `default`, `empty`, `kindIs`, `ternary` |
| Conversions and math | `div`, `divf`, `float64`, `int`, `min`, `mulf` |
| Encoding | `sha256sum` |

### KuboCD functions

KuboCD defines these functions. `fail`, `fromJson`, `keys`, `toJson`, `until` and `urlParse` take the place of the
Sprig functions of the same name, and behave as described here.

| Function | Description |
| :--- | :--- |
| [`fail`](#fail) | Fails the rendering with a message. |
| `fromJson` | Parses a JSON string into a map. On error, the map holds the error text under `Error`. |
| `fromYaml` | Parses a YAML string into a map. On error, the map holds the error text under `Error`. For a map key that cannot be converted to a string, such as `null`, this text is `fromYaml: the document has a map key that cannot be converted to a string, such as null`. A document holding a map with two keys that are the same once converted to strings, such as `1` and `"1"`, fails the rendering instead. |
| [`imageRedirect`](#imageredirect) | Applies the image redirects of the `Config` resources to an image reference. |
| [`keys`](#keys) | Returns the keys of one or more dicts, sorted. A key present in several dicts appears once per dict. |
| [`required`](#required) | Fails the rendering when a value is missing. |
| `toJson` | Serializes a value to JSON. On error, the result holds the error text. |
| `toYaml` | Serializes a value to YAML, without the trailing newline. On error, the result holds the error text. |
| `until` | Returns the list of the integers from 0 up to a count, the count excluded, as the Sprig function does: `until 3` gives `[0 1 2]` and `until -3` gives `[0 -1 -2]`. It builds at most 10000 items either way. A larger count fails the rendering with `until: the count is beyond the limit of 10000`. See [Limits](#limits). |
| `urlParse` | Parses a URL into a dict of its parts, as the Sprig function does. An invalid URL fails with `urlParse: invalid URL`, without the URL. |
| [`resolveDNS`](#resolvedns) | Package templates only, disabled by default. Returns the first IP address of a host name. |

### Go text/template built-ins

These are part of Go templates and always available: `and`, `call`, `eq`, `ge`, `gt`, `html`, `index`, `js`, `le`,
`len`, `lt`, `ne`, `not`, `or`, `print`, `printf`, `println`, `slice` and `urlquery`, plus the actions `if`, `else`,
`range`, `with`, `define`, `template`, `block`, `break`, `continue` and `end`. See the
[text/template documentation](https://pkg.go.dev/text/template).

- `printf` is `fmt.Sprintf`, except that it rejects the `%p` verb, which prints a memory address that changes from one
  rendering to the next. The rendering fails with an error ending in `printf: the %p verb is not allowed`.
- `slice` takes integer indexes only. Convert a number from the data with `int`:

    ```yaml
    zones: {{ slice .Parameters.zones 0 (int .Parameters.replicas) | toJson }}
    ```

### keys

`keys` returns the keys sorted. Pass the list through `toJson` to write it as YAML:

```yaml
values: |
  labelNames: {{ keys .Parameters.labels | toJson }}
```

With the labels `tier`, `app` and `team`, this renders `labelNames: ["app","team","tier"]`. Without `toJson`, the list
prints as `[app team tier]`, which YAML reads as a list holding the single string `app team tier`.

### fail

`fail MESSAGE` stops the rendering with `MESSAGE`, for instance to reject a combination of parameters:

```yaml
values: |
  {{- if and .Parameters.externalDatabase .Parameters.embeddedDatabase }}
  {{- fail "set 'externalDatabase' or 'embeddedDatabase', not both" }}
  {{- end }}
```

With both parameters set, the rendering fails with an error ending in
`line 2, column 5: error calling fail: set 'externalDatabase' or 'embeddedDatabase', not both`.

### required

`required MESSAGE VALUE` returns `VALUE`, or fails the rendering with `MESSAGE` when `VALUE` is missing or an empty
string.

```yaml
values: |
  auth:
    adminPassword: {{ required "parameter 'adminPassword' is required" .Parameters.adminPassword | quote }}
```

Without `adminPassword`, the rendering fails with an error ending in
`line 2, column 21: error calling required: parameter 'adminPassword' is required`.

### imageRedirect

`imageRedirect IMAGE ROOT` applies the image redirects defined in the `Config` resources to `IMAGE`, and returns its
parts. `ROOT` is the root of the data model: `.` at the top level of a template, `$` inside a `range` or a `with`.

```yaml
values: |
  image:
    {{- $image := imageRedirect "quay.io/jetstack/cert-manager-controller:v1.17.0" . }}
    repository: {{ $image.repository }}
    tag: {{ $image.tag }}
    {{- with $image.imagePullPolicy }}
    pullPolicy: {{ . }}
    {{- end }}
```

The result is a map:

| Key | Content |
| :--- | :--- |
| `url` | The image reference after redirection. |
| `repository` | `url` without the tag. |
| `tag` | The tag, `latest` if there is none. |
| `registry` | The first part of `repository` (the registry host). |
| `baseRepository` | `repository` without the registry. |
| `imagePullPolicy` | The pull policy of the matching redirect, or an empty string. |
| `imagePullSecrets` | The pull secrets of the matching redirect, or `null`. |

The redirects come from `Config.spec.imageRedirects`, described in [imageRedirect](./530-config.md#imageredirect).
`Config` resources are taken in name order, and the first redirect whose `oldPrefix` matches the image applies:

```yaml
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Config
metadata:
  name: conf01
  namespace: kubocd
spec:
  imageRedirects:
    - oldPrefix: quay.io/
      newPrefix: registry.internal:5000/quay/
      imagePullSecrets:
        - internal-registry
```

With this `Config`, the example above renders `repository: registry.internal:5000/quay/jetstack/cert-manager-controller`.

### resolveDNS

`resolveDNS HOST` returns the first IP address of `HOST`, resolved from the controller pod, or from the machine running
`kubocd render`.

- Package templates only. In `Release.spec.parameters`, it does not exist, even when allowed:
  `function "resolveDNS" not defined`.
- Disabled by default. `kubocd package` accepts a package calling it, but rendering the package fails, in the `Release`
  status as in `kubocd render`, with an error ending in:

    ```
    error calling resolveDNS: resolveDNS is disabled by platform policy
    ```

- The platform operator allows it for every package of the cluster, with the `controller.allowDNSLookups` value of the
  `kubocd-ctrl` Helm chart, which sets the `--allowDNSLookups` option of the controller:

    ```yaml
    controller:
      allowDNSLookups: true
    ```

    `kubocd render` takes the same `--allowDNSLookups` flag.

- Once allowed, the result also depends on the DNS answers: the same data can render another address after a DNS
  change. When a name has several addresses, the first one returned by the resolver is used, which can be an IPv6
  address.

A package can also take the address from its data instead, see [Upgrading from v0.3.x](#upgrading-from-v03x).

## Limits

Each execution of a template, such as the `values` of a module or `Release.spec.parameters`, renders at most 4 MiB
(4194304 bytes) of text. Past that, the rendering fails with `the rendered text is larger than 4194304 bytes`.
[`until`](#kubocd-functions) builds at most 10000 items, and fails past that with
`until: the count is beyond the limit of 10000`.

The controller schedules no retry on these errors, as the same data fails the same way: the `Release` stays in
`ERROR` until a change of what it reads reconciles it again. `kubocd render` fails the same way.

## Environment variables in manifests

A `${...}` sequence in a package or `Release` manifest stays as written. Neither `kubocd package` nor the controller,
nor any other command reading a manifest, replaces it with an environment variable. A CI pipeline that needs
substitutions runs them before `kubocd package`, for instance with `envsubst`:

```bash
envsubst '${BUCKET}' < package.tmpl.yaml > package.yaml
kubocd package package.yaml
```

## Errors

Where an error shows up:

- `kubocd package` parses the templates of the package, without running them, and fails on a function outside the
  list, naming the module and the field:

    ```
    ERROR: module 'main': could not parse 'values' template: template: :2: function "now" not defined
    ```

    The `templateHeader` of the package is placed before each of its templates. An error there names it:
    `in templateHeader: template: :1: function "now" not defined`. A `templateHeader` without a final newline, such as
    one written with `|-`, shares its last line with the first line of each template, and a parse error on that line
    names both places:
    `in templateHeader or on the first line of the template: template: :2: function "now" not defined`.

- A manifest holding a map with two keys that are the same once converted to strings, or a map key that cannot be
  converted to a string, such as `null`, is rejected when the CLI or the controller reads it, with a message that names
  neither the keys nor the values. `kubocd package` gives:

    ```
    ERROR: the YAML document has a map whose keys are the same once converted to strings, such as 1 and "1", or true and "true"
    ERROR: the YAML document has a map key that cannot be converted to a string, such as null
    ```

- The controller parses a package when it loads it. A package built with an older CLI, calling a function outside the
  list, puts the `Release` in error:
  `error while loading package from image: module 'main': could not parse 'values' template: ...`. A package whose
  manifest has such map keys puts it in error too:
  `error while parsing package original.yaml file: the YAML document has a map whose keys are the same ...`.
- `Release.spec.parameters` is parsed when the `Release` is reconciled. The `Release` goes to `ERROR` with
  `could not create template from parameters: template: :1: function "now" not defined`.
- The default of a `connectionRef` parameter (EXPERIMENTAL connections feature) is a template that `kubocd package`
  does not parse. It is parsed and rendered for each `Release` that does not set the parameter, with the functions and
  the data of `Release.spec.parameters` and without the `templateHeader`, so an error there shows up in the `Release`
  status. The value that `Release.spec.parameters` gives to the parameter is a connection name, never rendered a second
  time.
- The other errors happen while rendering: `fail`, `required`, `resolveDNS` disabled, a function or an action that
  fails on the data, a [limit](#limits) reached, a rendered text that is not valid YAML, or a rendered map with two
  keys that are the same once converted to strings, or with a key that cannot be converted, such as `null`. They show
  up in the `Release` status, its events and the controller log, and in the output of `kubocd render`.
- The errors of the steps that use the rendered values follow the same rule. See
  [After the rendering](#after-the-rendering).

What a message holds:

- The template, the place and the reason. A parse error keeps the message of Go text/template, which gives the line
  and can only quote the template. This line counts from the first line of the `templateHeader`, when the package has
  one. An execution error gives the line and the column of the failing action, counted from the start of the field, or
  from the start of the `templateHeader` when the action is there, which the message then names:
  `in templateHeader, line 1, column 14`. A rendered text that is not valid YAML is located by its line in this text,
  which begins with the output of the `templateHeader`.
- Never the data, nor the rendered text. The messages of Go text/template and of the Sprig functions, which can print
  the value at fault, give way to a category, such as `error calling regexReplaceAll` or
  `range over a value that is not a list or a map`, followed by `message withheld, as it can quote the data`.
- The messages of the KuboCD functions, and those written with `fail` and `required`, as written. Write the messages of
  `fail` and `required` without secrets.
- For a schema validation error on the parameters or the context, the path of the value, made of the properties the
  schema declares and of list indexes. The path stops before a property the schema does not declare, whose name can
  come from the data.

Some messages, as the `Release` status shows them:

```
error on rendering: module 'main': could not render 'values' template: line 2, column 21: error calling required: parameter 'adminPassword' is required
error on rendering: module 'main': could not render 'values' template: line 1, column 21: range over a value that is not a list or a map, message withheld, as it can quote the data
error on rendering: module 'main': could not render 'values' template: the rendered text is not valid YAML (line 4)
error on rendering: module 'main': could not render 'values' template: the rendered text has a map whose keys are the same once converted to strings, such as 1 and "1", or true and "true"
error on rendering: module 'main': could not render 'values' template: the rendered text has a map key that cannot be converted to a string, such as null
error on rendering: module 'main': could not render 'values' template: the rendered text is larger than 4194304 bytes
error on rendering: module 'main': could not render 'values' template: line 2, column 13: error calling until: until: the count is beyond the limit of 10000
could not validate parameters: parameters schema validation error: (root): a property is not declared in the schema
```

With a `templateHeader`, as in this excerpt of a package:

```yaml
templateHeader: |
  {{- $host := required "parameter 'host' is required" .Parameters.host }}
modules:
  - name: main
    values: |
      ingress:
        host: {{ $host }}
        port: {{ required "parameter 'port' is required" .Parameters.port }}
```

- Without `host`, the `templateHeader` fails in the first template rendered, here `roles`, which the package does not
  set:
  `could not render 'roles' template: in templateHeader, line 1, column 14: error calling required: parameter 'host' is required`.
- Without `port`, the error is on line 3 of `values`, the line of `port`:
  `module 'main': could not render 'values' template: line 3, column 12: error calling required: parameter 'port' is required`.

A `templateHeader` without a final newline shares its last line with the first line of each template. An execution
error on that line names the `templateHeader` when the failing action is in it. Otherwise, the message gives line 1,
with a column counted from the start of the template.

`kubocd render` writes the data model to `model.yaml` before rendering the package, which helps to reproduce an error.

### After the rendering

The values that the templates of a `Release` render then go through more steps: the controller looks up the input
connections, builds the HelmReleases, checks the conditions, creates and updates the connections of the outputs, and
sends the requests these steps need to the API server. Their errors follow the rule of the render errors: they name
the module, the field, or the input, the condition or the output by its position, such as `input#1` or `output[0]`,
and a category, never a rendered value. `kubocd render` follows the same rule for the steps it runs.

- A request refused by the API server gives the reason of the answer and the fields at fault, without the message of
  the answer, which can quote the value refused. A field keeps its first two names, without index or map key, such as
  `spec.install` for `spec.install.strategy.name`. A request without an answer gives
  `the request to the API server failed, message withheld, as it can quote the object`. This holds for every request
  after the rendering, the update of the `Release` status and the deletion of its connections included.
- A refused lookup of the connections of an input names the input by its position, such as
  `input#1: could not get the Connection: ...` or `input#1: could not list the Connections: ...`.
- A `Release` waiting for a condition names the object it checks, but quotes neither the CEL expression nor the message
  of the CEL evaluation, which can quote the object. A `Release` waiting for a connection or a role names it, the
  connections of its outputs included.
- The values of a `Connection` or a `ClusterConnection` are checked against its contract by the rule of the
  parameters: the path of the value and a category, such as
  `validation error on values: (root): a property is not declared in the schema`. Values that do not decode give
  `unable to parse connection values: they are not a map with distinct keys`, and a missing contract gives
  `the Contract or ClusterContract named in spec.contract is missing`, or
  `the ClusterContract named in spec.contract is missing` for a `ClusterConnection`. When the API server refuses to
  read the contract or to update the status of the connection, the message gives the reason and the fields of the
  answer only. The `Release` whose output creates the connection never copies the message that the status of the
  connection holds. When the connection is in `ERROR`, the `Release` checks it again as the connection controller
  does, and shows this reason in its status and events. A connection that no controller checked yet gives
  `Connection 'db': not checked yet` in the status of the `Release`, or
  `Connection 'db': not checked, as the connections feature gate is off` when the `connections` feature gate is off.
- The `ClusterConnection` of an output is named `kcd-<namespace>-<release>-<output>`. When one of that name exists
  and its `spec.parentRelease` names another `Release`, the `Release` leaves it as it is and goes to `ERROR` with
  `output[0]: its ClusterConnection belongs to another Release`, and the controller tries again later. One without
  `spec.parentRelease` is taken over.

Some messages, as the status of a `Release` named `demo` shows them:

```
failed to populate HelmRelease 'demo-main': specPatch of module 'main' has a field that HelmRelease does not define
error while creating HelmRelease 'demo-main': the API server answered Invalid for spec.timeout, message withheld, as it can quote the object
output[0]: error while creating its Connection: the API server answered Invalid for metadata.name, message withheld, as it can quote the object
output[0]: its ClusterConnection belongs to another Release
input#1: could not list the Connections: the API server answered Forbidden, message withheld, as it can quote the object
condition[0]: the cel expression is not valid, message withheld, as it can quote the condition or the object
Waiting for cond.#1: Context.kubocd.kubotal.io 'apps/cluster': the cel expression is false
Connection 'db': validation error on values: (root): a property is not declared in the schema
Connection 'db': not checked yet
```

## Upgrading from v0.3.x

The upgrade note of the [KuboCD changelog](https://github.com/kubocd/kubocd/blob/main/CHANGELOG.md#upgrading-from-v03x),
with the fixes of later versions:

- Before upgrading the controller, use the v0.4.0 CLI's `kubocd dump package <manifest>` to check package parsing
  without publishing an image. It rejects a template calling a function outside the list, and a manifest with map keys
  that are the same once converted to strings. This check does not execute templates or compile templated
  `connectionRef` defaults. Also test representative Releases with `kubocd render`. Upgrade consuming controllers before
  publishing packages that rely on the new literal `${...}` behaviour. Once the controller is upgraded, a Release whose
  package fails stays in error, its HelmReleases are left as they are, and the Releases depending on its roles wait.
- A package calling `resolveDNS` needs `controller.allowDNSLookups: true`, or the address written in its parameters or
  its context.
- A manifest relying on `${VAR}` substitution is prepared by the CI before `kubocd package`.
- A template reading `.ImageRedirector` reads `.ImageRedirects` instead. One calling `slice` with a non-integer index
  converts it with `int`.
- The message that v0.3.x wrote in the status of a Connection or a ClusterConnection, which can name a property of its
  values, is replaced when the connection controller, which runs with the `connections` feature gate on, reconciles
  the connection and records its status. It reconciles every connection when it starts. A refused read of the contract
  or a refused patch of the status delays the replacement to a later reconciliation. A Release never copies this
  message: it checks an output connection in `ERROR` again, as the connection controller does. What v0.3.x copied into
  the status of a Release is replaced, at the latest, when the Release reconciles its outputs, once its HelmReleases
  are ready. Events already emitted keep their text.

The functions removed in v0.4.0 are:

- the KuboCD functions `readFile`, `toToml`, `fromYamlArray` and `fromJsonArray`,
- the `include`, `tpl` and `lookup` placeholders, which returned placeholder values,
- every Sprig function that is neither in the [list](#sprig-functions) nor replaced by a
  [KuboCD function](#kubocd-functions) or a [built-in](#go-texttemplate-built-ins) of the same name, among them
  `getHostByName`, `now`, `date`, `randAlphaNum`, `uuidv4`, `genCA`, `b64enc` and `contains`.

`env` and `expandenv` were already unavailable in v0.3.x.

Up to v0.3.x, the CLI and the controller replaced `${NAME}`, where `NAME` is made of letters, digits and `_`, with the
environment variable `NAME` when they read a manifest, and failed when it was unset or empty. Other forms, such as
`${NAME:-x}` or `${NAME#s3://}`, already stayed as written.

A package calling `resolveDNS` to get an address can take it from its data instead. For example, replace:

```yaml
metallb.universe.tf/loadBalancerIPs: {{ resolveDNS .Context.ingress.host }}
```

with an address set in the context, such as `ingress.vip: 10.0.0.10`:

```yaml
metallb.universe.tf/loadBalancerIPs: {{ .Context.ingress.vip }}
```

`.ImageRedirector` was an object of the controller. `.ImageRedirects` is a plain list, and calls to `imageRedirect` are
unchanged.

The list only grows on request: a package that needs another function, whose result depends on its arguments only, can
ask for it.
