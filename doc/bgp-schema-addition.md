# SONiC BGP POLICIES tables schema proposal #

## Scope of the change ##

Currently BGP policy configuration is pretty much hardcoded into the bgp.conf.j2 template and tailored for one contributor's specific use case.  
Over the last four years we have developed our vendor agnostic configuration management system and have created a set of Jinja templates and data-models to implement routing policy configurations.  
We propose to contribute that work as the basis for configuring routing policies in the config DB and generating Quagga/FRR configurations from it.  
The following document proposes the database Schema in Json format and the list of required changes to the code.

## Current configurations for BGP ##
*sonic-buildimage* depends on the following files to generate a BGP configuration file:
1. */usr/share/sonic/templates/bgpd.conf.json*
   this file in the *bgp* container consumes the configuration database table *BGP_NEIGHBOR* to get a list of BGP neighbors and their limited number of attributes.  
2. */usr/bin/bgpcfgd*
   this file in the *bgp* container consumes the configuration database table *BGP_NEIGHBOR* to dynamically create new BGP neighborships.

In the end this is used to produce */etc/quagga/bgpd.conf* or */etc/frr/bgpd.conf*.

## Limits and incentives to change ##
- BGP policy is - out of the box - limited to the needs of the original contributor.
- There is no flexibility in changing routing policy as the configuration is hard coded.
- Currently there is no provision for human utilities like peer-groups.
- Custom policy requires the upload of a specific template file at router turn-up or application specific sonic builds.
- Barrier to adoption.

## Proposal: integrate the BGP policy configuration into config DB ##
### Proposed Schema ###

#### Additions to the schema ####
##### 1. Prefix-lists
```
"BGP_PREFIX_SET": {
    NAME: {
        "description": DESCRIPTION,
        CIDR : {
            "compare": "eq"|"ge"|"le",
            "length": INTEGER
        },
    },
}
```

We define the table *BGP_PREFIX_SET* to store a set of prefix-lists. A prefix-list has a:
- **NAME**; String, a user defined *NAME* eg.: *"pf_default"*
- **_"description"_**: String, usually a comment describing what the prefix-list is used for
The prefix list contains a set of:
- **CIDR**: String, IPv4 or IPv6 prefix in CIDR notation.
Each prefix has 2 keys:
- **_"compare"_**: Optional, the prefix's variable length comparator with 3 possible values
  - **_"eq"_**: equal
  - **_"ge"_**: greater or equal
  - **_"le"_**: less or equal
- **_"length"_**: required if **_"compare"_** is defined, length of the variable prefix

NOTE: we do not define a *"permit"* or *"deny"* action, we approach the prefix-list as in JunOS/IOSXR, in [IE]OS notation-land we consider everything to be "permit" 

Example:
```
"BGP_PREFIX_SET": {
    "pf_default": {
        "description": "matches the default route",
        "0.0.0.0/0" : {},
    },
    "pf_bogons4": {
        "description": "IPv4 Bogon list",
        "0.0.0.0/8" : {
            "compare": "le",
            "length": "32"
        },
        "10.0.0.0/8" : {
            "compare": "le",
            "length": "32"
        },
...
        "240.0.0.0/4": {
            "compare": "le",
            "length": "32"
        }
    }
}

```
##### 2. Named communities
```
"BGP_COMMUNITY": {
    NAME: COMMUNITY,
}
```

We define the table *BGP_COMMUNITY* to contain named communities. This table contains simple tuples. it is mainly a utility table; it can be argued that it could be removed.
- **NAME**: String, name of the community
- **COMMUNITY**: String, community in decimal notation, can be a regexp.

Example:
```
"BGP_COMMUNITY": {
    "co_default": "65000:60000",
    "co_l3_cust": "3356:123",
    "co_service": "649..:5...."
}
```

##### 3. Community-lists
```
"BGP_COMMUNITY_SET": {
    NAME : {
        "description": DESCRIPTION,
        "list": {
            BGP_COMMUNITY_NAME: {},
        }
    },
}
```

We define the table *BGP_COMMUNITY_SET* to contain a set of community-lists.  
A community-list is defined by a:
- **NAME**: String, name of the community-list
- **_"description"_**: String, usually explains what the list is for
- **_"list"_**: this key contains a set of named communities from the *BGP_COMMUNITY* table

Example:
```
"BGP_COMMUNITY_SET": {
    "cl_location" : {
        "description": "mark the route with the location it originates from",
        "list": {
            "co_sv8": {}
        }
    },
    "cl_clos": {
        "description": "routes that are part of the dc fabric",
        "list": {
            "co_pod": {},
            "co_spines": {}
        }
    }
}
```

##### 4. As-path lists
```
"BGP_AS_SET": {
    NAME: {
        "description": DESCRIPTION,
        "re": {
            REGEXP: {},
    },
},
```

The table *BGP_AS_SET* stores a set of as-path lists named:
- **NAME** : String, name of the as-path list
each as-path list is defined by a:
- **_"description"_**: String, usually explains what we are trying to filter
- **_"re"_**: key to a set of
- **REGEXP**: String, represents an aspath regexp.

Example:
```
"BGP_AS_SET": {
    "ap_l3": {
        "description": "level3 only routes,
        "re": {
            "^3356$": {}
        }
    },
    "ap_rack_101_312": {
        "description": "routes originating from racks 101 and 312",
        "re": {
            "_65101$": {},
            "_65312$": {}
        }
    }
}
```

