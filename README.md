<!-- markdownlint-disable-next-line -->

**This README has been modified from the original README for the OpenTelemetry Demo for the purposes of the following workshop.**

# RenderATL 2024 Workshop: What’s wrong with my app?: Using OpenTelemetry to observe your code 

**Workshop requirements:** 
1. Laptop
2. A Github account
3. A (free!) New Relic account -- sign up [here](https://newrelic.com/signup) with your personal (not work!) email

## What is this workshop about? 
Having observability into your applications and infrastructure empowers you with multiple benefits, such as being able to pinpoint issues in your code down to the line, minimize application downtime, and ensure your system is running smoothly. These, and many more benefits, are only possible when you have observability into your applications and infrastructure. 

[OpenTelemetry](https://opentelemetry.io/docs/what-is-opentelemetry/) is both a framework and a tool for implementing observability, and [New Relic](https://docs.newrelic.com/docs/new-relic-solutions/get-started/intro-new-relic/) is a data visualization and storage platform. By leveraging OpenTelemetry and New Relic, you can achieve observability. In this beginner-level workshop, you’ll use the [OpenTelemetry Demo App](https://opentelemetry.io/docs/demo/) to learn about OpenTelemetry instrumentation, basic Collector architecture, and how to debug common issues, such as missing data, as well as how to navigate your traces, metrics, and logs in New Relic.

## Why do I need this repo?
We will be creating a [codespace](https://docs.github.com/en/codespaces/overview) using this repo, which is a fork of the OpenTelemetry Demo and contains a few modifications for this workshop. You can view the original repo by switching to the main branch, or by clicking [here](https://github.com/open-telemetry/opentelemetry-demo). 

This README is intended to be an accompaniment to the workshop, and contains handy links and snippets to copy and paste. 

## What is the OpenTelemetry Demo app?
This repository contains a fork of the OpenTelemetry Astronomy Shop, which is a microservice-based
distributed system intended to illustrate the implementation of OpenTelemetry in
a near real-world environment. 

The labs in this workshop utilize the demo to illustrate a few key concepts, including instrumentation. 

## The goals of this workshop
After completing this workshop, you ~~hopefully will~~ should:

* Understand the basics of OpenTelemetry Collector configuration and Collector logs
* Understand the basics of OpenTelemetry auto-instrumentation and custom instrumentation
* Be able to navigate OTLP data and do basic troubleshooting in New Relic 

## The workshop labs

These are the labs that you will be completing in this workshop. Each lab includes at least one task. 

### Lab 1 The OpenTelemetry Collector
In this lab, you will:

* learn about the OpenTelemetry Collector
* configure an exporter to ship data to your New Relic account
* query your data in New Relic
* check out Collector logs

#### Task: Configure an exporter 
For this task, you will configure an exporter to ship data to your New Relic account. 

1. In your codespace, head to this directory: `src > otelcollector`.
2. You’ll see two Collector configuration files. Click on this file to open it in your codespace: `otelcol-config-extras.yml`.
3. On line 16, replace `YOUR_LICENSE_KEY` with your own [New Relic account license key](https://one.newrelic.com/launcher/api-keys-ui.api-keys-launcher). Learn more about New Relic keys in our [documentation](https://docs.newrelic.com/docs/apis/intro-apis/new-relic-api-keys/#overview-keys).
4. Run `docker compose up` to build and run the app.
5. Head to [your New Relic account](https://one.newrelic.com/) to look for your data!

#### Task: Query your data
Click on `Query your data` and use one of the following example queries for span data:

* `SELECT * FROM Span SINCE 5 minutes ago`
* `SELECT * FROM Span WHERE service.name='adservice' SINCE 5 minutes ago`

An example query for log data that includes a stacktrace:
* `SELECT * FROM Log WHERE exception.stacktrace IS NOT NULL SINCE 5 minutes ago`

#### Task: Check out Collector logs
Open a new codepsace terminal, and run the following command to view only the Collector logs: `docker compose logs otelcol`.

#### Task: Look for the error in the Collector logs
1. In your codespace, open up this file: `src > otelcollector > otelcol-config-extras.yml`.
2. On line 16, modify your license key by adding an extra letter or number.
3. Restart the app by hitting `control + c` and then `docker compose up`.
4. Using the Collector logs, can you spot any clues to the problem that was caused by step 2? (Hint: you should see a 403 error.) 

### Lab 2: Instrumentation
In this lab, you will learn how to add a custom span attribute and query custom attributes in New Relic.

#### Task: Find the custom attribute in New Relic
Use this query to find spans that contain the custom attribute, `app.products_recommended.count`: 

`SELECT * FROM Span WHERE app.products_recommended.count IS NOT NULL SINCE 10 minutes ago`

#### Task: Add a custom attribute and find it in New Relic
As the PM for `recommendationservice`, you want to add a custom attribute to collect a filtered list of products to track what products are being recommended to your users.

1. Open up the `recommendation_server.py` file.
2. On line 111, add this code (make sure it starts on the same indent as the previous line):
```yaml
span.set_attribute("app.filtered_products.list", prod_list)
```
3. Restart the app by hitting `control + c` and then `docker compose up`.
4. Query for your new custom attribute in New Relic: `SELECT * FROM Span WHERE app.filtered_products.list IS NOT NULL SINCE 10 minutes ago` 

### Lab 3: Troubleshooting: Diagnose an issue with New Relic
The OpenTelemetry Demo comes with several [feature flags](https://opentelemetry.io/docs/demo/feature-flags/) that you can enable to simulate different problem scenarios. 

1. Open this file: `src > flagd > demo.flagd.json`.
2. On line 11, change the `defaultVariant` value to `on`. This feature flag generates an error for `GetProduct` requests with the product id, OLJCESPC7Z.
3. Restart the app by hitting `control + c` and then `docker compose up`.
4. After a couple of minutes, head to `productcatalogservice` in your New Relic account and use the Summary page to find the issue. (Hint: Check the error rate chart on the Summary page. You can also use the [Errors inbox](https://docs.newrelic.com/docs/errors-inbox/errors-inbox/) feature.) 

### Lab 4: Sampling: Configure a probabilistic sampler
In this lab, you will configure a [probabilistic sampler](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/probabilisticsamplerprocessor#probabilistic-sampling-processor) to sample 15% of log records. 

1. To see how many logs your app is currently sending, use the following query in your New Relic account: `SELECT count(*) FROM Log SINCE 2 minutes ago`. Note down the total (hint: it's likely over 1k). 
2. In your codespace, open the following file: `src > otelcollector > otelcol-config.yml`. **Note that this is the base Collector configuration file, not the `extras` config file.**
3. On line 39, add this configuration in the `processors` section (make sure it starts on the same indent as the previous line):
```yaml
  probabilistic_sampler:
    sampling_percentage: 15
```
3. In the `services` section, add the sampler to the array of processors in your `logs` pipeline. See the following third line (make sure you place it BEFORE the `batch` processor, because we want to sample the logs before they are batched):
```yaml
    logs:
      receivers: [otlp]
      processors: [probabilistic_sampler, batch]
      exporters: [opensearch, debug]
```
4. Restart the app by hitting `control + c` and then `docker compose up`.
5. After a few minutes, use the following query in your New Relic account: `SELECT count(*) FROM Log SINCE 2 minutes ago`. Note down the total (hint: it should be ~250). That difference in this result demonstrates the sampler at work! 

## Documentation

For detailed documentation about the OpenTelemetry Demo, see [Demo Documentation][docs]. If you're curious
about a specific feature, the [docs landing page][docs] can point you in the
right direction.

## Contributing

To get involved with the project see the OpenTelemetry Community's [CONTRIBUTING](CONTRIBUTING.md)
documentation. The OpenTelemetry Demo [SIG Calls](CONTRIBUTING.md#join-a-sig-call) are every other
Monday at 8:30 AM PST, and everyone is welcome.

