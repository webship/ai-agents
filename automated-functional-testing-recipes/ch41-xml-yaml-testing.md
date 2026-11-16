# Chapter 41: XML and YAML Response Testing

---

Modern APIs default to JSON. But the industry runs on more than modern APIs. SOAP services power banking, telco, and government integrations. RSS and Atom feeds drive news aggregators. Sitemaps are XML. Kubernetes manifests, Helm charts, and CI configs are YAML. Configuration files for thousands of internal tools are YAML. If your test suite handles only JSON, you have a blind spot wherever a partner, vendor, or platform speaks anything else.

Webship-js ~2.0 adds **XML response testing** and **YAML response testing** families. You parse the response, then assert against elements, attributes, namespaces, types, and structural constraints -- without ever writing a custom step.

This chapter starts with XML, then moves to YAML. The two families share most concepts -- if you understand one, the other is a re-read.

---

## XML and YAML, Side by Side

| Concern | XML step prefix | YAML step prefix |
|---------|-----------------|------------------|
| Element exists | `the XML element "..."` | `the YAML element "..."` |
| Element value | `... should be equal to` | `... should be equal to` |
| Attribute | `the XML attribute "..." on element "..."` | `the YAML attribute ...` |
| Namespace | `the XML should use the namespace` | `the YAML should use the namespace` |
| Type | (XML schema) | `the YAML value at "..." should be of type` |
| Multi-document | -- | `the YAML response should have N document(s)` |

XPath is the path language for XML. YAMLPath (`a.b[0].c`) is the path language for YAML. Both are exposed via the same step shapes.

---

## Recipe 397: Asserting an XML Element Exists

### Problem
The third client your team onboards is a logistics company, and their order API speaks SOAP -- XML envelopes, not the JSON your whole suite is built around. The partner's documentation swears `<orderId>` is always in the response, but their Tuesday deploys have broken that promise twice, and each time your order pipeline failed downstream where it was hardest to trace. You need the integration test that catches it at the boundary, without writing a custom XML parser for one partner.

### Solution

```gherkin
Feature: SOAP order placement

  Scenario: Order envelope contains orderId
    Given the API base URL is "https://partner.example.com/soap"
    When I send a POST request to "/orders" with body:
      """
      <?xml version="1.0"?>
      <Envelope><Body><PlaceOrder><Item>SKU-42</Item></PlaceOrder></Body></Envelope>
      """
    Then the response should be in XML format
    And the XML element "//orderId" should exist
```

The plumbing is the same API family you know from Chapter 11 -- `the API base URL is` sets the host, `with body:` sends the docstring verbatim as the request payload. What changes is the assertion layer: `should be in XML format` parses the response body into an XML document (and fails loud, with the parser error, if the partner returns an HTML error page wearing an XML content type -- a thing partners do). The `exist` step then runs the XPath against that parsed document and passes if the match set is non-empty. `//orderId` means "an `<orderId>` element at any depth" -- forgiving of envelope restructuring. Swap in a strict path like `/Envelope/Body/Response/orderId` when the position itself is the contract. One trap worth knowing before it costs you an afternoon: if the partner namespaces every element, a bare `//orderId` may match nothing -- the portable XPath form is `//*[local-name()='orderId']`, and Recipe 401 shows how to assert the namespace itself.

**At scale:** SOAP partners multiply quietly across an enterprise portfolio -- banking, telco, government, logistics -- and the worst outcome is each project growing its own ad-hoc XML parsing. With one shared step family, the QA engineer who wrote the SOAP suite for project three reads project nine's suite without a glossary. Keep one feature file per partner contract and tag it with the partner name, so when "the partner deployed something" alerts fire, you can run exactly that contract suite on demand.

---

## Recipe 398: Asserting XML Element Value

### Problem
Your order code treats any status it doesn't recognise as a failure -- a sane default. Then the partner ships a minor release that changes `<status>Accepted</status>` to `<status>ACCEPTED</status>`, every order starts "failing" at 2am, and the on-call engineer spends an hour proving the orders actually went through. Existence checks can't catch this; the element was there all along. The *value* is the contract.

### Solution

```gherkin
  Scenario: Status is Accepted
    Given the API base URL is "https://partner.example.com/soap"
    When I send a POST request to "/orders" with body:
      """
      <?xml version="1.0"?>
      <Envelope><Body><PlaceOrder><Item>SKU-42</Item></PlaceOrder></Body></Envelope>
      """
    Then the XML element "//status" should be equal to "Accepted"
```

