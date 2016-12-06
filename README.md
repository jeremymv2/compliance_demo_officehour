# Chef Compliance demo & office hour
Demo some latest and greatest Chef Compliance things!

# Audit Cookbook updates

Initially the audit cookbook supported reporting 3 ways:
  * Directly to Compliance
  * To Compliance via Chef Server + Compliance integration
  * Directly to Visibility

Some recent reporting additions:
  * json file (NEW!)
  * Multiple reporting endpoints (NEW!)  
  * report to Visibility via Chef Server proxy (NEW!)

# Store profiles in Automate
There is now an option to store Inspec profiles via a new asset store in Automate!
Version Requirements:
  * chef-server 12.11.0
  * automate 0.6.6

# Using the Automate asset store

## inspec cli
```
$ inspec compliance login_automate https://automate-server.test --insecure true --user admin --dctoken 93a49a4f2482c64126f7b6015e6b0f30284287ee4054ff8807fb63d9cbd1c506 --ent brewinc
$ inspec compliance upload /Users/jmiller/Devel/compliance-profiles/ssh.tar.gz
```

Full example:
https://gist.github.com/jeremymv2/cb34e6dfcad040b1cad50636d256b44e

## Automate API
```
$ curl -X POST "https://automate-server.test/compliance/profiles/jmiller" \
-H "chef-delivery-enterprise: brewinc" -H "chef-delivery-user: jmiller" \
-H "chef-delivery-token: tzwlbWMtgBC0lo6sxkAYKSShxSJEohnU7IAE4NCUGCg=" \
--form "file=@/Users/jmiller/Devel/compliance-profiles/ssh.tar.gz" -k -D -

HTTP/1.1 100 Continue

HTTP/1.1 200 OK
Server: openresty
Date: Mon, 05 Dec 2016 17:52:30 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 0
Connection: keep-alive
Access-Control-Allow-Credentials: true
Access-Control-Allow-Headers: Content-Type, Content-Length, Accept-Encoding, X-CSRF-Token, Authorization, X-Requested-With
Access-Control-Allow-Methods: POST, GET, OPTIONS, PUT, DELETE, PATCH
Access-Control-Allow-Origin:
```

Full example:
https://gist.github.com/jeremymv2/a1cf41bbd2d3a5250e3cf97131efa1a5

# Thoughts on Compliance Server
Question: "With the advent of Automate Compliance profile storage, why even bother with Chef Compliance server?"

Answer: "At the moment, the Compliance server is the only option with a UI for managing profiles and running remote ssh scans for non-chef clients.  However, the Automate Compliance Profile UI will be added soon as well as other functionality, so Chef Compliance would be a better choice only for non-Chef ecosystems."

# Testing out the NEW
Let's examine what it takes to run inspec scan of a node migrating
From this:
  * Audit Cookbook 2.x
  * Profiles Stored on Compliance Server
  * Reports sent directly to Visibility (requires client.rb settings for `data_collector`)

To this:
  * Audit Cookbook 2.x
  * Profiles Stored in Automate (Requires >= 0.6.6)
  * Inspec Scan Reports + Converge data sent to Visibility with _NO_ client.rb `data_collector` settings required (requires Chef Client >= 12.16.42 and Chef Server >= 12.11.0)

**Assumption:** 'linux' and 'ssh' profiles have been uploaded to Automate under user 'jmiller' per example above.

We will define our Audit profile settings via a cookbook wrapper of the `audit` cookbook, called `audit_wrapper`

DEPRECATION WARNING:
If you're still using version Audit Cookbook 1.x
you may have something like:
```
# wrapper cookbook
default['audit']['collector'] = 'chef-visibility'
default['audit']['server'] = 'https://compliance-server.test/api'
default['audit']['refresh_token'] = '2/-YL_ht4owKI1WzczoDDXNhluoZl9Va8nEHpQyBF0w7OFCIa__RZ5vYrcfe5TB_ypycUeFN7BNVhs_4A5HgSvAw=='
default['audit']['profiles'] =
  {
    'base/linux' => true
  }
```

## Reporting with data_collector defined in client.rb
The `fetcher` attribute was introduced in Audit 2.0.0
Also, in 2.x `profiles` are now an array of hashes

The attributes below Fetch Profiles via Compliance Server, Post Reports to Visibility and requires Chef Server Integration with Compliance Server
```
# wrapper cookbook
default['audit']['collector'] = 'chef-visibility'
default['audit']['fetcher'] = 'chef-server'
default['audit']['profiles'] = [
  {
    'name' => 'linux',
    'compliance' => 'base/linux'
  }
]
```

In the scenario above, the client node MUST be configured
with the data_collector for ingesting data into Automate
```
# client.rb
data_collector['server_url'] = 'https://automate-server.test/data-collector/v0/'
data_collector['token'] = '93a49a4f2482c64126f7b6015e6b0f30284287ee4054ff8807fb63d9cbd1c506'
...
```

## No more data_collector in client.rb!

In this scenario, the audit cookbook now requires minimum attribute settings. The configuration below will fetch profiles from Automate and report through the Chef Server, without requiring any `data_collector` settings in the client.rb

```
# wrapper cookbook
default['audit']['collector'] = 'chef-server-visibility'
default['audit']['profiles'] = [
  {
    'name' => 'linux',
    'compliance' => 'jmiller/linux'
  }
]
```

No more client side `data_collector` settings to manage!

```
# client.rb
# data_collector['server_url'] = 'https://automate-server.test/data-collector/v0/'
# data_collector['token'] = '93a49a4f2482c64126f7b6015e6b0f30284287ee4054ff8807fb63d9cbd1c506'
```

However, you must configure the Chef Server.  **Note:** `root_url` is used on Chef Server, not `server_url` (which is used on client side in client.rb)

```
# chef-server.rb
data_collector['root_url'] = 'https://automate-server.test/data-collector/v0/'
data_collector['token'] = '93a49a4f2482c64126f7b6015e6b0f30284287ee4054ff8807fb63d9cbd1c506'
profiles['root_url'] = 'https://automate-server.test'
```

## Additional Resources
  * managing Chef Client client.rb settings: https://github.com/chef-cookbooks/chef-client
  * upgrading Chef Client: https://github.com/chef-cookbooks/omnibus_updater
