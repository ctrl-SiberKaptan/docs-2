# 2. Hosted Services

In the previous example, we used a local, browser-native service to facilitate the string generation and communication with another browser. The real power of the Fluence solution, however, is that services can be hosted on one or more nodes, easily reused and composed into decentralized applications with Aqua.

:::info
In case you haven't set up your development environment, follow the [setup instructions](../../tutorials/setting-up-your-environment.md) and clone the [examples repo](https://github.com/fluencelabs/examples):

```sh
git clone https://github.com/fluencelabs/examples
```
:::

### Creating A WebAssembly Module

In this section, we develop a simple `HelloWorld` service and host it on a peer-to-peer node of the Fluence testnet. In your IDE or terminal, change to the `2-hosted-services` directory and open the `src/main.rs` file:

```rust
use marine_rs_sdk::marine;
use marine_rs_sdk::module_manifest;

module_manifest!();

pub fn main() {}

#[marine]
pub struct HelloWorld {
    pub msg: String,
    pub reply: String,
}

#[marine]
pub fn hello(from: String) -> HelloWorld {
    HelloWorld {
        msg: format!("Hello from: \n{}", from),
        reply: format!("Hello back to you, \n{}", from),
    }
}

#[cfg(test)]
mod tests {
    use marine_rs_sdk_test::marine_test;

    #[marine_test(config_path = "../configs/Config.toml", modules_dir = "../artifacts")]
    fn non_empty_string(hello_world: marine_test_env::hello_world::ModuleInterface) {
        let actual = hello_world.hello("SuperNode".to_string());
        assert_eq!(actual.msg, "Hello from: \nSuperNode".to_string());
    }

    #[marine_test(config_path = "../configs/Config.toml", modules_dir = "../artifacts")]
    fn empty_string(hello_world: marine_test_env::hello_world::ModuleInterface) {
        let actual = hello_world.hello("".to_string());
        assert_eq!(actual.msg, "Hello from: \n");
    }
}
```

