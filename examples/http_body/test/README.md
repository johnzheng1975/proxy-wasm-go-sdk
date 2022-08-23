 Same with： https://github.com/johnzheng1975/proxy-wasm-go-sdk/edit/main/examples/http_headers/test/README.md

Difference is:
```
# cd ~/tmp/proxy-wasm-go-sdk/examples/http_body
  ... ...
# Test command
   k exec -ti nginx -- bash
   71  curl   httpbin:8000/post -H "buffer-operation: prepend"  --data '[initial body]'
   [this is prepended body][initial body]
   
   72  curl   httpbin:8000/post -H "buffer-operation: append"  --data '[initial body]'
   [initial body][this is appended body]
   
   73  curl   httpbin:8000/post -H "buffer-operation: replace"  --data '[initial body]'
   "[this is replaced body]"
```
