# Actions API Request

Firecracker microVMs can execute actions that can be triggered via `PUT`
requests on the `/actions` resource.

Details about the required fields can be found in the
[swagger definition](../../api_server/swagger/firecracker.yaml).

## BlockDeviceRescan

The `BlockDeviceRescan` action is used to trigger a rescan of one of the
microVM's attached block devices. Rescanning is necessary when the size of the
block device's backing file (on the host) changes and the guest needs to
refresh its internal data structures to pick up this change. This action is
therefore only allowed after the guest has booted. Its payload is a string and
represents the ID of the block device that needs to be rescanned, as it was
specified in the `PUT /drives` call that attached it during microVM
configuration. In order for rescanning to work properly, the block device must
not be mounted in the guest at the time of the API call; otherwise, the call
will silently fail - no error is returned from either the guest or the host,
but the guest's internal data structures end up in an inconsistent state.

### BlockDeviceRescan Example

```bash
# Create and set up a block device.
touch ${ro_drive_path}

curl --unix-socket ${socket} -i \
     -X PUT "http://localhost/drives/scratch" \
     -H "accept: application/json" \
     -H "Content-Type: application/json" \
     -d "{
            \"drive_id\": \"scratch\",
            \"path_on_host\": \"${ro_drive_path}\",
            \"is_root_device\": false,
            \"is_read_only\": true
         }"

# Finish configuring and start the microVM. Wait for the guest to boot.

# Resize the block device's backing file and create a filesystem in it.
truncate --size 100M ${ro_drive_path}
mkfs.ext4 ${ro_drive_path}

curl --unix-socket ${socket} -i \
     -X PUT "http://localhost/actions" \
     -H "accept: application/json" \
     -H "Content-Type: application/json" \
     -d "{
            \"action_type\": \"BlockDeviceRescan\",
            \"payload\": \"scratch\"
         }"
```

## InstanceStart

The `InstanceStart` action powers on the microVM and starts the guest OS. It
does not have a payload. It can only be successfully called once.

### InstanceStart Example

```bash
curl --unix-socket ${socket} -i \
     -X PUT "http://localhost/actions" \
     -H "accept: application/json" \
     -H "Content-Type: application/json" \
     -d "{
            \"action_type\": \"InstanceStart\"
         }"
```

## FlushMetrics

The `FlushMetrics` action flushes the metrics on user demand.

### FlushMetrics Example

```bash
curl --unix-socket /tmp/firecracker.socket -i \
    -X PUT "http://localhost/actions" \
    -H  "accept: application/json" \
    -H  "Content-Type: application/json" \
    -d "{
             \"action_type\": \"FlushMetrics\"
    }"
```

## SendCtrlAltDel

This action will send the CTRL+ALT+DEL key sequence to the microVM. By
convention, this sequence has been used to trigger a soft reboot and, as such,
most Linux distributions perform an orderly shutdown and reset upon receiving
this keyboard input. Since Firecracker exits on CPU reset, `SendCtrlAltDel`
can be used to trigger a clean shutdown of the microVM.

For this action, Firecracker emulates a standard AT keyboard, connected via an
i8042 controller. Driver support for both these devices needs to be present in
the guest OS. For Linux, that means the guest kernel needs
`CONFIG_SERIO_I8042` and `CONFIG_KEYBOARD_ATKBD`.

**Note1**: at boot time, the Linux driver for i8042 spends
a few tens of milliseconds probing the device. This can be disabled by using
these kernel command line parameters:

```i8042.noaux i8042.nomux i8042.nopnp i8042.dumbkbd```

**Note2** This action is only supported on `x86_64` architecture.

### SendCtrlAltDel Example

```bash
curl --unix-socket /tmp/firecracker.socket -i \
    -X PUT "http://localhost/actions" \
    -H  "accept: application/json" \
    -H  "Content-Type: application/json" \
    -d "{
             \"action_type\": \"SendCtrlAltDel\"
    }"
```