Fluence hosted services are comprised of WebAssembly modules implemented in Rust and compiled to [wasm32-wasi](https://doc.rust-lang.org/stable/nightly-rustc/rustc_target/spec/wasm32_wasi/index.html). Let's have look at our code:

```rust
// quickstart/2-hosted-services/src/main.rs
use marine_rs_sdk::marine;
use marine_rs_sdk::module_manifest;

module_manifest!();

pub fn main() {}

#[marine]
pub struct HelloWorld {
    pub msg: String,
    pub reply: String,
}

#[marine]
pub fn hello(from: String) -> HelloWorld {
    HelloWorld {
        msg: format!("Hello from: \n{}", from),
        reply: format!("Hello back to you, \n{}", from),
    }
}
```

At the core of our implementation is the `hello` function which takes a string parameter and returns the `HelloWorld` struct consisting of the `msg` and `reply` field, respectively. We can use the `build.sh` script in the `scripts` directory, `./scripts/build.sh` , to compile the code to the wasi32-wasm target from the VSCode terminal.

In addition to some housekeeping, the `build.sh` script gives the compile instructions with [marine](https://crates.io/crates/marine), `marine build --release` , and copies the resulting Wasm module, `hello_world.wasm`, to the `artifacts` directory for easy access.

### Testing And Exploring Wasm Code

So far, so good. Of course, we want to test our code and we have a couple of test functions in our `main.rs` file:

```rust
// quickstart/2-hosted-services/src/main.rs
use marine_rs_sdk::marine;
use marine_rs_sdk::module_manifest;

//<snip>

#[cfg(test)]
mod tests {
    use marine_rs_sdk_test::marine_test;

    #[marine_test(config_path = "../configs/Config.toml", modules_dir = "../artifacts")]
    fn non_empty_string(hello_world: marine_test_env::hello_world::ModuleInterface) {
        let actual = hello_world.hello("SuperNode".to_string());
        assert_eq!(actual.msg, "Hello from: \nSuperNode".to_string());
    }

    #[marine_test(config_path = "../configs/Config.toml", modules_dir = "../artifacts")]
    fn empty_string(hello_world: marine_test_env::hello_world::ModuleInterface) {
        let actual = hello_world.hello("".to_string());
        assert_eq!(actual.msg, "Hello from: \n");
    }
}

```

\
To run our tests, we can use the familiar [`cargo test`](https://doc.rust-lang.org/cargo/commands/cargo-test.html) . However, we don't really care all that much about our native Rust functions being tested but want to test our WebAssembly functions. This is where the extra code in the test module comes into play. In short., we are running `cargo test` against the exposed interfaces of the `hello_world.wasm` module and in order to do that, we need the `marine_test` macro and provide it with both the modules directory, i.e., the `artifacts` directory, and the location of the `Config.toml` file. Note that the `Config.toml` file specifies the module metadata and optional module linking data. Moreover, we need to call our Wasm functions from the module namespace, i.e. `hello_world.hello` instead of the standard `hello` -- see lines 13 and 19 above, which we specify as an argument in the test function signature (lines 11 and 17, respectively).

:::info
In order to able able to use the macro, install the [marine-rs-sdk-test](https://crates.io/crates/marine-rs-sdk-test) crate as a dev dependency:

`[dev-dependencies] marine-rs-sdk-test = "<version>"`
:::

From the IDE or terminal, we now run our tests with the`cargo +nightly test --release` command. Please note that if `nightly` is your default, you don't need it in your `cargo test` command.

Well done -- our tests check out. Before we deploy our service to the network, we can interact with it locally using the [Marine REPL](https://crates.io/crates/mrepl). In your VSCode terminal the `2-hosted-services` directory run:

```sh
mrepl configs/Config.toml
```

which puts us in the REPL:

```sh
mrepl configs/Config.toml
Welcome to the Marine REPL (version 0.9.1)
Minimal supported versions
  sdk: 0.6.0
  interface-types: 0.20.0

app service was created with service id = 8a2d946d-b474-468c-8c56-9e970ee64743
elapsed time 53.593404ms

1> i
Loaded modules interface:
data HelloWorld:
  msg: string
  reply: string

hello_world:
  fn hello(from: string) -> HelloWorld

2> call hello_world hello ["Fluence"]
result: Object({"msg": String("Hello from: \nFluence"), "reply": String("Hello back to you, \nFluence")})
 elapsed time: 278.5µs

3>
```

We can explore the available interfaces with the `i` command and see that the interfaces we marked with the `marine` macro in our Rust code above are indeed exposed and available for consumption. Using the `call` command, still in the REPL, we can access any available function in the module namespace, e.g., `call hello_word hello [<string arg>]`. You can exit the REPL with the `ctrl-c` command.

### Exporting WebAssembly Interfaces To Aqua

In anticipation of future needs, note that `marine` allows us to export the Wasm interfaces ready for use in Aqua. In your VSCode terminal, navigate to the `2-hosted-services` directory

```sh
marine aqua artifacts/hello_world.wasm
```

Which gives us the Aqua-ready interfaces:

```aqua
data HelloWorld:
  msg: string
  reply: string

service HelloWorld:
  hello(from: string) -> HelloWorld
```

That can be piped directly into an aqua file , e.g. `marine aqua my_wasm.wasm >> my_aqua.aqua`.

### Deploying A Wasm Module To The Network

Looks like all is in order with our module and we are ready to deploy our `HelloWorld` service to the world by means of the Fluence peer-to-peer network. For this to happen, we need two things: the peer id of our target node(s) and a way to deploy the service. The latter can be accomplished with the `aqua` command line tool and with respect to the former, we can get a peer from one of the Fluence testnets with `aqua` . In your VSCode terminal:

```sh
aqua config default_peers
```

Which gets us a list of network peers:

```
/dns4/kras-00.fluence.dev/tcp/19990/wss/p2p/12D3KooWSD5PToNiLQwKDXsu8JSysCwUt8BVUJEqCHcDe7P5h45e
/dns4/kras-00.fluence.dev/tcp/19001/wss/p2p/12D3KooWR4cv1a8tv7pps4HH6wePNaK6gf1Hww5wcCMzeWxyNw51
/dns4/kras-01.fluence.dev/tcp/19001/wss/p2p/12D3KooWKnEqMfYo9zvfHmqTLpLdiHXPe4SVqUWcWHDJdFGrSmcA
/dns4/kras-02.fluence.dev/tcp/19001/wss/p2p/12D3KooWHLxVhUQyAuZe6AHMB29P7wkvTNMn7eDMcsqimJYLKREf
/dns4/kras-03.fluence.dev/tcp/19001/wss/p2p/12D3KooWJd3HaMJ1rpLY1kQvcjRPEvnDwcXrH8mJvk7ypcZXqXGE
/dns4/kras-04.fluence.dev/tcp/19001/wss/p2p/12D3KooWFEwNWcHqi9rtsmDhsYcDbRUCDXH84RC4FW6UfsFWaoHi
/dns4/kras-05.fluence.dev/tcp/19001/wss/p2p/12D3KooWCMr9mU894i8JXAFqpgoFtx6qnV1LFPSfVc3Y34N4h4LS
/dns4/kras-06.fluence.dev/tcp/19001/wss/p2p/12D3KooWDUszU2NeWyUVjCXhGEt1MoZrhvdmaQQwtZUriuGN1jTr
/dns4/kras-07.fluence.dev/tcp/19001/wss/p2p/12D3KooWEFFCZnar1cUJQ3rMWjvPQg6yMV2aXWs2DkJNSRbduBWn
/dns4/kras-08.fluence.dev/tcp/19001/wss/p2p/12D3KooWFtf3rfCDAfWwt6oLZYZbDfn9Vn7bv7g6QjjQxUUEFVBt
/dns4/kras-09.fluence.dev/tcp/19001/wss/p2p/12D3KooWD7CvsYcpF9HE9CCV9aY3SJ317tkXVykjtZnht2EbzDPm
```

Let's use the peer `12D3KooWFEwNWcHqi9rtsmDhsYcDbRUCDXH84RC4FW6UfsFWaoHi` as our deployment target and deploy our service from the VSCode terminal.

To do so we first need to generate a `secret key`. Run:

```sh
aqua key create
```

Replace `<YOUR_SECRET_KEY>` with the generated secret key and while in the `quickstart/2-hosted-services` directory run:

```sh
aqua remote deploy_service \
    --addr /dns4/kras-04.fluence.dev/tcp/19001/wss/p2p/12D3KooWFEwNWcHqi9rtsmDhsYcDbRUCDXH84RC4FW6UfsFWaoHi \
    --sk <YOUR_SECRET_KEY> \
    --config-path configs/hello_world_deployment_cfg.json \
    --service hello-world
```

Which gives us a unique service id:

```
Your peerId: 12D3KooWAnbFkXk3UFm2MyuNGsSQ6uXHAtjizRC2xv9Q6avN3JBx
"Going to upload a module..."
2022.02.12 00:03:48 [INFO] created ipfs client to /ip4/164.90.164.229/tcp/5001
2022.02.12 00:03:48 [INFO] connected to ipfs
2022.02.12 00:03:50 [INFO] file uploaded
"Now time to make a blueprint..."
"Blueprint id:"
"5efb45e9442ae681d35dcfd4ab40a9927d47b5e16d380d02f71536ba2a2ee427"
"And your service id is:"
"09d9a052-8ccd-4627-9b3a-b72fe6571c87"
```

Take note of the service id, 09d9a052-8ccd-4627-9b3a-b72fe6571c87 in this example but different for you, as we need it to use the service with Aqua.

Congratulations, we just deployed our first reusable service to the Fluence network and we can admire our handiwork on the Fluence [Developer Hub](https://dash.fluence.dev):

![HelloWorld service deployed to peer](./HelloWorld-service-deployed-to-peer.png)

With our newly created service ready to roll, let's move on and put it to work.
