
Same with：
https://github.com/johnzheng1975/proxy-wasm-go-sdk/edit/main/examples/http_headers/test/README.md


Difference is:
1. cd "tmp/proxy-wasm-go-sdk/examples/vm_plugin_configuration"  
2. k create -f .\envoyfilter-wasm.yaml #It has configure value, diff with the previous one.
3. Need restart pods after create envoyfilter



## Can wasm read local file?
Current spike answer, is "NO"
https://discuss.istio.io/t/how-to-go-about-accessing-a-file-from-a-wasm-filter/9936

But can read env var. https://istio.io/latest/docs/reference/config/proxy_extensions/wasm-plugin/#VmConfig

