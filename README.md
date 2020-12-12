# Panoptes Monitor Schema

This schema is intended to standardize requests and responses for a client to interact
with a Panoptes monitoring plugin. The overall idea is that any server that implements
the monitor schema can then interact with any client that does the same.
The specification can be found [here](spec.md).

## Use Cases

### Snakemake integration

The workflow manager [snakemake](https://snakemake.github.io/) can accept an argument,
`--wms-monitor`, that points to an endpoint (url and port) that is expected to implement
the monitor schema, meaning a set of endpoints along with expected data to send
and receive from them. By way of having these endpoints served by [panoptes](https://github.com/panoptes-organization/panoptes)
snakemake can then send data to known endpoints to allow the tool to monitor progress.

## Snakemake Interface (snakeface) Integration

Currently under development is the Snakemake Interface (repository currently private)
that also uses snakemake with `--wms-monitor` under the hood to deploy snakemake
workflows that update a monitoring endpoint. In this case, however, the endpoints
are served by the snakeface iterface itself, essentially being used as a hook
to update models and statuses to show the user.

## Getting Started

### Creating a monitor server

If you want to create some kind of server to interact with a tool like snakemake,
you should ensure that it provides the endpoints defined in the [spec](spec.md),
and that it expects to receive data formatted as described as well. 

### Creating a monitor client

If you want to add monitoring to some client or tool to interact with a server
that implements the monitor schema, then you should ensure that your client
uses the endpoints defined in [spec](spec.md) and sends the minimal required
data to them.
