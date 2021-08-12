---
title: "How to Configure Prometheus with Appmetrics in ASP.NET Core"
date: 2020-06-25T13:48:21+02:00
draft: true
---

One of a super crucial part of each application obviously is metrics, first of all, we need to figure it out what actually metric means and make a distinction between metrics and logging.
I don't want to go through details however metric represents everything related to the numbers, basically, if you have numerical stuff, it would be involved with metrics as well; in other hand loggings come with some human readable information such as "Order was inserted with these parameters lablablab ;)"
I guess it's clear enough right now, so in case of having some sort of numbers as same as request counts, CPU or memory consumptions, etc.

Clarification is enough though, let's dig into codes :)


### What am I supposed to be doing?
I'll be creating an ASP.NET core project, afterward start configuring appmetrics which literally makes life easier for us, then using docker and docker-compose will be running Prometheus as a timeseries database and also Grafana as a visualizer, we are gonna have fun!    

### Let's do it
First, create a simple asp.net core web project using command line (I'm using .net core 3.1 though!)

Install some bunch of packages there:
```
App.Metrics.AspNetCore
App.Metrics.AspNetCore.Endpoints
App.Metrics.AspNetCore.Tracking
App.Metrics.Formatters.Prometheus
```

now you are meant to be adding metric into your startup 
```
services.AddMetrics();
```

and overall you should have something like the below one
```
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace WebCoreDocker
{
    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddMetrics();
            services.AddControllers();
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseRouting();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
    }
}
```

so far so good, half of the job most likely is over ;) pretty straight forward, right? 
The only thing which is left, you need to apply metrics to your host builder in "Program.cs" as well, we gotta change codes a bit there
```
using App.Metrics.AspNetCore;
using App.Metrics.Formatters.Prometheus;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Hosting;

namespace WebCoreDocker
{
    public class Program
    {
        public static void Main(string[] args)
        {
            CreateHostBuilder(args).Build().Run();
        }

        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .UseMetricsWebTracking()
                .UseMetrics(options =>
                {
                    options.EndpointOptions = endpointOptions =>
                    {
                        endpointOptions.MetricsTextEndpointOutputFormatter = new MetricsPrometheusTextOutputFormatter();
                        endpointOptions.MetricsEndpointOutputFormatter = new MetricsPrometheusProtobufOutputFormatter();
                        endpointOptions.EnvironmentInfoEndpointEnabled = true;
                    };
                })
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                });
   
``` 

I would say configuration is clear enough, but as you are able to see, I've added Prometheus text and also protobuf formatter, cool ;)
Tada, Done for now!
run your project and navigate to "/metrics-text", you will be faced with a bunch of key pair values, all values are number for sure!


### What is Prometheus?
Prometheus is a timeseries database which is highly being used as a metrics system with so many rich facilities like:
1) Powerful queries
2) Great Visualization
3) Dimensional Data
4) Efficent storage
and so on, more information here https://prometheus.io/

nevertheless, the fascinating part is Prometheus has a different approach for collecting data, it scrapes from other endpoints and stores information another way around  ;)


### What is Grafana?
Without any doubt Grafana is one of the best tools for visualizing data, creating dashboards, etc.
It can be connected to a variety of databases, prometheus, influxdb, elastic search, ...
I've fallen in love with grafana, it's awesome in my opinion

