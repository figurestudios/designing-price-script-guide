# Designing a Provider Price Script

## Introduction & what could be next

This page was created for the 2023 Akash-a-thon to help providers understand how to design their own price script. In the community, there are some strategies implemented by some providers that are not documented anywhere. One of these is not bidding on tiny deployments, due to reasons such as staking rewards being higher, it not being worth a provider's time or lockup, and due to these usually only being short-term leases.

Here, you will be able to find snippets on how to handle resource management, tenant addresses, different values, and more. Different people will believe in different strategies, and this is a collection of how to implement them in your own script. With enough feedback and demand, a web-tool creating a price script with no scripting knowledge can be an expansion for the project.

## Usefulness

Lists in the form of e.g. whitelists/blacklists can help identify certain good or bad actors of the network and being able to use this information in price scripts help save time monitoring and the potential of money loss. If someone were to create a lot of dummy leases to clog up the provider collateral for 5 minutes, it would be ideal to catch this early and not bid on this tenants' leases. This would be as simple as only bidding on leases of tenants of a certain characteristic, or limiting it to a certain number per hour.

Resource management in price scripts can orchestrate renting of a machine to fill it up to 100%, especially if done in an advanced way. This would make a provider more competitive if each machine could be filled with no wasted resources. It also helps against the potential of money loss in the form of bidding on deployments not deemed profitable. If someone is trolling by requesting to bid an absurd number of ports, it could be beneficial to ignore this request or charge a lot for it as not to clog up the public ports for normal use.

## Resource values

These values are applicable for modified versions of the [generic price script](https://github.com/akash-network/helm-charts/blob/main/charts/akash-provider/scripts/price_script_generic.sh). To create a price script from scratch, these values must be read in in the same way that this script retrieves them.

| Value | Description |
| --- | --- |
| `$AKASH_OWNER` | tenant address (akash1...) requesting |
| `cpu_requested` | threads requested |
| `memory_requested` | GB of RAM requested |
| `ephemeral_storage_requested` | GB of normal storage requested |
| `hdd_pers_storage_requested` | GB of beta1 storage requested |
| `ssd_pers_storage_requested` | GB of beta2 storage requested |
| `nvme_pers_storage_requested` | GB of beta3 storage requested |
| `ips_requested` | number of unique IP addresses |
| `endpoints_requested` | number of ports |
| `gpu_units` | number of GPU units |
| `model` | GPU model name |

> Note: if you are simply changing how much you want to earn per resource per month, these values are available in the [generic price script](https://github.com/akash-network/helm-charts/blob/main/charts/akash-provider/scripts/price_script_generic.sh) commented as "USD/resource-month".

With the values above, you can create custom strategies that for example limits your bids only to:
- deployments of certain amount of resources
- deployments with a certain ratio between two resources
- deployments only from tenants who has already has active leases on the network

## Tenant-based strategy examples

Only bid on deployments created by whitelisted addresses based off variable the `$WHITELIST_URL` in [this format](https://gist.github.com/andy108369/1fa6cfa81674bce438a450d6c14395ea):

```sh
if ! [[ -z $WHITELIST_URL ]]; then
  WHITELIST=/tmp/price-script.whitelist
  if ! test $(find $WHITELIST -mmin -10 2>/dev/null); then
    curl -o $WHITELIST -s --connect-timeout 3 --max-time 3 -- $WHITELIST_URL
  fi

  if ! grep -qw "$AKASH_OWNER" $WHITELIST; then
    echo -n "$AKASH_OWNER is not whitelisted" >&2
    exit 1
  fi
fi
```

> taken from [generic price script](https://github.com/akash-network/helm-charts/blob/main/charts/akash-provider/scripts/price_script_generic.sh)

Same as above, but gives a discount to whitelisted addresses. Another script could query the Akash network to create a filter or tenants to give disconted rates to. Last, multiply the multiplier to the `total_cost_uakt` variable.

```sh
if ! [[ -z $WHITELIST_URL ]]; then
  WHITELIST=/tmp/price-script.whitelist
  if ! test $(find $WHITELIST -mmin -10 2>/dev/null); then
    curl -o $WHITELIST -s --connect-timeout 3 --max-time 3 -- $WHITELIST_URL
  fi

  if grep -qw "$AKASH_OWNER" $BLACKLIST; then
    echo -n "$AKASH_OWNER is whitelisted" >&2
    discount_multiplier=0.50
  fi
fi
```

Similar to the ones above, but instead of bidding, you are ignoring bids from specific tenants. These could have been identified by another script that queries the network for spamming leases, or shared by other tenants to have hosted content you do not wish to host.

```sh
if ! [[ -z $BLACKLIST_URL ]]; then
  BLACKLIST=/tmp/price-script.blacklist
  if ! test $(find $BLACKLIST -mmin -10 2>/dev/null); then
    curl -o $BLACKLIST -s --connect-timeout 3 --max-time 3 -- $BLACKLIST_URL
  fi

  if grep -qw "$AKASH_OWNER" $BLACKLIST; then
    echo -n "$AKASH_OWNER is blacklisted" >&2
    exit 1
  fi
fi
```

Whitelists/blacklists/informational lists could be created from all information that can be queried on addresses, plus outside information based off trust in the form of KYC. Some example values that could be used are: total delegated funds, total funds, first transaction date, number/info of leases, and authorized funds.

## Resource-based snippets

If less than 1 (thread) of a resource (`$cpu_requested`) is requested, the script will exit as not to bid. Useful for only bidding on larger deployments.

```sh
if [[ $cpu_requested -lt 1 ]]; then
  exit 1
fi
```

If you want to bid on everything but with a minimum price, you can always simply check if the `$total_cost_usd_target` variable is large enough for your liking. The following example will have a minimum `$total_cost_usd_target` of $5/month:

```sh
if [[ $total_cost_usd_target -lt 5 ]]; then
  total_cost_usd_target=5
fi
```

Alternatively for the above, you could simply not bid on it as not to waste fees when you are not competitive here:

```sh
if [[ $total_cost_usd_target -lt 5 ]]; then
  exit 1
fi
```

If less than a preferred ratio of resources are requested, the script will exit as not to bid. This is useful for managing resources in a way that fills up the entire server of one or more resources, in the case that you want to lease out 100% of e.g. your storage, memory, or threads. 

```sh
# example: server with 128 threads and 128 GB of ephemeral storage where we want the threads to fill up first, or low disk usage on our leases.

minimum_resource_ratio=1 # 128/128 = 1 thread/GB ratio
if [[ ($cpu_requested / $ephemeral_storage_requested) -lt $minimum_resource_ratio ]]; then
  exit 1
fi
```

## Contact

Please tag @figur in the [Akash Discord Server](https://discord.akash.network/) if you have feedback or if you need help creating a price script from these snippets.
