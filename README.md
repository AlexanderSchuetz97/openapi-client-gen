# openapi-client-gen
Generate a single rust source file with a http-client based on an OpenApi 3.0 or 3.1 schema.
This Project is still experimental and NOT anywhere near ready for any serious use.

The generated code will use the "reqwest" library to make http requests.

# Focus
The focus of this project is providing a single .rs source file which
you can copy (or generate) into your project. This .rs source file should work in a diverse set
of development circumstances:
- Wasm in Browser
- Static/Semi-Static Library that should be used from C/C++ (while libcurl is great, it's still a pain to set up and there appears to be no working generator for these languages)
  - Semi-Static: `.dll`/`.so` with only dependencies to the C-Runtime of the system and NOTHING else.
  - Static: a `.a` archive containing object files
  - headers should be able to be generated using cbindgen
- Rust applications that are
  - Statically linked to libc-musl
  - Dynamically linked to the C-Runtime (GNU, Windows or Other proper OS targets)
- Rust libraries

The "feel" of using the api should be similar as possible for all of these.
In addition to that the .rs file should not conflict with multiple other generated clients should an application
need more than a single OpenApi api to do its job.


# usage
Generate api.rs
```bash
openapi-client-gen --help
openapi-client-gen --out src/api.rs schema.json
```

Toml of your project should contain at these dependencies to compile `api.rs`.
I recommend using the latest version of the respective crates.
These are just the versions used at the time of writing this.
Depending on your target you will have to adjust this.
```toml
[dependencies]
json = "0.12.4"
either = "1.13.0"
http = "1.1.0"
urlencoding = "2.1"
linked-hash-map = "0.5.6"
#If you do not want to use the blocking feature then reqwest doesn't need to have the blocking feature either!
#Using rustls-tls is optional, I use it because you can more easily link with libc-musl.
#The stream feature is required.
reqwest = { version = "0.12.5", features = ["blocking", "rustls-tls", "stream"], default-features = false}
#Not needed if you don't want async feature or target wasm
#Feel free to reduce the amount of required features. To compile only 'codec' is needed.
tokio-util = {version = "0.7.11", features = ["full"]}
#Not needed if you don't want async feature or target wasm
#Feel free to reduce the amount of required features. To compile no particular features are needed.
tokio = { version = "1.39.2" , features = ["full"]}

[features]
#adjust for your target!
#ffi always requires blocking, or it won't compile. It doesn't care if "async" is present or not.
#wasm targets must use only "async"
#normal rust targets require at least either "async" or "blocking" but can have both.
default = ["async", "blocking", "ffi"]
ffi = []
blocking = []
async = []
```

Using cbindgen:
Assuming your library crate is called "turbo_api" you will have to use cbindgen like this (and you will have to enable the ffi feature.)
```bash
cbindgen --config cbindgen.toml --lang c --output turbo_api.h
```
cbindgen.toml:
```toml
[parse.expand]
crates = ["turbo_api"]
```
As of writing this cbindgen requires using the nightly toolchain for header generation.
You can compile the library shared object itself just fine with the stable toolchain.


# Not implemented yet
- Multiple OpenApi source files
- Polymorphism
- Some rarely used "re-usable" objects in OpenApi (if you generate the schema from server-code it will NEVER have this)
- XML (any xml endpoint will just yield a Stream/Vec<u8> which you can manually process)
- YAML OpenApi specs (Only JSON is implemented, which should be good enough since tools exist to convert YAML to JSON)
- Cookies
- Complex Path parameters (Matrix/Objects/Lists etc...)
- Complex Query parameters (Objects/Lists/Multiple etc...)
- Wasm Streams
  - So far in wasm all response bodies of type application/octet-streams are fully streamed into a Vec<u8> before an operation returns.
    As should be obvious, this is not ideal and should be changed.
  - Streams as request bodies is not yet supported by reqwest so there is not much I can do.

# Won't be implemented
- Authentication
  - The api has a way for the programmer (you) to set an interceptor, 
    that will be called before every request where you can attach your "Authorization" header or do whatever you need to do
- Sanitizing of Schemas (To prevent code injection)
  - Only use schemas that you know and trust, or review the schemas/generated code
- Any type of custom styling/templating of the generated code.
  - If you have a valid openapi schema and the generated code for it does not work then I will be happy to adjust this program.
  - If you need to do cosmetic customizations then feel free to suggest them, if you want larger non-functional breaking 
    changes then fork this program.
- Handling of non UTF-8 input files.
- FFI constructor.
  - You will notice that the only FFI function not exported by default is the constructor for the API object.
    This is intentional as I have no idea how your application will come by the base url and what kind of interceptors it will
    want to perhaps add authentication headers to every request. Please implement it yourself in for example lib.rs and export it.