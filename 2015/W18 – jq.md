# Getting along with JSON using `jq`

Last week I stumbled upon `jq`. It's a shell command that let's you search, filter and parse JSON. It's like `grep` or `sed` for JSON.


`jq` needs some time to get used to, but its [online playground](https://jqplay.org) helped me a lot.

I use it to work with a JSON “database” that contains our servers. The JSON looks like that:

```json
[
  {
    "name": "Webserver01",
    "domain": "web01.example.com",
    "tags": ["ubuntuutopic", "vm", "web", "dc01"]
  },
  {
    "name": "Webserver02",
    "domain": "web02.example.com",
    "tags": ["ubuntuutopic", "vm", "web", "dc02"]
  },
  {
    "name": "Storage01",
    "domain": "storage01.example.com",
    "tags": ["debianwheezy", "host", "dc01"]
  },
  {
    "name": "VMHost01",
    "domain": "vmhost01.example.com",
    "tags": ["ubuntuvivit", "host", "dc01"]
  },
  {
    "name": "VMHost02",
    "domain": "vmhost02.example.com",
    "tags": ["ubuntuvivit", "host", "dc02"]
  }
]
```

With the help of `jq` I can quickly filter for those servers I'm looking for.

This returns all machines running as webservers:

    $ cat hosts.json | jq 'map( select(.tags | contains(["web"]) ) )'
    [
      {
        "name": "Webserver01",
        "domain": "web01.example.com",
        "tags": [
          "ubuntuutopic",
          "vm",
          "rz01"
        ]
      },
      {
        "name": "Webserver02",
        "domain": "web02.example.com",
        "tags": [
          "ubuntuutopic",
          "vm",
          "rz02"
        ]
      }
    ]

Or all hosts from datacenter 01

    $ cat hosts.json | jq 'map( select(.tags | contains(["dc01", "host"]) ) )'
    [
      {
        "name": "Storage01",
        "domain": "storage01.example.com",
        "tags": [
          "debianwheezy",
          "host",
          "dc01"
        ]
      },
      {
        "name": "VMHost01",
        "domain": "vmhost01.example.com",
        "tags": [
          "ubuntuvivit",
          "host",
          "dc01"
        ]
      }
    ]

Both commands return an JSON array with the filtered output.

## What can I do with it now?
Most of our servers run on Ubuntu, some of them on Debian, some are virtual machines, others are hosts. We use [Chef](https://downloads.chef.io/chef-server/) to tame those systems and to align configurations and setups, so we don't have to ssh into every single machine all of the time.

But there are cases in which I need to execute an individual command on multiple machines. For example, when I detect an attacker who tries to bruteforce his way in I'd like to block him on all of our servers. I use [pssh](https://code.google.com/p/parallel-ssh/) to execute commands simultanously on multiple servers, but I need to give it a list of those first.

Using `jq` I can quickly create this list and pass it to pssh:

    pssh -i -H "`jq --raw-output 'map( select(.tags | contains(["dc01", "host"]) ) )' hosts.json`" aptitude update

Of course that command is pretty heavy and not easy to remember. I stuffed an enhanced version of it into a shell function in my `.zshrc`:

    function psshjq(){
      pssh -i -H "`jq --arg tags $1 -r -f /path/to/filter.jq /path/to/hosts.json`" $2
    }

I can now quickly use it to quickly send commands to a subset of our server collection:

    $ psshjq "dc01,host" "aptitude update"