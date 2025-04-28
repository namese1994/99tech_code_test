Provide your CLI command here:
```
jq -r 'select(.symbol=="TSLA" and .side=="sell") | .order_id' ./transaction-log.txt | xargs -I{} curl -s "https://example.com/api/{}" > ./output.txt
```

## Notes:
This command does three things in sequence:

1. Uses jq to read each JSON line in the file and extract order_ids where symbol is TSLA and side is sell.

2. Pipes those IDs into xargs, which runs curl -s https://example.com/api/<order_id> for each one.

3. Redirects all of the HTTP responses into output.txt.