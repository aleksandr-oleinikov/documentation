---
description: 'This feature is introduced in August 2021 - v:2108 release'
---

# Draft: Profiling with zen-lang

For a long timeAidbox provided validation with JSON Schema and Basic FHIR Profiles. We're switching to **zen/schema** as the main engine for the validation and configuration of Aidbox.

### Before start

Install the devbox sample repo following [this guide](../getting-started/installation/setup-aidbox.dev.md) 

Make sure that you have the latest `devbox:edge` image

```bash
docker pull healthsamurai/devbox:edge
```

Make sure your devbox is configured to use `devbox:edge` image.  
In the `.env` file find the line starting with `AIDBOX_IMAGE` and edit it to be like this if it is not:

{% code title=".env" %}
```bash
AIDBOX_IMAGE=healthsamurai/devbox:edge
```
{% endcode %}

### Create a zen project

Inside of the cloned Devbox directory create a zen project directory and open it.   
In this example zen project directory is `my-zen-project/`

```bash
mkdir my-zen-project
cd my-zen-project
```

Inside of your zen project directory initialize npm package for your project and then install zen dependencies you need. FHIR R4 and US core v3 are used as dependencies in this example

```bash
npm init -y
npm install @zen-lang/fhir-r4 @zen-lang/us-core-v3
```

Create zen entry namespace.

{% code title="my-zen-project/my-zen-devbox.edn" %}
```yaml
{ns my-zen-devbox}
```
{% endcode %}

Create namespace with your profiles

{% code title="my-zen-project/my-zen-profiles.edn" %}
```text
{ns my-zen-profiles
 import #{us-core-v3.us-core-patient}

 MyPatientProfile
 {:zen/tags #{zen/schema zenbox/profile-schema}
  :confirms #{us-core-v3.us-core-patient/schema}
  :zenbox/type "Patient"
  :zenbox/profileUri "urn:profile:MyPatientProfile"
  :type zen/map
  :require #{:birthDate}}}
```
{% endcode %}

Import the profiles namespace in the entry namespace

{% code title="my-zen-project/my-zen-devbox.edn" %}
```text
{ns my-zen-devbox
 import #{my-zen-profiles}}
```
{% endcode %}

### Setup Devbox to use Zen project

Go to Devbox directory and add configuration variables to the end of `.env` file:

{% code title=".env" %}
```bash
AIDBOX_ZEN_PROJECT=/my-zen-project # mind the / at the beginning of the dir name
AIDBOX_ZEN_ENTRY=my-zen-devbox
AIDBOX_ZEN_DEV_MODE=enable # set this variable if you want to have 
                           # zen reload namespaces on the fly as they change
```
{% endcode %}

To enable your Devbox reading the zen project data add zen project directory volume mount. In the `docker-compose.yaml` file find the section `devbox:` and add the following line under the `volumes:` section:

```yaml
- "./my-zen-project:/my-zen-project"
```

The `volumes` section should be like this

```yaml
services:
  devbox:
    volumes:
    - "./my-zen-project:/my-zen-project"
```

So the file should look like this:

{% code title="docker-compose.yaml" %}
```yaml
version: '3.7'
services:
  devbox-db:
    <...omitted...>

  devbox:
    container_name: "devbox"
    image: "${AIDBOX_IMAGE}"
    <...omitted...>
    volumes:
    - "./my-zen-project:/my-zen-project"
  <...omitted...>
```
{% endcode %}

### Start Devbox and check if your profile is loaded

Start Devbox using [this guide](../getting-started/installation/setup-aidbox.dev.md#run-devbox) \(perform steps from Devbox directory, not in zen project dir\)

Open Devbox in your browser and click `Profiles` tab in the left menu:

![](../.gitbook/assets/image%20%2871%29.png)

You should see the list of zen namespaces loaded.

![](../.gitbook/assets/image%20%2889%29.png)

{% hint style="info" %}
 On this page you see the namespaces that are explicitly included in the zen project or used by Aidbox
{% endhint %}

Open your profile by clicking its name

![](../.gitbook/assets/image%20%2891%29.png)

Using **validate** tab you can test the data against this profile

![](../.gitbook/assets/image%20%2880%29.png)

If your profile is tagged `zenbox/profile-schema` it can be used to validate your data  
On FHIR CRUD API requests a profile will be applied if data includes `:zenbox/profileUri` in the `meta.profile` attribute:

{% tabs %}
{% tab title="Request" %}
{% code title="Data is missing \`birthDate\` attribute, required by the profile" %}
```http
POST /Patient
Content-Type: text/yaml

name: [{given: [Harry], family: Potter}]
meta: {profile: ["urn:profile:MyPatientProfile"]}
```
{% endcode %}
{% endtab %}

{% tab title="Response" %}
{% code title="An error has been returned" %}
```yaml
# status 422
resourceType: OperationOutcome
text:
  status: generated
  div: Invalid resource
issue:
  - severity: fatal
    code: invalid
    expression:
      - Patient.birthDate
    diagnostics: ':birthDate is required'
```
{% endcode %}
{% endtab %}
{% endtabs %}

{% tabs %}
{% tab title="Request" %}
{% code title="Data contains \`birthDate\` required by profile" %}
```http
POST /Patient
Content-Type: text/yaml

name: [{given: [Harry], family: Potter}]
birthDate: "1980-07-31"
meta: {profile: ["urn:profile:MyPatientProfile"]}
```
{% endcode %}
{% endtab %}

{% tab title="Response" %}
```yaml
# status 201

id: 20fdde9d-ede6-4edb-a16a-278ddfeef952
resourceType: Patient
name:
  - given:
      - Harry
    family: Potter
birthDate: '1980-07-31'
meta:
  profile:
    - urn:profile:MyPatientProfile
  lastUpdated: '2021-09-03T14:40:11.300957Z'
  createdAt: '2021-09-03T14:40:11.300957Z'
  versionId: '57'
```
{% endtab %}
{% endtabs %}
