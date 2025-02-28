# SoT Aggregation Overview 

The Source of Truth Aggregation Overview is driven by a few key components.

* The ability to have a single GraphQL query to aggregate information.
* The ability to modify data with a "transposer" function.
* The usage of config contexts and the Nautobot's native git platform.

## GraphQL

There is currently support to make an arbitrary GraphQL query that has "device_id" as a variable. It is likely best to use the GraphiQL interface to model
your data, and then save that query to the configuration. The application configuration ensures the following two components.

* The query is a valid GraphQL query.
* The query starts with exactly "query ($device_id: ID!)"". This is to help fail fast and help with overall user experience of clear expectations.

It is worth noting that the graphQL query returned is modified to remove the root key of `device`, so instead of all data being within device, such as
`{"device": {"site": {"slug": "jcy"}}}`, it is simply `{"site": {"slug": "jcy"}}` as an example.

It is helpful to make adjustments to the query, and then view the data from the Plugin's home page and clicking on a given device's `code-json` icon.

## Transposer Function

The transposer function is an optional function to make arbitrary changes to the data after the fact. There is a Plugin configuration that allows the
operator to point to a function within the python path by a string. The function will receive a single variable, that by convention should be called
`data`. The function should return a valid Python json serializable data structure.

```python
def transposer(data):
    """Some."""
    if data["platform"]["slug"] == "cisco_ios":
        data["platform"].update({"support-number": "1-800-ciscohelp"})
    if data["platform"]["slug"] == "arista_eos":
        data["platform"].update({"support-number": "1-800-aristahelp"})
    return data
```

While the example transposer is silly and untested, it provides the structure for which a transposer can be use. The possibilities are obviously endless,
such as reaching out to an external system but operators should use caution not to overload complexity into the transposer. 

The configuration required in the Plugin configuration is as described below.

```python
PLUGINS_CONFIG["nautobot_golden_config"]["sot_agg_transposer"] = "nautobot_golden_config.transposer.transposer"
```
The path described must be within the Python path of your worker. It is up to the operator to ensure that happens.

## Config Contexts

Outside of the scope of this document, but it is worth mentioning the power that configuration context's with integration to Git can provide in this
solution. This is important since config contexts can be used for arbitrary JSON serializable data structures. That is helpful to model configuration
that is not within Nautobot Core or within a Nautobot Plugin. A common use case is to model "global configuration" like data, such as NTP, DNS, SNMP, etc.
For more information, please refer to the Nautobot Core documentation on
[Config Contexts](https://nautobot.readthedocs.io/en/latest/additional-features/config-contexts/#configuration-contexts) and leveraging
[Git Data Sources](https://nautobot.readthedocs.io/en/stable/user-guides/git-data-source/#using-git-data-sources).

## Performance

The GraphQL and transposer functionality could seriously impact the performance of the server. There are no restrictions imposed as it is up to the
operator to weigh the pros and cons of the solution.

## Sample Query

To test your query in the GraphiQL UI, obtain a device's uuid, which can be seen in the url of the detailed device view. Once you have a valid device uuid, you can use the "Query Variables" portion of the UI, which is on the bottom left-hand side of the screen.

Example: Query Variables
```
{
  "device_id": "c2dfa612-3c6b-4a67-8492-a7ca346641f9"
}
```

GraphQL may be new to many users, and while the GraphiQL interface is great way to get started, the following query is for reference. It is
highly recommended to alias name (as in `hostname: name` shown below), as there will be a namespace issue with nornir tasks, which often
take in name as a parameter. 

```
query ($device_id: ID!) {
  device(id: $device_id) {
    config_context
    hostname: name
    position
    serial
    primary_ip4 {
      id
      primary_ip4_for {
        id
        name
      }
    }
    tenant {
      name
    }
    tags {
      name
      slug
    }
    device_role {
      name
    }
    platform {
      name
      slug
      manufacturer {
        name
      }
      napalm_driver
    }
    site {
      name
      slug
      vlans {
        id
        name
        vid
      }
      vlan_groups {
        id
      }
    }
    interfaces {
      description
      mac_address
      enabled
      name
      ip_addresses {
        address
        tags {
          id
        }
      }
      connected_circuit_termination {
        circuit {
          cid
          commit_rate
          provider {
            name
          }
        }
      }
      tagged_vlans {
        id
      }
      untagged_vlan {
        id
      }
      cable {
        termination_a_type
        status {
          name
        }
        color
      }
      tagged_vlans {
        site {
          name
        }
        id
      }
      tags {
        id
      }
    }
  }
}
```