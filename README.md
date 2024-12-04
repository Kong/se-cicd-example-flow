# CI/CD example flows from Insomnia to Konnect

## What is this about?
Kong offers the full tool chain to automate the API lifecycle. This repository is a showcase of how to automate the API lifecycle from design to production using the Kong tool chain.

![Gitops](https://prd-mktg-konghq-com.imgix.net/images/2023/09/6512dda2-diagram-apollo-kong-api-ops.png?auto=format&fit=max&w=1920)

This service and the GitHub repo this OpenOPI is living in is about showing all the different ways of automation using the Kong tool chain.
    
1. Designing an OpenAPI in Insomnia
2. Using collections to test auto APIs
3. Creating test collection
4. Automating the creation of a sandbox / mockup version in Kong Gateway
5. Propagate automatically to the Kong Gateway
6. Publish and document on the Konnect Developer Portal

## Overview of the flow

![Overview of all steps](https://raw.githubusercontent.com/Kong/se-cicd-example-flow/refs/heads/main/images/flow-overview.jpeg)

### Checkout workspace from GitHub

All data being worked on by the team is stored in a [GitHub repostiory](https://github.com/Kong/se-cicd-example-flow). This includes the OpenAPI spec, the Insomnia workspace, and the test collection.

In the first step our CI/CD pipeline checks out the workspace from GitHub <https://github.com/Kong/se-cicd-example-flow>.

```bash
git clone https://github.com/Kong/se-cicd-example-flow.git
cd se-cicd-example-flow
```

### Export and validate the OpenAPI spec

The OpenAPI itself is extracted from the repository by running the [Insomnia CLI](https://insomnia.rest/features/insomnia-cli) in a script

```bash
inso export spec -w . "CI/CD example flows from Insomnia to Konnect" -o openapi-from-insomnia.yaml
```

The OpenAPI spec is then validated using `inso cli` to ensure it is correct.

```bash
inso lint spec -w . openapi-from-insomnia.yaml
```

### Run the tests / the request collection

In the Insomnia workspace there is a collection of requests that can be used to test the API. This collection is exported and run using the Insomnia CLI.

```bash
inso run collection -w . "CI/CD example flows from Insomnia to Konnect" --env "OpenAPI env"
```

### Deploy a sandbox version using the Mocking plugin

Using the OpenAPI spec and the [Mocking plugin](https://docs.konghq.com/hub/kong-inc/mocking/), a sandbox version of the API is created on the Kong Gateway on the path `/cicd/sandbox` which is synced into the gateway using [decK](https://docs.konghq.com/deck/latest/)

```bash
deck gateway sync (...) --select-tag import-OAS-from-Insomnia-sandbox ../sandbox.yaml
````

The service, route and mocking plugin are defined in the `sandbox.yaml` file. The contents of the environment variable `DECK_CICD_OPENAPI` are filled in the script from the OpenAPI file.

```yaml
_format_version: "3.0"
services:
- host: fake.example.com
  id: 804b2296-681b-4e92-a4bf-0e702defa27d
  name: sandbox-ci-cd-example-flows-from-insomnia-to-konnect
  path: /
  port: 443
  protocol: https
  plugins:
    - name: mocking
      config:
        custom_base_path: /cicd/sandbox
        include_base_path: true
        random_examples: true
        api_specification: |
${{ env "DECK_CICD_OPENAPI" }}
      enabled: true
  routes:
  - name: sandbox-ci-cd-example-flows-from-insomnia-to-konnect
    paths:
    - /cicd/sandbox
```

### Deploy the API to production incl. plugins

The production version of the API is deployed to the Kong Gateway by generating the configuration file with `deck file openapi2kong` and adjusting the external url to `/cicd/v1` by `deck file namespace --path-prefix=/cicd/v1` .

```bash
deck file openapi2kong -s ./openapi-from-insomnia.yaml -o kong-deck-generated.yaml
deck file namespace --path-prefix=/cicd/v1 --state=kong-deck-generated.yaml -o kong-deck-generated.yaml
```

As part of the pipeline the platform owners have automated that all API request are validated againt the OpenAPI spec. This is done by using the [OAS Validation](https://docs.konghq.com/hub/kong-inc/oas-validation/) plugin which is auto-injected into the configuration before it gets applied to the gateway.

```yaml
cat kong-deck-generated.yaml | deck file add-plugins ../oas-validation.yaml -o kong-deck-generated.yaml
```

### Create an API Product and publish both versions to developer portal

Now that the API is deployed to the Kong Gateway, an API Product is created to group the sandbox and production versions of the API.

As a result the automatically created service is now discoverable in the Developer Portal and both sandbox and production are fully documented.

The documentation also inserts based on the pipeline additional informations from the GitHub repository like the Markdown file you are reading right now.

```bash
http POST https://eu.api.konghq.com/v2/api-products name="CI/CD Example Pipeline"
http POST https://eu.api.konghq.com/v2/api-products/$productId/documents slug=cicd-example-doc
http POST https://eu.api.konghq.com/v2/api-products/$productId/product-versions name="sandbox"
http POST https://eu.api.konghq.com/v2/api-products/$productId/product-versions name="v1"
``` 
