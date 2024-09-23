# Elastic OTel Workshop

## Preparation

Worth doing before the workshop to save time.
On Windows, use Git Bash or another Linux-emulating shell.

1. Clone this repository to have easy access to configuration files.

   ```shell
   git clone git@github.com:andrzej-stencel/elastic-otel-workshop.git
   cd elastic-otel-workshop
   export OTEL_WORKSHOP_DIR=$PWD
   ```

1. Download the Elastic Agent binary release. The scenario descriptions below assume you've downloaded it inside the OTel Workshop directory.

   - Go to <https://www.elastic.co/downloads/elastic-agent> and download the latest release of the Elastic Agent for your platform (version must be at least 8.15.0 or higher).

      - MacOS (Apple Silicon):

      ```shell
      curl -LO https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.15.1-darwin-aarch64.tar.gz
      tar zxvf elastic-agent-8.15.*.tar.gz
      cd elastic-agent-8.15.*/
      ```

      - Linux (x86_64):

      ```shell
      curl -LO https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.15.1-linux-x86_64.tar.gz
      tar zxvf elastic-agent-8.15.*.tar.gz
      cd elastic-agent-8.15.*/
      ```

      - Windows (x86_64):

        Note that Windows is currenty officially not supported for the Tech Preview.
        The Windows build of the Elastic Agent 8.15 binary does not include the `otel` command.
        You can run the below scenarios using the [upstream OpenTelemetry Collector Contrib binary](https://github.com/open-telemetry/opentelemetry-collector-releases/releases/).
        The below commands download the latest binary for Windows x86_64, unpack it and put it in a subdirectory
        as `otelcol` so that the execution command to run is the same as that for Elastic Agent Otel: `./otelcol`.

      ```shell
      # curl -LO https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.15.1-windows-x86_64.zip
      # unzip elastic-agent-8.15.*.zip
      # cd elastic-agent-8.15.*/
      curl -LO https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.109.0/otelcol-contrib_0.109.0_windows_amd64.tar.gz
      mkdir otelcol-contrib_0.109.0_windows_amd64
      tar zxvf otelcol-contrib_*.tar.gz otelcol-contrib.exe
      mv otelcol-contrib.exe otelcol-contrib_0.109.0_windows_amd64/otelcol
      cd otelcol-contrib_0.109.0_windows_amd64
      ```

1. Prepare an Elasticsearch instance.

   - Go to <https://cloud.elastic.co/>, log in via Google with your Elastic account and create a hosted deployment (needs to be at least 8.15.0 or higher, preferably latest version) or a serverless deployment ("Elastic for Observability" type).
     Store the password to the instance when offered, so that you can use it later.

   - Alternatively, use a local or any other Elasticsearch instance. Any Elasticsearch instance will do, as long as it is version at least 8.15.0 or higher.

## Basics of the OpenTelemetry Collector

- [Pipelines, receivers, processors, exporters](https://opentelemetry.io/docs/collector/architecture/)
  - This doesn't mention [connectors](https://opentelemetry.io/docs/collector/building/connector/#what-is-a-connector)

- [Configuration](https://opentelemetry.io/docs/collector/configuration/)

- [Troubleshooting](https://opentelemetry.io/docs/collector/troubleshooting/)

## Scenarios

All the scenarios assume you have downloaded the Elastic Agent, unpacked it and entered the unpacked directory, as described in the [Preparation](#preparation) section above.

Every scenario has its initial configuration file, e.g. `logs-from-file.yaml`, and a final version of the configuration file, e.g. `logs-from-file-final.yaml`.

Note that after updating the configuration file, you need to stop the collector process with `Ctrl+C` and re-run it in order to load the new config.
The OTel Collector does not support hot reloading of configuration.

### Collect logs from file

- Look at the file [./scenarios/logs-from-file.yaml](./scenarios/logs-from-file.yaml).
  - [Filelog receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/filelogreceiver/README.md)
  - [Debug exporter](https://github.com/open-telemetry/opentelemetry-collector/blob/main/exporter/debugexporter/README.md)

- Run `./otelcol --config ../scenarios/logs-from-file.yaml`

- Logs from file not available in collector's output. There's a warning: "no files matched the configured criteria"
  - The path `simple-logs.yaml` used in Filelog receiver's `include` config is relative to the configuration file,
    but the collector resolves file paths relative to the working directory.

- Fix the path by prepending the file's full path to it. You can use environment variables like `${env:HOME}` or `${env:OTEL_WORKSHOP_DIR}` as set earlier.

- The warning is gone, but still logs from the file not available in the collectors output.

- Add `start_at: beginning` to Filelog receiver's config

- Output from Debug exporter is available, but very basic. Set `verbosity` to `normal` in the exporter's configuration.

- Add `use_internal_logger: false` to Debug exporter's config to clean up the output.

- Change `verbosity` to `detailed`, analyze the output. It logs all all the log records' fields, as described in [Logs Data Model](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/logs/data-model.md#log-and-event-record-definition).

### Send the logs to Elasticsearch

- Look at the file [./scenarios/logs-into-elasticsearch.yaml](./scenarios/logs-into-elasticsearch.yaml). It is the same as `logs-from-file-final.yaml`, with Debug exporter's verbosity set to `normal`.

- Add [Elasticsearch exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/exporter/elasticsearchexporter/README.md) to the list of exporters. This involves adding a definition of the exporter in the `exporters` section of the config, as well as adding the exporter to the list of exporters in the pipeline at `service::pipelines::logs::exporters`.

- Let's start with setting connection properties, to see what happens. Set `endpoint` to `http://localhost:1234` or some other URL that does not point to Elasticsearch.

- When running with this configuration, there's no immediate error. The metrics show that the logs have been successfully sent!

- Stop the collector. Add `flush::interval: 100ms` to the Elasticsearch exporter's configuration. Now we're getting errors right away.

- Fill in the right value for `endpoint`.
  - For Elastic Cloud managed deployments, go to "Manage this deployment" and click on "Copy endpoint". I recommend to put the value in `ELASTIC_HOSTED_ENDPOINT` env var.
  - For Elastic Cloud serverless, click on "Manage project", "View" connection details and copy the value of the "Elasticsearch endpoint". I recommend to put the value in `ELASTIC_SERVERLESS_ENDPOINT` env var.

- Configure authentication in the Elasticsearch exporter.
  - For Elastic Cloud hosted deployments, you can use the user and password that you obtained when creating the deployment. Use the values as `user` and `password` configuration options of the Elasticsearch exporter.
  - Alternatively, [create an API key](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api-create-api-key.html) and set the `api_key` configuration option to the "encoded" value of the API key.

- Run the collector, check if it reports errors. Check if the logs are available in Elasticsearch.
  - Via "Observability" - "Logs"
  - Via Dev Tools Console:
    - Go to "Management" - "Dev Tools". Dismiss the help panel on the right-hand side of the screen.
    - `GET /_data_stream/_stats` to get the list of data streams.
    - `GET /_cat/indices?v` to get the list of indices.
    - `GET /logs-generic-default/_search` to get documents from the `logs-generic-default` data stream.

- Change `mapping::mode` to `otel`, re-run the collector, see how the logs structure differs.

- Change `mapping::mode` to `ecs`, re-run the collector, see how the logs structure differs.

### Keeping track of sent logs

- Notice how every time we re-run the collector, the same logs are re-collected.
  This is great for testing, but in production scenarios, we'd want the collector to pick up where it started from on restart, instead of re-reading the whole file again.

- Run and re-run `./otelcol --config ../scenarios/logs-keep-track.yaml`. Notice how the same logs appear in the Debug exporter's output every time the collector is run.

- Add the [File Storage extension](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/extension/storage/filestorage/README.md) to the configuration.
  - Create `extensions::file_storage:` entry to define the extension with the default configuration
  - Create `service::extensions: [file_storage]` entry to include the extension in the config.
  - Configure the Filelog receiver to use the extension by setting `storage: file_storage` under `receivers::filelog`.

- Run the collector, observe the error `invalid configuration: extensions::file_storage: directory must exist: stat /var/lib/otelcol/file_storage: no such file or directory`.

- Configure the File Storage extension to use a different directory with `directory: ./data`

- Run and re-run the collector. On the re-run, there should be a log saying `Resuming from previously known offset(s). 'start_at' setting is not applicable.` and the logs should not be re-collected.

- Go look in the agent's `data` directory. A file named `receiver_filelog_` should be created. This is a [BBolt](https://developer.hashicorp.com/vault/tutorials/monitoring/inspect-data-boltdb#using-the-bbolt-cli) database file.

- Delete the file and re-run the collector. The logs will be re-collected again.

### Collect host metrics

- Run the collector with the barebones [Host Metrics receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/hostmetricsreceiver/README.md) configuration: `./otelcol --config ../scenarios/host-metrics.yaml`.

- Check [docs](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/hostmetricsreceiver/README.md) for what scrapers are available and what metrics they emit. Note operating system exclusions.

- Add `load` scraper, run the collector. You should see three time series.

- Add `cpu` scraper. You should see multiple time series for each logical CPU.
  - The docs describe `cpu` metrics as not working on MacOS, but you can see they work fine with the Elastic Agent OTel.
    This wouldn't work with the upstream OpenTelemetry Collector Contrib release.

- Enable the optional `system.cpu.utilization` metric for the `cpu` scraper, re-run the collector.

- The utilization metric is not available in the first scrape, as it calculates the utilization in a time period.

- Change `collection_interval` to `5s` to not have to wait the default `1m` for next scrape.

- (Optional) Add `process` scraper.
  - Observe errors.
  - Set all the `mute_...` options to `true`.
  - Observe errors.
  - Run the agent as root: `sudo ./otelcol --config ../scenarios/host-metrics.yaml`.
  - Limit the `process` scraper to only observe `elastic-agent` processes.
  - Without `sudo`, you only get the metrics for the currently running process `./elastic-agent otel`.
  - With `sudo`, you only get the metrics for the globally installed Elastic Agent.

- When to run as root? This isn't specified in docs, although it would probably be useful (contributions welcome!).
  Most of the scrapers read some files in `/proc`. You need to run as root if the files are only readable by root.

### Modifying logs

- Use [Transform processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/transformprocessor/README.md) to modify telemetry.

- [Resource processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/resourceprocessor/README.md), [Attributes processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/attributesprocessor/README.md) offer a simple(r) UX to add/remove/modify resource, record attributes.

- Any suggestions of scenarios?

- Some ideas
  - Extract time from log body
  - Extract severity from log body
  - Extract an attribute from log body

### Modifying metrics

- Use [Transform processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/transformprocessor/README.md) to modify telemetry.

- Any suggestions of scenarios?

- Some ideas
  - Aggregate CPU metrics: [./scenarios/aggregate-cpu-metrics.yaml](./scenarios/aggregate-cpu-metrics.yaml)

### Filtering logs

- Use [Filter processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/filterprocessor/README.md) to filter out telemetry.

- Any suggestions of scenarios?

### Filtering metrics

- Use [Filter processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/filterprocessor/README.md) to filter out telemetry.

- Any suggestions of scenarios?

### Other

- [Multiline logs](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/filelogreceiver/README.md#multiline-configuration)

- ⏯️ Use telemetrygen to generate basic telemetry for testing

- Run the collector in Kubernetes

- ? Use builder to build a custom distro

- ? Run otelcol as systemctl service

### Elastic onboarding - collect logs and metrics from host into Elasticsearch

Essentially follow the steps described in this blog post: <https://andrzej-stencel.github.io/2024/08/28/elastic-distro-with-elastic-cloud.html>
