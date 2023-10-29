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