# springxd_exporter
[https://github.com/riclib/springxd_exporter](https://github.com/riclib/springxd_exporter)

# Purpose
The *springXD_exporter* allows monitoring of springxd in solidmon/grafana*.*
- exposes openmetrics about executions on a springxd server
- saves a log of all executions it observes in json
- exposes a realtime endpoint that can be easily used from grafana to see current job status without slowing springxd

# Usage
```yaml
usage: springXD_exporter [<flags>]

Flags:
  -h, --help                     Show context-sensitive help (also try --help-long and --help-man).
      --config.file=CONFIG.FILE   exporter configuration file.
      --lokioutput.file=LOKIOUTPUT.FILE  
                                 File to write loki output to
      --web.listen-address=":9098"  
                                 The address to listen on for HTTP requests.
      --config.check             If true validate the config file and then exit.
      --encrypt.password=ENCRYPT.PASSWORD  
                                 Password to encrypt
      --log.level=info           Only log messages with the given severity or above. One of: [debug, info, warn, error]
      --log.format=logfmt        Output format of log messages. One of: [logfmt, json]
      --version                  Show application version.
```

# Exporting Logic

When the adapter boots it uses the stream definitions api to get the last execution of each stream, susequently it efficiently tails the executions and updates the data structures, metrics and logs.

While a job is running the exporter reports metrics on it, logging to file when there are status changes or new jobs.

# Metrics List

This exporter reports both metrics and any needed mappings for string statuses.

``` bash
# HELP Will be zero if the server couldn\t be reached or timed out
springxd_scrape_success 1.0

# HELP Job Execution Create Time. Unix Epoch ms
springxd_execution_create_time_ms{name="xxx"} 1.653909541049e+12

# HELP  Job Execution End Time. Unix Epoch ms
springxd_execution_end_time_ms{name="xxx"} 1.653909570592e+12

# HELP  Job Execution Id. 
springxd_execution_execution_id{name="xxx"} 936519

# HELP  Is the Job running
springxd_execution_is_running{name="xxx"} 0

# HELP  Job Execution Last Update. Unix Epoch ms
springxd_execution_last_update_ms{name="xxx"} 1.65390959265e+12

# HELP  Job Execution Start Time . Unix Epoch ms
springxd_execution_start_time_ms{name="xxx"} 1.653909541053e+12

# HELP Execution Status. Uses config:springxd:execution_status as map
springxd_execution_status{name="xxx"} 0

# HELP  Number of executions scanned
springxd_execution_step_count{name="xxx"} 6

# HELP Mapping between status names and values for springXD
# TYPE config:springxd:execution_status gauge
config:springxd:execution_status{status="COMPLETED"} 0
config:springxd:execution_status{status="FAILED"} 1
config:springxd:execution_status{status="STOPPED"} 2
config:springxd:execution_status{status="UNKNOWN"} 3
```

## Config Metrics

`config:springxd:execution_status:` maps between `status` and float values

```bash
# HELP config:springxd:execution_status Mapping between status names and values for springXD
# TYPE config:springxd:execution_status gauge
config:springxd:execution_status{status="COMPLETED"} 0
config:springxd:execution_status{status="FAILED"} 1
config:springxd:execution_status{status="STOPPED"} 2
config:springxd:execution_status{status="UNKNOWN"} 3
```

- If any new custom statuses are found they are mapped incrementally
    
```bash
# HELP config:springxd:execution_status Mapping between status names and values for springXD
# TYPE config:springxd:execution_status gauge
config:springxd:execution_status{status="COMPLETED"} 0
config:springxd:execution_status{status="CUSTOM"} 4
config:springxd:execution_status{status="FAILED"} 1
config:springxd:execution_status{status="STOPPED"} 2
config:springxd:execution_status{status="UNKNOWN"} 3
# HELP springXD_springxd_execution_status Execution Status. Uses config:springxd:execution_status as map
# TYPE springXD_springxd_execution_status gauge
springXD_springxd_execution_status{execution_id="1580957"} 0
springXD_springxd_execution_status{execution_id="1580958"} 0
springXD_springxd_execution_status{execution_id="1580959"} 0
springXD_springxd_execution_status{execution_id="1580960"} 0
springXD_springxd_execution_status{execution_id="1580961"} 4
```

# Endpoints
`/probe?target=xxx` → return metrics

`/now?target=xxx` → return metrics for currently running jobs, flattened from cached metrics from last time probe was called. To be used for building real time dashboards.

```json
[
    {
        "name": "JOB1",
        "execution_id": 936261,
        "status": "COMPLETED",
        "status_mapped": 0,
        "schedule": "0 0 14 ? * SUN",
        "start_time": 1653804004091,
        "end_time": 1653804032425,
        "create_time": 1653804004081,
        "last_updates": 1653804033446,
        "region": "ap"
    },
    {
        "name": "JOB2",
        "execution_id": 936996,
        "status": "COMPLETED",
        "status_mapped": 0,
        "schedule": "0 5 2 * * ?",
        "start_time": 1654106708629,
        "end_time": 1654106772094,
        "create_time": 1654106708618,
        "last_updates": 1654106809240,
        "region": "ap"
    }
]
```

# Configuration
To configure the adapter create a yaml file and pass it as `--config.file=your_file.yml`
```json
page:
  size: 10
  max: 3
cache:
  duration: 24h
  expiry: 24h
targets:
  na:
    endpoint: http://localhost:9393
    username: user
    encrypted_password: kcmdiywxHvvXjxrq
    labels:
      region: na
```

To obtain an encrypted password call the exporter with the parameter ecrypt.password=password

```bash
$ bin/springXD_exporter --encrypt.password="Hello World"
kcmdiywxHvvXjxrq
```
you can also pass an unencrypted password, just use the key password instead of encrypted_password. This should only be used for development, never in production.

```yaml
targets:
  na:
    endpoint: http://localhost:9393
    username: user
    password: Plain_Text_Password
```
> You can add as many labels as you wish, they will be added to the loki export.
```yaml
    labels:
      region: na
      another: label 
```

depending on the number of jobs running per minute on the server you should configure the keys `page/size` and `page/max`. 

To provide real time data the exporter uses an in memo0ry cach with can be configures under the `cache` key. defaults should work well for most situations.

# API’s Used

## Tailing Job Executions
```bash
/jobs/executions?size=5&page=0
```
with parameters 
- size - size of page
- page - number of page

## Getting executions for a job
```bash
/jobs/executions?jobname={jobName}
```

## SpringXD API Documentation
[Spring XD Guide](https://docs.spring.io/spring-xd/docs/current-SNAPSHOT/reference/html/#REST-API)