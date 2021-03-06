# Introduction

The demo7 directory is a work in progress.  The P4_16 program compiles
without error, but that is about all that is known to work so far.

The intent is to demonstrate how to configure multicast groups from
the P4Runtime API, and/or the simple_switch_CLI, and process packets
in simple_switch_grpc, some of which are forwarded via the v1model
architecture unicast capability, and others of which are forwarded via
the v1model architecture multicast capability.


# Compiling

See [README-using-bmv2.md](../README-using-bmv2.md) for some things
that are common across different P4 programs executed using bmv2.

To compile the P4_16 version of the code:

    p4c --target bmv2 --arch v1model --p4runtime-files demo7.p4info.txt demo7.p4
                                                                        ^^^^^^^^ source code

If you see an error message about `mark_to_drop: Passing 1 arguments
when 0 expected`, then see
[`README-troubleshooting.md`](../README-troubleshooting.md#compiler-gives-error-message-about-mark_to_drop)
for what to do.

Running that command will create these files:

    demo7.p4i - the output of running only the preprocessor on the P4
        source program.
    demo7.json - the JSON file format expected by BMv2 behavioral
        model `simple_switch_grpc`.
    demo7.p4info.txt - the text format of the file that describes
        the P4Runtime API of the program.

Only the last two files are needed to run your P4 program.  You can
ignore the file with suffix `.p4i` unless you suspect that the
preprocessor is doing something unexpected with your program.

There is no P4_14 version of this program.  There is no particular
reason why not, other than lack of interest on my part in writing the
other version.  It should be straightfoward to write one if you have
seen how the P4_14 and P4_16 versions of other demo programs like
demo1 and demo2 are related to each other.


# One-time setup

See the section with this name in
[`../demo1/README-p4runtime.md`](../demo1/README-p4runtime.md).


# Running simple_switch_grpc

See the section with this name in
[`../demo1/README-p4runtime.md`](../demo1/README-p4runtime.md).


# Using a Python interactive session as a controller

To start an interactive Python session that can be used to load the
compiled P4 program into the running `simple_switch_grpc` process, and
install table entries:

```bash
cd p4-guide/demo7
export PYTHONPATH="`realpath ../pylib`:$PYTHONPATH"
python

# NOTE: For most interactive Python sessions, typing Ctrl-D or the
# command `quit()` is enough to quit Python and go back to the shell.
# For this Python session, one or more of the commands below cause
# this interactive session to 'hang' if you try that.  In the most
# commonly used Linux/OSX shells you can type Ctrl-Z to put the Python
# process in the background and return to the shell prompt.  You may
# want to kill the process, e.g. using `kill -9 %1` in bash.
```

Enter these commands at the `>>> ` prompt of the Python session:

```python
# Note: 50051 is the default TCP port number on which the
# simple_switch_grpc process is listening for connections.

my_dev1_addr='localhost:50051'
my_dev1_id=0
import base_test as bt

# Convert the BMv2 JSON file demo7.json, created by the p4c
# compiler, into the binary format file demo7.bin expected by
# simple_switch_grpc.  We will send this binary file to
# simple_switch_grpc over the P4Runtime connection.

bt.bmv2_json_to_device_config('demo7.json', 'demo7.bin')

# Load the binary version of the compiled P4 program, and the text
# version of the P4Runtime info file, into the simple_switch_grpc
# 'device'.

bt.update_config('demo7.bin', 'demo7.p4info.txt', my_dev1_addr, my_dev1_id)
# Result of successful bt.update_config() call is "P4Runtime
# SetForwardingPipelineConfig" output from simple_switch_grpc log.

h=bt.P4RuntimeTest()
h.setUp(my_dev1_addr, 'demo7.p4info.txt')
# Result of successful h.setUp() call is "New connection" output from
# simple_switch_grpc log.
```

Note: Unless the `simple_switch_grpc` process crashes, or you kill it
yourself, you can continue to use the same running process, loading
different compiled P4 programs into it over time, repeating the
`bt.update_config` call above with the same or different file names.


```python
# The full names of the tables and actions begin with 'ingress.' or
# 'egress.', but some layer of software between here and there adds
# these prefixes for you, as long as the table/action name is unique
# after that point.

# The tables all have 'const default_action = <action_name>' in the
# P4_16 program, so the control plane is not allowed to change them.
# Do not try.  I have, and you get an appropriate error message if you
# try.

# add new non-default table entries by filling in at least one key field

# bt.ipv4_to_binary takes an IPv4 address written as a string in
# dotted decimal notation, e.g. '10.1.2.3', and converts it to a
# string with binary contents expected by the table add operation.

# bt.mac_to_binary is similar, but takes a MAC address as a string,
# with colons separating each byte, specified in hex,
# e.g. '00:de:ad:be:ef:ff'.

# bt.int2string takes a non-negative integer 'n' as the first
# parameter, and a positive integer 'width_in_bits' as the second
# parameter.  It returns a string with binary contents expected by the
# Python P4Runtime client operations.  If 'n' does not fit in
# 'width_in_bits' bits, an exception is raised.

# Note: I have tried adding table entries using calls to bt.int2string
# that had fewer bits than shown in the calls below, and got back an
# error from the P4Runtime server, I think with "INVALID_ARGUMENT" in
# the printed error message at the client (from memory).  Apparently
# the current simple_switch_grpc code is not permissive in the size of
# the encoding of these numbers?  If so, it would be nice to have some
# code that would automatically determine the necessary width from the
# P4 Info file, without a user having to look it up and do it
# themselves.

# Define a few small helper functions that help construct parameters
# for the function table_add()

def ipv4_mc_route_lookup_key(h, ipv4_addr_string, prefix_len):
    return ('ipv4_mc_route_lookup', [h.Lpm('hdr.ipv4.dstAddr',
                            bt.ipv4_to_binary(ipv4_addr_string), prefix_len)])

def set_mcast_grp(mcast_grp_int_val):
    return ('set_mcast_grp', [('mcast_grp', bt.int2string(mcast_grp_int_val, 16))])

def ipv4_da_lpm_key(h, ipv4_addr_string, prefix_len):
    return ('ipv4_da_lpm', [h.Lpm('hdr.ipv4.dstAddr',
                            bt.ipv4_to_binary(ipv4_addr_string), prefix_len)])

def set_l2ptr(l2ptr_int_val):
    return ('set_l2ptr', [('l2ptr', bt.int2string(l2ptr_int_val, 32))])

def mac_da_key(h, l2ptr_int_val):
    return ('mac_da', [h.Exact('meta.fwd_metadata.l2ptr',
                       bt.int2string(l2ptr_int_val, 32))])

def set_dmac_intf(dmac_string, intf_int_val):
    return ('set_dmac_intf',
            [('dmac', bt.mac_to_binary(dmac_string)),
             ('intf', bt.int2string(intf_int_val, 9))])

def send_frame_key(h, port_int_val):
    return ('send_frame', [h.Exact('stdmeta.egress_port',
                           bt.int2string(port_int_val, 9))])

def rewrite_mac(smac_string):
    return ('rewrite_mac', [('smac', bt.mac_to_binary(smac_string))])

h.table_add(ipv4_mc_route_lookup_key(h, '224.3.3.3', 32), set_mcast_grp(91))
h.table_add(ipv4_da_lpm_key(h, '10.1.0.1', 32), set_l2ptr(58))
h.table_add(mac_da_key(h, 58), set_dmac_intf('02:13:57:ab:cd:ef', 2))

h.table_add(send_frame_key(h, 0), rewrite_mac('00:de:ad:00:00:ff'))
h.table_add(send_frame_key(h, 1), rewrite_mac('00:de:ad:11:11:ff'))
h.table_add(send_frame_key(h, 2), rewrite_mac('00:de:ad:22:22:ff'))
h.table_add(send_frame_key(h, 3), rewrite_mac('00:de:ad:33:33:ff'))
h.table_add(send_frame_key(h, 4), rewrite_mac('00:de:ad:44:44:ff'))
h.table_add(send_frame_key(h, 5), rewrite_mac('00:de:ad:55:55:ff'))

# When a packet is sent from ingress to the packet buffer with
# mcast_grp=91, configure the (egress_port, instance) places to which
# the packet will be copied.

# The first parameter is the mcast_grp value.

# The second is a list of 2-tuples.  The first element of each
# 2-tuples is the egress port to which the copy should be sent, and
# the second is the "replication id", also called "egress_rid" in the
# P4_16 v1model architecture standard_metadata struct, or "instance"
# in the P4_16 PSA architecture psa_egress_input_metadata_t struct.
# That value can be useful if you want to send multiple copies of the
# same packet out of the same output port, but want each one to be
# processed differently during egress processing.  If you want that,
# put multiple pairs with the same egress port in the replication
# list, but each with a different value of "replication id".

h.pre_add_mcast_group(91, [(2, 5), (5, 75), (1, 111)])
```

I do not have Python code right now that uses the P4Runtime API to
read the multicast groups configured on `simple_switch_grpc`.  In the
mean time, you can run the `simple_switch_CLI` command in a separate
terminal window and use its commands to read table entries and/or the
multicast configuration state.

```bash
simple_switch_CLI
```

At the `simple_switch_CLI` prompt `RuntimeCmd: `, type the command
`mc_dump` to see output like this:

```
RuntimeCmd: mc_dump
==========
MC ENTRIES
**********
mgrp(91)
  -> (L1h=0, rid=111) -> (ports=[1], lags=[])
  -> (L1h=1, rid=75) -> (ports=[5], lags=[])
  -> (L1h=2, rid=5) -> (ports=[2], lags=[])
==========
LAGS
==========
```

The `mgrp(91)` shows the `mcast_grp` value of 91 configured above.
Each of the next 3 lines of output below that show a list of ports,
one line for each replication id value.  I do not know precisely what
the `L1h` values represent.  Likely they are unique id values selected
inside of `simple_switch_grpc` controller interface code for
identifying parts of the multicast group configuration.  I believe
they are invisible to the P4Runtime API, and a P4Runtime controller
need not concern itself with them.


# Scapy session for sending packets

```bash
$ sudo scapy
```

The packet below should match the example table entry created above in
the `ipv4_mc_route_lookup` table, finish its ingress processing with
`standard_metadata.mcast_grp` equal to 91, and be replicated to the 3
configured output ports.

```python
pkt1=Ether(src='00:00:c0:01:d0:0d', dst='ff:ff:ff:ff:ff:ff') / IP(src='10.2.3.5', dst='224.3.3.3') / UDP()
sendp(pkt1, iface='veth2')
```