##### 5. Route-maps or policy-statements
```
"BGP_POLICY": {
    NAME: {
        "description": DESCRIPTION,
        TERM: {
            "description": DESCRIPTION,
            "conditions": {
                "prefix_set": BGP_PREFIX_SET_NAME,
                "community": BGP_COMMUNITY_NAME
                "communities": BGP_COMMUNITY_SET_NAME,
                "as_path": BGP_AS_SET_NAME,
                "protocol": "connected"|"static"|"isis"|"bgp",
            },
            "actions": {
                "next_hop": "self"|IPADDR,
                "permit": "true"|"false",
                "local_preference": INTEGER,
                "communities": {
                    "additive": "true"|"false",
                    BGP_COMMUNITY_NAME: {},
                }
            }
        },
    },
}
```

The *BGP_POLICY* table contains all route-maps that reference the contents of the previously defined tables.
A route-map has a:
- **NAME**: String, name of the route-map
- **_"description"_**: String, describes to humans what the route-map does
- **TERM**: String, contains an integer defining the sequence of the term
- **_"conditions"_**: Key having one or multiple values, defined the *match* conditions of the route-map term. these can be:
  - **_"prefix_set"_**: references by *NAME* of a prefix-list from the table *BGP_PREFIX_SET*
  - **_"community"_**: references by *NAME* of a community from the table *BGP_COMMUNITY*
  - **_"communities"_**: references by *NAME* of a community-list from the table *BGP_COMMUNITY_SET*
  - **_"as_path"_**: references by *NAME* of a community-list from the table *BGP_AS_SET*
  - **_"protocol"_**: can be one of 4 values that match the source protocol of the route: *"connected"*,*"static"*,*"isis"*,*"bgp"*
- **_"actions"_**: Key having one or multiple values, defined the *set* actions of the route-map term. these can be:
  - **_"next_hop"_**: can one of 2 values to set the next hop of the route: "self" or a String representing an IP address
  - **_"permit"_**: can be one of 2 values, *"true"* or *"false"*, basically the behavior of the term: it either permits or denies the route that was matched
  - **_"local_preference"_**: String, represents the value of the local-pref to set to the route
  - **_"communities"_**: set of references by *NAME* of a community from the table *BGP_COMMUNITY* to add to the route
  - **_"additive"_**: can have 2 values *"true"* or *"false"* this defines if the communities are added (*"true"*) or replaced (*"false"*), default is *"false"*

Example:
```
"BGP_POLICY": {
    "rm_clos_in": {
        "description": "routes accepted into the clos matrix router,
        "10": {
            "description": "accept routes from servers"
            "conditions": {
                "community": "co_server"
            },
            "actions": {
                "permit": "true",
                "local_preference": "100"
            }
        },
        "20": {
            "description": "accept default route from edge pod"
            "conditions": {
                "prefix_set": "pf_default"
            },
            "actions": {
                "permit": "true"
            }
        }
    },
    "rm_clos_out": {
        "description": "routes announced by the router to the rest of the matrix,
        "10": {
            "description": "server routes"
            "conditions": {
                "protocol": "connected",
                "prefix_set": "pf_dc01_servers"
            },
            "actions": {
                "permit": "true",
                "communities": {
                    "co_location": {},
                    "co_server": {}
                }
            }
        },
        "20": {
            "description": "allow loopbacks"
            "conditions": {
                "prefix_set": "pf_loopbacks"
            },
            "actions": {
                "permit": "true"
            }
        }
    }
}
```

#### Tying it all together: Modifications to the *BGP_NEIGHBOR* table: ####
```
"BGP_NEIGHBOR": {
    NEIGHBOR_IP: {
        "group": GROUP_NAME,
        "policy_in": BGP_POLICY_NAME,
        "policy_out": BGP_POLICY_NAME,
        "password": PASSWD,
    },
    GROUP_NAME: {
    }
}
```

Where:
- **NEIGHBOR_IP**: String, represents an IPv4 or IPv6 address, use *ipaddr* jinja module to validate and differentiate with *GROUP_NAME*.
- **GROUP_NAME**: String, represents the name of a peer-group
- **BGP_POLICY_NAME**: String, refers to the *NAME* key in the *BGP_POLICY* table
- **PASSWD**: String, the password of the bgp session

By detecting if the bgp neighbor is a name or an IP in the jinja template we can differentiate between neighbors and peer-groups in the configuration generator. The *GROUP_NAME* key has the same schema as the *NEIGHBOR_IP* key.  
We introduce the following keys to *NEIGHBOR_IP*:
- **_"group"_**: String, defines the name of the peer-group this neighbor belongs to, by default this key does not exist and we detect that in the jinja template to maintain backwards compatibility
- **_"policy_in"_**: String, defines the name of the policy ( route-map ) applied to learned routes
- **_"policy_out"_**: String, defines the name of the policy ( route-map ) applied to advertised routes
- **_"password"_**: String, the password used for authentication with the neighbor


### Files needing modification for implementation ###

The changes we propose are only additive to remain compatible with the current install base and the current way of doing things.

In repo *sonic-buidlimage*:

*dockers/docker-fpm-quagga/bgpd.conf.j2*:  
*dockers/docker-fpm-frr/bgpd.conf.j2*:  

*dockers/docker-snmp-quagga/bgpcfgd*:  
*dockers/docker-snmp-frr/bgpcfgd*:  


in repo *sonic-swss-common*: 

*common/schema.h*:  
```
#define CFG_BGP_PREFIX_SET_TABLE_NAME           "BGP_PREFIX_SET"
#define CFG_BGP_AS_SET_TABLE_NAME               "BGP_AS_SET"
#define CFG_BGP_COMMUNITY_TABLE_NAME            "BGP_COMMUNITY"
#define CFG_BGP_COMMUNITY_SET_TABLE_NAME        "BGP_COMMUNITY_SET"
#define CFG_BGP_POLICY_TABLE_NAME               "BGP_POLICY"
```
## Unsolved Issues ##
- This is small subset of features in the myriad that BGP proposes, we believe that this model can be extended to provide support for future developments
