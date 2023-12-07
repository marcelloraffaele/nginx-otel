# Intro
With the release R29, Nginx Plus announced some enhanced features ( more info at Announcing NGINX Plus R29 - NGINX ) the one that I want to talk about is the Native OpenTelemetry.
OpenTelemetry is a collection of tools, APIs, and SDKs that can be used to instrument, generate, collect, and export telemetry data (metrics, logs, and traces) to help you analyze your software's performance and behavior.

## useful links

[nginx-plus Native_OpenTelemetry](https://www.nginx.com/blog/nginx-plus-r29-released/#_Native_OpenTelemetry)
[Docs otel module](https://nginx.org/en/docs/ngx_otel_module.html)

## Architecture
![zipkin-traces](/images/demo-architecture.png)

## How to build
First of all we need to build and Nginx Plus image and install on it the ngx_otel_module. To this this is needed a NGINX Plus subscription (purchased or trial). You can ask a [trial from here](https://www.nginx.com/free-trial-request/).
To create a Docker image put the Dockerfile, nginx-repo.crt, and nginx-repo.key files in the same directory and run the following command:

```bash
cd build
docker build --no-cache -t nginxplus --secret id=nginx-crt,src=your_cert_file --secret id=nginx-key,src=your_key_file .
```

## how to test
the project is configured with docker compose.
You can clone locally this project:
```bash
git clone https://github.com/marcelloraffaele/nginx-otel.git
```

To run the project, run the following command:

```bash
docker compose up -d
```

it will create 4 containers, with two nginx plus instances, one for an api gateway and one for a time service and a Otel collector and a Zipkin instances.

```bash
Creating network "nginx-otel_default" with the default driver
Creating zipkin ... done
Creating nginx-otel_collector_1 ... done
Creating nginx-otel_backend_1   ... done
Creating nginx-otel_api-gateway_1 ... done
```


## TEST:

First of all we open a browser at the zipkin page: [http://localhost:9411/](http://localhost:9411/)

After we create some traffic:
```bash
for i in {1..5}
do
    curl http://localhost:8080/uuid
    curl http://localhost:8080/time
done

```

we can se the traces from the zipkin webpage:
### Traces
![zipkin-traces](/images/zipkin-traces.png)

### Trace details
![zipkin-trace-details](/images/zipkin-trace-details.png)

### dependencies
![zipkin-dependencies](/images/zipkin-dependencies.png)


## Shutdown the test
To destroy all the resources:
```bash
docker-compose down
```

# Forward the Traces to Azure App Insights
As mentioned in the previous article [Nginx Plus Monitoring and Tracing: Harnessing the Power of OpenTelemetry](https://medium.com/p/65477020d864), using OpenTelemetry provides an opportunity to extend and enhance our architecture to integrate with Cloud Services. In this post, we will explore how it is possible to forward the traces that our application creates to Azure Application Insights. The updated architecture will look like this:
![forward-app-insights-architecture](/images/forward-app-insights/architecture.png)

As illustrated, from the OpenTelemetry (Otel) collector, traces can be forwarded to one or more compatible destinations. Currently, the core distribution of [OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector/tree/main) does not include an exporter for Azure Application Insights. For this reason, we will use the [OpenTelemetry Collector contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main), which, in addition to core components (such as Prometheus, Jaeger), offers many other components. One of these components is the [Azure Monitor Exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/azuremonitorexporter).

With the Azure Monitor Exporter, we can forward the metrics to Azure Application Insights and visualize the traces from the Azure Portal. Let's get started!

# Create an Application Insights
Since this is a demo, I created a resource group named `demo-otel` to facilitate the quick deletion of everything at the end of the test. Now, we can proceed to create the Azure Application Insights resource:
![forward-app-insights-architecture](/images/forward-app-insights/azure-create-app-insights.png)

Once the resource creation is complete, we can retrieve the `Connection String` from the Property menù:

![forward-app-insights-architecture](/images/forward-app-insights/azure-app-insights-connection-string.png)

# Change the Configuration
To implement this, we need to modify the Docker Compose configuration by changing the image of the OpenTelemetry Collector to an `OpenTelemetry Collector contrib` image,
```yaml
collector:
    image: otel/opentelemetry-collector-contrib:0.90.1
```

I created another docker compose file named `docker-compose-app-insights.yml`:

```yaml
version: '3'
services:
  zipkin:
    image: openzipkin/zipkin:2.24
    container_name: zipkin
    environment:
      - STORAGE_TYPE=mem
    ports:
      # Port used for the Zipkin UI and HTTP Api
      - 9411:9411
  collector:
    image: otel/opentelemetry-collector-contrib:0.90.1
    command: ['--config=/etc/otel-collector-config.yaml']
    volumes:
      - ./otel-collector-config-app-insights.yaml:/etc/otel-collector-config.yaml
    depends_on:
      - zipkin
  api-gateway:
    image: nginx-plus:r29
    ports:
      - "8080:80"
    volumes:
      - ./api-gtw/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - collector
      - backend
  backend:
    image: nginx-plus:r29
    ports:
      - "8081:80"
    volumes:
      - ./backend/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - collector
```
To facilitate the comparison of traces on the Azure Portal with local traces, I've removed the Zipkin instance. The updated Docker Compose file now utilizes a new `otel-collector-config` file named `otel-collector-config-app-insights.yaml`.

Within this new configuration file, I've made additions to the `exporter` section to include the `azuremonitor` configuration:
```yaml
  azuremonitor:
    connection_string: "InstrumentationKey=00000000-0000-0000-0000-000000000000;IngestionEndpoint=https://ingestion.azuremonitor.com/"
```

Additionally, I've extended the `traces.exporters` section to incorporate `azuremonitor`:
```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [zipkin/nontls, azuremonitor]
```
With these adjustments, the OpenTelemetry (Otel) Collector will now send collected traces to Zipkin locally and simultaneously to our Application Insights.

## Test
To run the project, specify the different `docker-compose-app-insights.yml` file using the following command:

```bash
docker-compose -f docker-compose-app-insights.yml up -d
```

Generate some traffic with the following commands:
```bash
for i in {1..5}
do
    curl http://localhost:8080/uuid
    curl http://localhost:8080/time
done
```

Visit the Azure portal to observe the traces that have arrived:
![app-insights-demo-1](/images/forward-app-insights/azure-app-insights-1.png)

For a more in-depth analysis of API and application performance, navigate to:
![app-insights-demo-2](/images/forward-app-insights/azure-app-insights-2.png)

If you prefer to check locally on Zipkin, you will find the same data:
![forward-app-insights-architecture](/images/forward-app-insights/zipkin-demo.png)

Once the test is complete, shut down the Docker Compose:
```bash
docker-compose -f docker-compose-app-insights.yml down
```

Finally, delete the Resource group to free up resources.

## Conclusion
This post illustrates how to configure the OpenTelemetry Collector to forward traces to Azure Application Insights. OpenTelemetry is highly flexible and can be employed to send traces to various Cloud Services. As an open-source project, it allows the creation of ad-hoc exporters to expand the catalog of components for different technologies. While this version of the collector is open source and not directly supported by the provider, it remains valuable for many use cases.
Forwarding traces to a cloud service like Azure Application Insights can be incredibly useful for consolidating traces from various locations within our architecture. From Application Insights, these traces can be analyzed, providing the opportunity for end-to-end analysis. This allows for measuring performance and identifying bottlenecks.