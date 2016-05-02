# mongotape
##### Purpose

`mongotape` is a traffic capture and replay tool for MongoDB. It can be used to inspect commands being sent to a MongoDB instance, record them, and replay them back onto another host at a later time.
##### Use cases
- Preview how well your database cluster would perform a production workload under a different environment (storage engine, index, hardware, OS, etc.)
- Reproduce and investigate bugs by recording and replaying the operations that trigger them 
- Inspect the details of what an application is doing to a mongo cluster (i.e. a more flexible version of [mongosniff](https://docs.mongodb.org/manual/reference/program/mongosniff/))

## Quickstart

Make a recording:

    mongotape record -i lo0 -e "port 27017" -p playback.bson
Analyze it:

    mongotape stat -p playback.bson --report playback_stats.json
Replay it against another server, at 2x speed:

    mongotape play -p playback.bson --speed=2.0 --report replay_stats.json --host 192.168.0.4:27018

## Detailed Usage

Basic usage of `mongotape` works in two phases: `record` and `play`. Analyzing recordings can also be performed with the `stat` command.
* The `record` phase takes a [pcap](https://en.wikipedia.org/wiki/Pcap) file (generated by `tcpdump`) and analyzes it to produce a playback file (in BSON format). The playback file contains a list of all the requests and replies to/from the Mongo instance that were recorded in the pcap dump, along with their connection identifier, timestamp, and other metadata.
* The `play` reads in the playback file that was generated by `record`, and re-executes the workload against some target host. 
* The `stat` command reads a playback file and analyzes it, detecting the latency between each request and response. 

#### Capturing TCP (pcap) data

To create a recording of traffic, use the `record` command as follows:

    mongotape record -i lo0 -e "port 27017" -p recording.bson
    

This will record traffic on the network interface `lo0` targeting port 27017.
The options to `record` are:
* `-i`: The network interface to listen on, e.g. `eth0` or `lo0`. You may be required to run `mongotape` with root privileges for this to work.
* `-e`: An expression in Berkeley Packet Filter (BPF) syntax to apply to incoming traffic to record. See http://biot.com/capstats/bpf.html for details on how to construct BPF expressions.
* `-p`: The output file to write the recording to.

#### Recording a playback file from pcap data

Alternatively, you can capture traffic using `tcpdump` and create a recording from a static PCAP file. First, capture TCP traffic on the system where the workload you wish to record is targeting. Then, run `mongotape record` using the `-f` argument (instead of `-i`) to create the playback file.

    sudo tcpdump -i lo0 -n "port 27017" -w traffic.pcap

    $ ./mongotape record -f traffic.pcap -p playback.bson

Using the `record` command of mongotape, this will process the .pcap file to create a playback file. The playback file will contain everything needed to re-execute the workload.

### Using playback files

There are several useful operations that can be performed with the playback file.

##### Re-executing the playback file
The `play` command takes a playback file and executes the operations in it against a target host.

    ./mongotape play -p playback.bson --host mongodb://target-host.com:27017
    
To modify playback speed, add the `--speed` command line flag to the `play` command. For example, `--speed=2.0` will run playback at twice the rate of the recording, while `--speed=0.5` will run playback at half the rate of the recording.

    mongotape play -p workload.playback --host staging-mongo-cluster-hostname

###### Playback speed
You can also play the workload back at a faster rate by adding the --speed argument; for example, --speed=2.0 will execute the workload at twice the speed it was recorded at. 

###### Logging metrics about execution performance during playback
Use the `--report=<path-to-file>` flag to save  detailed metrics about the performance of each operation performed during playback to the specified json file. This can be used in later analysis to compare performance and behavior across  different executions of the same workload.

##### Inspecting the operations in a playback file

The `stat` command takes a static workload file (bson) and generates a json report, showing each operation and some metadata about its execution. The output is in the same format as that used by the json output generated by using the `play` command with `--report`.

###### Report format

The data in the json reports consists of one record for each request/response. Each record has the following format:
```json
{
    "connection_num": 1,
    "latency_us": 89,
    "ns": "test.test",
    "op": "getmore",
    "order": 16,
    "play_at": "2016-02-02T16:24:16.309322601-05:00",
    "played_at": "2016-02-02T16:24:16.310908311-05:00",
    "playbacklag_us": 1585
}             
```

The fields are as follows:
 * `connection_num`: a key that identifies the connection on which the request was executed. All requests/replies that executed on the same connection will have the same value for this field. The value for this field does *not* match the connection ID logged on the server-side.
 * `latency_us`: the time difference (in microseconds) between when the request was sent by the client, and a response from the server was received.
 * `ns`: the namespace that the request was executed on.
 * `op`: the type of operation represented by the request - e.g. "query", "insert", "command", "getmore"
 * `order`: a monotonically increasing key indicating the order in which the operations were recorded and played back. This can be used to reconstruct the ordering of the series of ops executed on a connection, since the order in which they appear in the report file might not match the order of playback.
 * `data`: the payload of the actual operation. For queries, this will contain the actual query that was issued. For inserts, this will contain the documents being inserted. For updates, it will contain the query selector and the update modifier, etc.
 * `play_at`: The time at which the operation was supposed to be executed.
 * `played_at`: The time at which the `play` command actually executed the operation.
 * `playbacklag_us`: The difference (in microseconds) in time between `played_at` and `play_at`. Higher values generally indicate that the target server is not able to keep up with the rate at which requests need to be executed according to the playback file.