The step resolves the XPath, takes the first matching element's text content, trims surrounding whitespace, and compares it to the expected string -- exact and case-sensitive, which is exactly what you want for an enum-like status field, because `ACCEPTED` vs `Accepted` is precisely the bug this test exists to catch. The whitespace trim handles pretty-printed XML, where the text node is really `\n      Accepted\n    `. Use `should contain` when the element wraps prose and you only care about a fragment. One thing XML doesn't give you: types. Without a schema, `<count>3</count>` is the *string* "3" -- compare it as one.

**Pro Tip:** put `Then the API response code should be 200` before the value assertion. When the partner is down, you want the failure to read "got 503" -- transport problem -- not "expected Accepted, found nothing", which sends the triager hunting a content bug that doesn't exist.

---

## Recipe 399: Asserting XML Attribute Value

### Problem
The partner's `<order>` element carries a `currency="USD"` attribute -- and their parser quietly defaults to EUR when the attribute is missing or misspelled. You learned this from an invoice that was off by exactly one exchange rate. Element-level assertions look straight past attributes, so this contract needs its own check. And you'd rather verify your assertion logic against a saved sample than depend on the partner's sandbox being up during every CI run.

### Solution

```gherkin
  Scenario: Order currency is USD
    Given the response content from the file "fixtures/order.xml"
    Then the XML attribute "currency" on element "//order" should be equal to "USD"
```

Two things are happening here. First, `the response content from the file` loads a saved XML sample from disk and treats it as if it were the live response -- so the rest of the assertion family works identically against fixtures and against real calls. That decouples your contract tests from partner uptime: the live smoke test runs once, the fixture-based assertions run on every commit. Second, the attribute step takes two arguments -- the attribute name and the XPath of the element that owns it -- finds the first matching element, and compares that element's attribute value. `should not be equal to`, `should contain`, and `should not contain` complete the family.

**At scale:** check the `fixtures/` directory into the repo and treat it as the partner's living archive. Every time a partner sends a "here's our new response format" sample -- or breaks you in production -- the offending document goes into fixtures and becomes a permanent regression test. When three projects integrate the same partner, share one fixtures package between them; the partner has one contract, your portfolio should have one copy of it.

---

## Recipe 400: Asserting XML Element Count

### Problem
A deduplication bug doubles every line item in the cart export, and not a single existence or value assertion notices -- `//cart/item` still exists, the first item's value is still right. The customer notices, though, because they were invoiced twice per item. "The element is there" and "the element is there *exactly three times*" are different contracts, and list-shaped data needs the second one.

### Solution

```gherkin
  Scenario: Cart has three items
    Given the response content is the following:
      """
      <cart>
        <item id="1"/>
        <item id="2"/>
        <item id="3"/>
      </cart>
      """
    Then the XML element "//cart/item" should have 3 element(s)
```

The step evaluates the XPath, collects the full match set instead of stopping at the first hit, and asserts the set's size -- exact equality, so both the dedup bug (6 items) and the lost-row bug (2 items) fail it. This recipe also shows the third way to feed the XML family: `the response content is the following:` takes the document inline as a docstring, which keeps tiny examples self-contained -- handy for the assertion-logic tests themselves and for teaching scenarios like this one. Live response, fixture file, inline docstring: same assertions over all three.

---

## Recipe 401: Asserting XML Namespace Use

### Problem
Your service's SOAP responses are consumed by a strict .NET client at the partner's end -- the kind that validates the envelope namespace and rejects anything else with an opaque fault. Then someone upgrades the XML library on your side, the default namespace prefix changes, and a well-meaning "cleanup" drops the declaration entirely. Browsers and lenient parsers shrug; the partner's gateway returns errors, and the integration is down while two teams blame each other's logs. Namespace compliance is invisible until the strictest consumer in the chain enforces it.

### Solution

```gherkin
  Scenario: SOAP envelope declares standard namespace
    Given the response content from the file "fixtures/soap-response.xml"
    Then the XML should use the namespace "http://schemas.xmlsoap.org/soap/envelope/"
```

The step scans the parsed document for `xmlns` declarations and passes if any of them declares the given URI. The match is on the *URI*, not the prefix -- and that's the correct semantics, because `soap:Envelope`, `s:Envelope`, and `SOAP-ENV:Envelope` are all the same document as long as the prefix binds to the same URI. Prefixes are arbitrary local names; the URI is the identity. That's also why the library-upgrade scenario above is so sneaky: the prefix changing is harmless, the URI disappearing is the outage. `should not use` inverts the check -- use it to assert a legacy or internal namespace has *not* crept into a public document.

