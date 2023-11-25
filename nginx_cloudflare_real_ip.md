p: cloudflares IPs might change invalidating the `set_real_ip_from` stuff in nginx configs<br>
s: auto-update `set_real_ip_from` config to match cloudflares current IP range

to do that, one must generate the `set_real_ip_from` directives for all the IPs currently used by CloudFlare
> https://www.cloudflare.com/ips-v4/
> https://www.cloudflare.com/ips-v6/

to this less annoying the `set_real_ip_from` directives can be outsourced to a seperate file that just gets regenerated periodicly.<br>
this file can then be referenced with `include cf_restore_real_ip.conf;` in the websites nginx config .

`generate_cf_ip_restore_conf.py`:
``` python
import datetime
import ipaddress
import requests


cf_ip4_list_url = "https://www.cloudflare.com/ips-v4/"
cf_ip6_list_url = "https://www.cloudflare.com/ips-v6/"

nginx_real_ip_header_param = "real_ip_header"
nginx_real_ip_from_param = "set_real_ip_from"
cf_real_ip_header = "CF-Connecting-IP"

out = ""
out += f"# generated {datetime.datetime.now()}\n\n"
out += f"{nginx_real_ip_header_param} {cf_real_ip_header};\n"
out += "real_ip_recursive on;\n"

# one should probably sanitize the responses better just in case cloudflare turns/is evil.
# but thats no fun
ip_list = requests.get(cf_ip4_list_url).text.split("\n") + requests.get(cf_ip6_list_url).text.split("\n")

for ip in ip_list:
    # print(ip)
    try:
        ipaddress.ip_network(ip)
        out += f"{nginx_real_ip_from_param} {ip};\n"
    except ValueError:
        out += f"# INVALID IP RANGE {ip}"

print(out)
```

after generating the new `set_real_ip_from` directives we compare them to the old ones to check if they have changed.<br>
if they changed we replace the old file with the new one and restart nginx to apply the changes

comparing could be done on the raw ip list from `https://www.cloudflare.com/ips` to save minimal overhead of generating the whole file first but whatever 

`update_cf_ips.sh`
```
python3 ./generate_cf_ip_restore_conf.py > cf_ips.tmp
has_changed="$(cmp -i 38 -s cf_ips.tmp /etc/nginx/cf_restore_real_ip.conf)"
if [[ $? != 0 ]]; then
        mv cf_ips.tmp /etc/nginx/cf_restore_real_ip.conf
	service nginx restart
	echo "cloudflare ip range changed, restarting nginx"
fi
```

and finally, to the nginx config responsible for the traffic routed through CF add: `include cf_restore_real_ip.conf;`
