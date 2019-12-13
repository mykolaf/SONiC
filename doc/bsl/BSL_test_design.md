# BSL test design

- [BSL test design](#bsl-test-design)
- [Overview](#overview)
  - [Scope](#scope)
- [Setup configuration](#setup-configuration)
- [Test cases](#test-cases)
  - [Pytest infrastructure changes](#pytest-infrastructure-changes)
  - [Setup of fanout switch](#setup-of-fanout-switch)
  - [Setup of VMs](#setup-of-vms)
- [Test cases](#test-cases-1)
  - [Test case #1 - basic sanity](#test-case-1---basic-sanity)
    - [Test objective](#test-objective)
    - [Script used](#script-used)
    - [Test description](#test-description)
  - [Test case #2 - FDB](#test-case-2---fdb)
    - [Test objective](#test-objective-1)
    - [Script used](#script-used-1)
  - [Test case #3 - VLAN](#test-case-3---vlan)
    - [Test objective](#test-objective-2)
    - [Test configurations](#test-configurations)
    - [Script used](#script-used-2)
  - [Test case #4 - SNMP](#test-case-4---snmp)
    - [Test objective](#test-objective-3)
    - [Test configurations](#test-configurations-1)
    - [Script used](#script-used-3)
- [Open Questions](#open-questions)


**Related documents**
[SONiC BSL Test Plan](https://github.com/Azure/SONiC/blob/master/doc/Sonic%20BSL%20Test%20plan.md)

# Overview

This document describes the implementation of BSL test suite.

## Scope

The test is targeting a running SONIC system with L2 configuration on a T0 topology. Minimal functional verification is performed.


# Setup configuration

For this test a switch is deployed with t0 topology, then new config db (generated with `--preset l2` option) is applied.
This generates a `config_db` with port for each entry according to `port_config.ini`. All ports are members of vlan 1000.

We make sure that the config has a mac by providing `-H` option, and correct ports by providing HWSKU to `-k` and `-p` options.
Configuration example:

```
sonic-cfggen -H -p -k $HWSKU --preset l2 > /etc/sonic/config_db.json
```

# Test cases

According to BSL high level design a number of test should be performed:

| **\#** | **Test Description** | **Expected Result** |
|--------|----------------------|---------------------|
| 1.     |  Sanity              |      Pass           |
| 2.     |  FDB                 |      MAC Learn      |
| 3.     |  VLAN                |   PING,ARP succeeds |
| 4.     |  SNMP                |    CPU, PSU, interface table, MAC table test pass    |


## Pytest infrastructure changes

The BSL test consists of several other tests, so the best approach is to introduce a pytest mark.

The tests will be marked with `@pytest.mark.bsl`.
This way the whole suite can be executed by 

`pytest -m bsl --inventory=inventory --testbed_file=testbed.csv --testbed=${SWITCH}-${TOPO} --host-pattern=${SWITCH}-${TOPO} -vvv`

## Setup of fanout switch

Same as regular t0 testbed.

## Setup of VMs

vEOS VMs will be setup during testbed setup as for regular t0 testbed. The VS are actually not used for the test, optionally they can be disconnected after deploy with `testbed-cli.sh disconnect-vms <topo-name> <vault-password-file>`.

# Test cases

A lot of existing tests, including pytest tests are relying on minigraph facts to get things like 
 - minigraph_ports
 - minigraph_interfaces
 - minigraph_vlans
 - minigraph_portchannels
 - etc.

In our case the configuration is not generated from the minigraph, so there will be a mismatch.
To overcome the issue a new facts, `config_facts` can be introduced.
`config_facts` will return facts in format compatible with minigraph_facts.

A parameter `source` can be passed to the module to specify the source of facts data:
 - `running` (read from redis)
 - `persistent` (read from /etc/sonic/config_db.json)

For redis data the facts module can take output of `sonic-cfggen -d --print-data` as input. (Contents of redis formated as json)

It will return the same facts structure, but based on the contents of redis.
Minigraph was declared deprecated long time ago, and binding our new pytest infrastructure to it is not the way to go.


## Test case #1 - basic sanity
### Test objective

Verify links are up.

### Script used

tests/test_interfaces.py

### Test description

Compare the interfaces between observed states and target state.

## Test case #2 - FDB

### Test objective

Verify FDB forwarding and MACs are learned.

### Script used

tests/fdb/test_fdb.py

## Test case #3 - VLAN
### Test objective

Verify Vlan configurations, ARP and PING. 

### Test configurations

### Script used

VLAN test should be converted to pytest, and adapted to be able to run on l2 configuration. Only l2 traffic tests need to be run(test cases 1-3). 

## Test case #4 - SNMP
### Test objective

Verify MAC, Interface, CPU, PSU tables are available over SNMP.

### Test configurations

Public community string may need to be configured. 

### Script used

tests/snmp/test_snmp_cpu.py
tests/snmp/test_snmp_interfaces.py

Need to imlement tests for MAC and PSU tables.
Need to extend `snmp_facts` to add PSU MIB and Extended Bridge MIB.

# Open Questions

 - 