### How to setup this stuff?
In despite of the fact that you are able to install all these tools individually, I have a better way to be doing that.
A guy called Docker, if you don't know what that is, please do some research regarding this great tool.
either way, I have setup everything using visual studio orchestration tool which is unbelievably nice,
this is my docker-compose.yml (before that add docker container support to your web project! if you are not using visual studio do it manually or considering to your desire IDE, you might be able to configure this stuff automatically either!)
```
version: '3.4'

services:
  webcoredocker:
    image: ${DOCKER_REGISTRY-}webcoredocker
    build:
      context: .
      dockerfile: WebCoreDocker/Dockerfile
  grafana:
    image: "grafana/grafana"
    ports:
      - "3000:3000"
  prom:
    image: "prom/prometheus"
    ports:
      - "9090:9090"
    volumes:
      - "./vols/prometheus.yml:/etc/prometheus/prometheus.yml"
```
it creates 3 isolated containers in same network, my web project, grafana and also prometheus
I've added volume as well because I'm about to change defualt prometheus configuration something like this
```
global:
    scrape_interval:     5s
    evaluation_interval: 5s
  
alerting:
    alertmanagers:
    - static_configs:
      - targets:
  
rule_files:
  
scrape_configs:
    - job_name: 'prometheus'
  
      metrics_path: /metrics-text
      static_configs:
      - targets: ['webcoredocker:80']
```
above prometheus.yml is a tiny configuration of prometheus specially in the last part is saying connect to "http://webcoredocker:80" and also the endpoint in our web app exposed is "/metrics-text"
nice :) now prometheus is able to collect metrics from our endpoint! 
my regards to docker because of how convenient everything is ;) hope you have mentioned that "webcoredocker" comes from my docker-compose!

### Enhance our metrics!
before jumping to grafana and create some super cool dashboards, I prefer adding some more metrics to our web project
Let's create a middleware for detecting memory consumption out, create a class like the below one
```
public class MemoryMetricsMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly IMetrics _metrics;

        public MemoryMetricsMiddleware(RequestDelegate next, IMetrics metrics)
        {
            _next = next;
            _metrics = metrics;
        }

        public Task Invoke(HttpContext httpContext)
        {
            var processPhysicalMemoryGauge = new GaugeOptions
            {
                Name = "Process Physical Memory",
                MeasurementUnit = Unit.Bytes
            };

            var process = Process.GetCurrentProcess();

            _metrics.Measure.Gauge.SetValue(processPhysicalMemoryGauge, process.WorkingSet64);

            return _next(httpContext);
        }
    }
```

and then easily apply this middleware in your startup, it would being called in every request though, but cool for now
```
    app.UseMiddleware<MemoryMetricsMiddleware>();
```

FYI: we have some different types for our metrics
1) Apdex
2) Counter
3) Gauge
4) Histogram
5) Meter
6) Timer

For more information have a look at this page https://www.app-metrics.io/getting-started/metric-types/

For memory consumption I have used gauge, a Gauge is simply an action that returns the instantaneous measure of a value, where the value arbitrarily increases and decreases, for example CPU/Memory usage.


### Let's create a dashboard in Grafana
if you have setup grafana, you are able to connect  to its dashboard using http://localhost:3000 then you can login with username/password "admin".
Afterward, click on "Add data source" and now you are capable to be connecting to a variety of databases, but we would like to choose Prometheus obviously.

![image](/blog/source/prometheus1.PNG)

just I need to remind you to change the url to "http://prom:9090" as you can see as a result of the magic of docker, our services are in same the network, and we don't need to change anything else for now, it's not on production mode :)
scroll down and click on "Save and test", it'll say "Data source is working".

now you are meant to be creating a dashboard, for sure we are able to create customized panels, but before that I would rather show you how everything is convenient, you have this ability to import some predefined dashboards, just click on import, google this text "appmetrics prometheus grafana", you will have found plenty of open source dashboards, and need a number, I'm using this one "2204",
load it and then you gotta have something like this

![image](/blog/source/prometheus2.PNG)

Cool ha? 

We are not over yet, now I would like to show you briefly you can do further more without any doubts, create a panel and then you obliged to change stuff as well as below one

![image](/blog/source/prometheus3.PNG)

now we have applied our customized middleware whose memory consumption metrics are determined in the grafana panel right now.

## Conclusion
1) Not only we can use in our web app, are there tones of prometheus exporters we're able to use
https://prometheus.io/docs/instrumenting/exporters/

2) Try to exploit Grafana and understand how it's practical.

3) Use docker to manage stuff as a piece of cake ;)

Download entire the project here
http://alikhalili.me/blog/source/prometheus.zip

Be safe :)