**Pro Tip:** when a namespace assertion fails, diff the fixture against the live response before touching code. Nine times out of ten the declaration moved (from the envelope to the body element, say) rather than vanished -- still a contract change worth flagging to the partner, but a different conversation than "we broke SOAP."

---

## Recipe 402: Asserting a YAML Element Exists

### Problem
Your platform team built an endpoint that renders pod manifests for the deploy tooling -- generated YAML, assembled from templates and overrides. Generated is the key word: one template refactor can drop the whole `metadata` section, the endpoint still returns 200 with perfectly valid YAML, and the deploy tool crashes at apply time. Which, by Murphy's law, is during the release window. The cheapest insurance is a test that asks the endpoint for a manifest and checks the load-bearing sections are actually in it.

### Solution

```gherkin
Feature: Manifest endpoint

  Scenario: Pod manifest has metadata
    Given the API base URL is "https://api.example.com"
    When I send a GET request to "/k8s/pods/payment-api"
    Then the response should be in YAML format
    And the YAML element "metadata" should exist
```

`should be in YAML format` parses the body first and fails with the parser's error -- line and column -- if the response isn't valid YAML at all. That's your fail-fast layer: a tab character or a stray template artifact gets reported as "invalid YAML at line 12", not as a confusing path-not-found further down. After parsing, the document is a plain object tree, and `should exist` resolves the YAMLPath against it: dotted keys for maps, brackets for array indices, so `spec.containers[0].image` reads "the image of the first container under spec". Exists means the path resolves to *something* -- including `null` or an empty map -- so pair it with value or type assertions (Recipes 403-404) when "present" isn't enough.

**At scale:** the same assertions don't have to wait for a live endpoint. Render your Helm charts or kustomize overlays in CI, point the file-based YAML steps at the output, and every one of your twenty services gets manifest checks *before* anything reaches a cluster. The QA suite and the platform team end up sharing one vocabulary for "the manifest is well-formed" -- and that conversation otherwise tends to happen in an incident channel.

---

## Recipe 403: Asserting YAML Element Value

### Problem
The payment service was bootstrapped the way every service is bootstrapped: copy the billing service's manifests, rename what you remember to rename. Someone missed `metadata.name`. The YAML is valid, the endpoint returns 200, the deploy goes green -- and the tooling just rolled payment code out under the billing service's name, on top of the billing deployment. Existence checks (Recipe 402) can't see this; the field is there. It's just wrong.

### Solution

```gherkin
  Scenario: Pod name is payment-api
    Given the YAML response content from the file "fixtures/pod.yaml"
    Then the YAML element "metadata.name" should be equal to "payment-api"
```

Same shape as XML, different path language. `the YAML response content from the file` parses the fixture once into a plain object tree; the assertion then walks the YAMLPath one dotted segment at a time -- map key `metadata`, then map key `name` -- and compares the scalar it lands on against the expected string. Where XPath queries a document and can return many matches, YAMLPath walks exactly one route to exactly one value, which makes failures unambiguous: either the path didn't resolve, or the value didn't match, and the message tells you which. Both families share the same comparison set -- `should be equal to`, `should not be equal to`, `should contain`, `should not contain` -- so the muscle memory transfers.

**At scale:** copy-paste bootstrapping is exactly how a twenty-service portfolio gets born, which means copy-paste residue is a portfolio-wide failure class, not a one-off. Keep a tiny "identity check" scenario -- name, namespace, team label -- in every service's suite, generated from the same template with one value changed. It is the cheapest test you will ever write, and it is aimed at the single most common manifest mistake in multi-project teams.

---

## Recipe 404: Asserting YAML Value Type

### Problem
A templating change wraps a value in quotes -- `replicas: "3"` instead of `replicas: 3` -- and nothing complains until the deploy tool does, at apply time, during the release window. This is YAML's signature trap: the difference between a number and a string is two invisible characters, every value assertion passes either way ("3" equals "3"), and the type only matters to the strictest consumer downstream. You want the type checked where it's cheap: in the suite, on every commit.

### Solution

```gherkin
  Scenario: Replicas is a number
    Given the YAML response content from the file "fixtures/deployment.yaml"
    Then the YAML value at "spec.replicas" should be of type "number"
```

