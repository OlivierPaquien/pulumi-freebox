# pulumi-provider-freebox

Native Pulumi provider for Freebox, ported from [terraform-provider-freebox](https://github.com/NikolaLohinski/terraform-provider-freebox) **without** using the Terraform Bridge. Implemented from scratch with the [Pulumi Go Provider SDK](https://www.pulumi.com/docs/iac/guides/building-extending/providers/sdks/pulumi-go-provider-sdk/).

## Resources

- **freebox:fw:PortForwarding** – Port forwarding rule (NAT)
- **freebox:virtual:Disk** – Virtual disk (qcow2/raw) on the Freebox
- **freebox:virtual:Machine** – Virtual machine on the Freebox
- **freebox:downloads:File** – Download a file from a URL to the Freebox

## Functions (invokes)

- **freebox:api:getApiVersion** – Freebox API discovery (version, model, etc.)
- **freebox:virtual:getVirtualDisk** – Virtual disk information (type, sizes)

## Configuration

Environment variables or Pulumi config:

| Option       | Env              | Description                                    |
|-------------|------------------|------------------------------------------------|
| `endpoint`  | `FREEBOX_ENDPOINT`  | Freebox URL (default: http://mafreebox.freebox.fr) |
| `apiVersion`| `FREEBOX_VERSION`   | API version (default: latest)                 |
| `appId`     | `FREEBOX_APP_ID`    | Freebox API application ID                   |
| `token`     | `FREEBOX_TOKEN`     | API authentication token                      |

App authorization is done via the [Freebox API](https://dev.freebox.fr/sdk/login/). You can use the Terraform provider’s `authorize` command to obtain `app_id` and `token`.

**Debug**: Pulumi hides plugin output. The provider also writes to a file so you can follow logs during a `pulumi destroy`:

- **Default**: it first tries `/tmp/pulumi-freebox-provider.log`, then `$HOME/.pulumi/pulumi-freebox-provider.log` if `/tmp` is not writable (e.g. plugin running in a sandbox). On Windows: `%TEMP%\pulumi-freebox-provider.log`.
- **Custom**: set `FREEBOX_DEBUG_LOG=/path/to/file` (if Pulumi passes env to the plugin).

```bash
# In one terminal: destroy
pulumi destroy

# In another: follow logs (or ~/.pulumi/pulumi-freebox-provider.log if /tmp fails)
tail -f /tmp/pulumi-freebox-provider.log
# or
tail -f ~/.pulumi/pulumi-freebox-provider.log
```

Optional: `FREEBOX_DEBUG_HTTP=1` to trace HTTP request/response details (also written to the log file).

**To see logs during `pulumi destroy`**: Pulumi runs the provider from its cache (or from the configured path). After each `go build`, reinstall the plugin so the correct binary is used, and force a known log file with `FREEBOX_DEBUG_LOG` (Pulumi passes env vars to the plugin):

```bash
pulumi plugin install resource freebox 0.1.1 --file ./bin/pulumi-resource-freebox
FREEBOX_DEBUG_LOG=$HOME/freebox-provider.log pulumi destroy
# in another terminal: tail -f $HOME/freebox-provider.log
```

The log file also contains a `[freebox] log file: ...` line at startup to confirm where it is writing.

## Requirements

- Go 1.24+

## Build

```bash
go build -o bin/pulumi-resource-freebox .
```

The binary must be named `pulumi-resource-freebox` to be recognized by Pulumi.

## Tests

### Unit tests

Provider unit tests use a mock HTTP server and do not require a Freebox. Run them with:

```bash
go test -v -run TestProvider_ .
```

They cover invalid endpoint handling and the `getApiVersion` invoke with different configs (env and block).

### Integration tests

Integration tests (PortForwarding, RemoteFile, and optionally VirtualMachine) use the real Freebox API. They are skipped if `FREEBOX_TOKEN` is not set.

- **PortForwarding**: create/delete, create/update/delete + API verification (CheckDestroy).
- **RemoteFile**: download a small file then delete + verify the file no longer exists.
- **VirtualMachine**: run only when `FREEBOX_TEST_DISK_PATH` is set (path to an existing qcow2 on the Freebox, e.g. `Freebox/VMs/alpine.qcow2`). Create (stopped) then delete.

Optional: `FREEBOX_ROOT` (default `Freebox`) for the base directory used by RemoteFile tests.

```bash
export FREEBOX_APP_ID=your_app_id
export FREEBOX_TOKEN=your_token
# optional for RemoteFile:
export FREEBOX_ROOT=Freebox
# optional for VM tests:
export FREEBOX_TEST_DISK_PATH=Freebox/VMs/alpine.qcow2
go test -v -run TestProvider .
```

## Usage (YAML)

Example with Pulumi YAML pointing to the local binary:

```yaml
name: freebox-example
runtime: yaml
config:
  freebox:endpoint: http://mafreebox.freebox.fr
  freebox:appId: "your_app_id"
  freebox:token: secret:your_token

resources:
  pf:
    type: freebox:fw:PortForwarding
    properties:
      enabled: true
      ipProtocol: tcp
      portRangeStart: 22
      targetIp: "192.168.1.10"
      comment: "SSH"
```

## Dependencies

- [free-go](https://github.com/NikolaLohinski/free-go) – Freebox API client
- [pulumi-go-provider](https://github.com/pulumi/pulumi-go-provider) – Pulumi Go provider SDK

## License

See [LICENSE](LICENSE).
