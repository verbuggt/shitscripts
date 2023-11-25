# DO NOT USE (SHIT)
### migraine warning

p: automaticly updating DNS entries on CloudFlare to dynamic IP address
s: just do it

### noise

this specificly extracts the `temporary` IPv6 from `ip addr show` in a horrible way.<br>
horrible because if there are multiple temporary ips assigned (possibly on different interfaces) things get messy.<br>
also depending on configuration the public IPv6 address might not be `temporary` rendering the regular expression used below useless.

one could use an api like `https://ip6only.me/api/`<br>
or do proper sanitaion of the output of `ip addr show`<br>
but why bother, just fix it when it becomes a problem on friday at 19:00 

### files and code

`check_for_new_ip.sh`:
``` bash
DYN=$(ip addr show | sed -n "s/^.*inet6\s*\(\S*\)\/.*temporary.*$/\1/p" | xargs)
OLD=$(cat ip6.old | xargs)

if [ "$DYN" != "$OLD" ]; then
    echo "[IP CHANGE] OLD $OLD"
    echo "[IP CHANGE] NEW $DYN"
    ./update_cloudflare_dns_entry.sh "$DYN"
    echo "$DYN" > ip6.old
else
    exit 0
fi
```

`update_cloudflare_dns_entry.sh`:
```
echo "$1"
curl --request PUT \
  --url https://api.cloudflare.com/client/v4/zones/ZONE_URL \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer SUPER_SECRET_AUTH_TOKEN_FROM_CLOUDFLARE' \
  --data '{
  "content": "'"$1"'",
  "name": "subdomain.example.com",
  "proxied": false,
  "type": "AAAA",
  "comment": "dynamic ip6",
  "tags": [],
  "ttl": 60
}'
```

### persist shit

to be effective this shit needs to be run all the time to detect ip changes and update dns accordingly.

`crontab -e`

`  * *  *   *   *     /path/to/destruction/check_for_new_ip.sh`