Here is what's really being tested: YAML's own typing rules decided what landed in the parsed tree. An unquoted `3` parses as a number; a quoted `"3"` parses as a string; `true`, `false`, `null`, and `~` parse as their own types. The step resolves the path and applies JavaScript's `typeof` semantics to whatever the parser produced, with `Array.isArray` distinguishing arrays from objects. Supported types: `string`, `number`, `boolean`, `array`, `object`, `null`. So a failure here isn't pedantry -- it means the document's quoting changed, which usually means a template changed, which is precisely the regression you wanted flagged before the cluster saw it.

**Pro Tip:** the most famous version of this bug is the Norway problem -- in YAML 1.1-style parsers, an unquoted `country: NO` parses as boolean `false`. Put a `should be of type "string"` assertion on any field whose values come from a country, locale, or version list (`version: 1.20` parses as the number `1.2`). One line of Gherkin, and an entire category of "works for every country except Norway" bugs is gone.

---

## Recipe 405: Asserting YAML Numeric Range

### Problem
Someone load-tested on a Friday and left `replicas: 40` in the values file. Nothing is invalid -- the cluster will cheerfully schedule forty pods -- and the first signal anyone gets is the infrastructure bill, three weeks later, routed to a manager who has to ask "what changed?" The opposite direction is worse: `replicas: 0` is also valid YAML, and it serves exactly zero requests. Valid-but-absurd values are a class of bug that no schema and no parser will ever catch for you, because only your team knows what "sane" means here.

### Solution

```gherkin
  Scenario: Replicas in safe range
    Given the YAML response content from the file "fixtures/deployment.yaml"
    Then the YAML value at "spec.replicas" should be greater than or equal to 1
    And the YAML value at "spec.replicas" should be less than or equal to 10
```

The step resolves the path, requires the resolved value to be numeric, then applies the comparison. Stacking two assertions, as here, gives you a closed range with two readable failure messages -- or use the single `between X and Y` form when one line reads better. The full family: `greater than`, `greater than or equal to`, `less than`, `less than or equal to`, `between X and Y`. Floats are accepted. One useful side effect: if the value is the *string* `"3"`, the numeric assertion fails -- which is correct behaviour, because it just surfaced the quoting bug from Recipe 404 for free.

**At scale:** range checks are how tribal knowledge becomes executable policy. Every platform team has unwritten rules -- "never more than four replicas in dev", "memory limits under 2Gi outside prod" -- that live in people's heads until a values file violates them. Write the guardrails once as a shared feature file, render every service's manifests in CI, and run the same file against all of them. The violation then shows up in a pull request review instead of on the cloud invoice, and the rule survives the person who made it leaving the team.

---

## Recipe 406: Asserting YAML Array Contents

### Problem
Arrays are where path-based assertions get brittle. The `app` container you care about sits at a different index in every service -- index 0 here, index 1 where a proxy sidecar comes first -- so `spec.containers[0].name` is a guess dressed up as a contract, and it breaks the day someone reorders the list for readability. Worse is the inverse case: during an incident, someone attached a debug sidecar to the pod spec and forgot to remove it. Both problems need assertions about *membership*, not position.

### Solution

```gherkin
  Scenario: Containers list has the app container
    Given the YAML response content from the file "fixtures/pod.yaml"
    Then the YAML array at "spec.containers" should contain an item where "name" is "app"
```

The predicate iterates every entry in the array, reads the named subfield (`name`) on each, and passes on the first match -- order-independent by design, because the order of `spec.containers` is an implementation detail nobody promised you. That is the whole argument against index paths: `containers[0]` encodes an ordering, this step encodes an identity. The negation, `should contain no item where "name" is "debug-shell"`, walks the *entire* array and fails on any match -- that's your forbidden-config guard, the YAML cousin of the ZIP secrets check from Recipe 393. Stack both in one scenario when the contract is "app must be there, debug must not."

---

## Recipe 407: Asserting YAML Has Exactly the Expected Keys

### Problem
Extra keys are how config rot starts. A field gets renamed -- `replicaCount` becomes `replicas` -- and the old key stays behind, silently ignored by the tooling but faithfully copied into every new service bootstrapped from this file. Eighteen months later there are nine keys at the top level, nobody can say which four are load-bearing, and the new hire's first PR edits the dead one. Presence checks can't stop this; only an *exact* key list can, because rot is made of keys that shouldn't be there.

### Solution

```gherkin
  Scenario: Top-level keys are exact
    Given the YAML response content from the file "fixtures/deployment.yaml"
    Then the YAML keys at "" should be exactly "apiVersion,kind,metadata,spec"
```

The path `""` refers to the document root; any other YAMLPath works too, so you can pin the keys of `metadata` or `spec` just as tightly. The step collects the keys of the map at that path and compares them as a set against the comma-separated list -- it fails on a missing key *and* on an extra one, naming the difference in the message, which turns triage into a one-glance job. Choose your strictness deliberately: `should be exactly` for formats you own, where every new key should be a conscious, reviewed decision; `should have keys` for "must contain at least these" semantics, when a tool you don't control is allowed to add fields around yours. Exact-keys assertions are the lint rule that makes cleanup permanent -- once the dead key is removed, it can never quietly come back.

---

## Recipe 408: Asserting YAML Document Count and No Duplicates

### Problem
Your stack file holds three documents -- a Service, a Deployment, an HPA -- separated by `---`. Then a messy merge-conflict resolution deletes one separator, two documents fuse into one, and the apply creates two objects instead of three. The YAML still parses, so nothing upstream complains. The other trap is quieter still: the same key pasted twice into one document. Most parsers shrug and keep the last occurrence, which means the first half of your duplicated config simply stops applying -- and there is no error anywhere to tell you.

### Solution

```gherkin
  Scenario: Multi-document manifest is well-formed
    Given the YAML response content from the file "fixtures/stack.yaml"
    Then the YAML response should have 3 document(s)
    And the YAML should have no duplicate keys
```

The parser splits the stream on `---` separators and the count assertion checks how many documents came out -- so both failure directions are covered: a lost separator fuses documents and the count drops; a stray separator splits one and the count rises. Either way the number changes, and the number is the contract. The duplicate-keys check then re-examines each document with duplicate detection switched on, because default last-wins parsing is exactly what makes this bug invisible at runtime. Together the two lines are a structural lint for any hand-edited or template-generated multi-document file -- the well-formedness floor under all the value assertions above.

**At scale:** these two steps make a good universal pre-flight. Every rendered manifest bundle in the portfolio, whatever the service, can run "expected document count, no duplicate keys" before any service-specific assertion -- one shared scenario outline, parameterized only by file and count. When a platform-wide template change breaks rendering, twenty pre-flights fail with the same clear message at the same moment, which tells the on-call engineer "template problem, one fix" instead of twenty mysterious service-level failures.

---

## Bonus: JSON Schema for YAML

### Problem
Path-by-path assertions scale to a point. By the time the manifest contract covers forty fields, the feature file is longer than the manifest itself -- and it still can't say "and nothing else changed shape." Meanwhile the schema describing this format already exists: your platform team maintains one, or the upstream tool publishes one. Re-transcribing it into Gherkin, one assertion at a time, is the kind of duplication that drifts the first month nobody maintains it.

### Solution

```gherkin
  Scenario: Pod manifest matches schema
    Given the YAML response content from the file "fixtures/pod.yaml"
    Then the YAML should match JSON Schema "fixtures/schemas/pod.schema.json"
```

The schema is loaded from disk and the parsed YAML is validated against it -- and yes, JSON Schema validates YAML just fine, because by the time the validator runs, the YAML is already an object tree and the schema never knows what text format it came from. One step asserts the whole shape: required fields, types, enums, nesting, and (if the schema sets `additionalProperties: false`) the "nothing extra" guarantee that would otherwise take a stack of exact-keys assertions. Failures report path and message -- "spec.replicas must be number", not "schema mismatch" -- so a red run points at the field, not at the concept of schemas. Keep the targeted steps from Recipes 402-408 for the two or three values your team actually argues about; let the schema carry the other forty.

**At scale:** the schema is the contract -- so have exactly one. When twelve services emit the same manifest format, publish the schema once (an internal package, or a plain shared fixtures repo) and point every suite's step at that single file. A format change then becomes one reviewed pull request that every consuming team can see and veto, instead of twelve slowly diverging copies of the same assertions. This is also the artifact to hand the PMO when they ask what "config is valid" means: a versioned schema beats a wiki page every time.

---

## When to Use XML/YAML Testing

You need this family when:

- Your service speaks SOAP, RSS, Atom, or XHTML
- Your CI publishes Kubernetes manifests, Helm charts, or CloudFormation YAML
- A partner API returns XML and you want to test integration without a custom parser
- A config file format is YAML and you want lint-style assertions on its structure

For pure JSON APIs, stick with the JSON assertions from Chapter 11 (`the JSON response should have property "..."`). The XML and YAML families exist for everything else.
