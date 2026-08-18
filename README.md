# Hello World Example

The repository is intended as part of the tutorial flow for the hello-world example in the [Midnight documentation](https://docs.midnight.network/getting-started/hello-world). It does not operate as a complete repository without the accompanying documentation.

The below documentation will be provided here to "finish" this example.

> **Local deviations from upstream.** This checkout differs from the official
> repository in two ways, both documented in the relevant sections below:
> the `proof-server` service has been removed from `compose.yml` and runs as a
> standalone container, and the contract pragma uses a range rather than an
> exact version pin. See [Known issues](#known-issues).

## Set up project

```bash
git clone git@github.com:midnightntwrk/example-hello-world.git
```

Install dependencies:

```bash
yarn install
```

Requires Node.js 22 or newer.

## Toolchain versions

Compiler and runtime versions must be a compatible pair. The compiler emits
JavaScript that targets a specific `@midnight-ntwrk/compact-runtime` version;
if the installed runtime is older, the contract fails to load at test time
with:

```
CompactError: Version mismatch: compiled code expects 0.18.0-rc.1, runtime is 0.16.0
```

`compact-runtime` is **not** a direct dependency of this project — it arrives
transitively via `@midnight-ntwrk/midnight-js-protocol`. Do not `yarn add` it
directly; that creates a conflict with what the protocol package expects.
Pin the compiler to match the runtime instead:

```bash
compact compile --version                  # what you have
yarn list --pattern compact-runtime        # what the repo needs
compact update <version>                   # install and set as default
```

See the [compatibility matrix](https://docs.midnight.network/relnotes/support-matrix)
for validated compiler ↔ runtime ↔ ledger ↔ node combinations.

If you work across more than one machine, pin the same compiler version on
each. Different compiler versions produce different verifier keys, and
therefore different contract addresses, from identical source.

## Create the contract file

Create a new file named `hello-world.compact` in the `contracts` directory:

```bash
touch contracts/hello-world.compact
```

Open this file in VS Code:
```bash
code .
```

## Create the Compact Smart Contract

```compact
pragma language_version >= 0.16;

export ledger message: Opaque<"string">;

export circuit storeMessage(newMessage: Opaque<"string">): [] {
  message = disclose(newMessage);
}
```
- `pragma language_version` specifies which version of Compact your contract uses. Use a range (`>= 0.16`) rather than an exact pin (`0.23`). An exact pin fails against any compiler whose language version differs, with `language version X mismatch` at line 1.
- `ledger message` creates a state variable named `message` that stores a string value in the on-chain state. On-chain state is public and persistent on the blockchain.
- `circuit storeMessage` is a Compact circuit (function) that defines the logic to modify on-chain state.
- `newMessage: Opaque<"string">` is the input parameter. *Circuit parameters are always private by default.* The `disclose()` function marks the private value as safe to store publicly. Without it, trying to assign `newMessage` directly to the ledger returns a compiler error.

`Opaque<"string">` is an unstructured blob. It carries no privacy guarantees
of its own beyond what the surrounding circuit enforces — for real
applications, use typed fields with explicit commitments and constraints.

## Compile the contract

Compiling transforms your Compact code into zero-knowledge circuits, generates cryptographic keys, 
and creates TypeScript APIs and a JavaScript implementation for the contract to be used by DApps. 

Run the compiler from the contracts folder:

```bash
cd contracts
compact compile hello-world.compact managed/hello-world
cd ..
```

The output path matters: `contracts/index.ts` imports from
`./managed/hello-world/contract/index.js`, so the artifacts must land at
`contracts/managed/hello-world`.

You should see output similar to:

```
Compiling 1 circuits:
  circuit "storeMessage" (k=6, rows=24)
```

The exact `rows` count varies by compiler version and is not a correctness
check. It is the constraint count, and it predicts proving time — worth
watching as circuits grow.

On a first compile the compiler may also download public parameters for the
circuit's `k` value from `https://srs.midnight.network/`. This is expected.

The compilation process will:
1. Parse and validate your Compact code.
2. Generate zero-knowledge circuits from your logic.
3. Create proving and verifying keys for the circuits.
4. Generate the TypeScript API and JavaScript implementation for the contract.

When compilation completes, you'll see a new directory structure:

```
contracts/
├── managed/
|   └── hello-world/
|        ├── compiler/
|        ├── contract/
|        ├── keys/
|        └── zkir/
└── hello-world.compact
└── index.ts
```

Here's what each directory contains:

- **contract/**: The compiled contract artifacts, which includes the JavaScript implementation and type definitions.
- **keys/**: Cryptographic proving and verifying keys that enable zero-knowledge proofs.
- **zkir/**: Zero-Knowledge Intermediate Representation—the bridge between Compact and the ZK backend.
- **compiler/**: Compiler-generated JSON output that other tools can use to understand the contract structure.

The `managed/` directory is generated. It is not checked in, so a fresh
clone must be compiled before the tests will run — otherwise the suite fails
with `Cannot find module './managed/hello-world/contract/index.js'`.

## Start the local environment

Be sure the Docker engine is running.

The proof server runs as a standalone container in this checkout. Start it
once; the restart policy brings it back after reboots and Docker restarts:

```bash
docker run -d --name proof-server --restart unless-stopped \
  -p 127.0.0.1:6300:6300 \
  midnightntwrk/proof-server:8.1.0 midnight-proof-server -v
```

On first start it downloads roughly 30 MB of Zswap and Dust key material
before binding port 6300. Wait for `starting service:
"actix-web-service-0.0.0.0:6300"` before running the tests:

```bash
docker logs -f proof-server
```

Then start the node and indexer from the project root:

```bash
yarn env:up
```

Verify all three are up:

```bash
docker ps
```

| Service      | Port           |
| ------------ | -------------- |
| proof-server | 127.0.0.1:6300 |
| indexer      | 127.0.0.1:8088 |
| node         | 127.0.0.1:9944 |

## Deploy Contract to Local Devnet

Now that your contract is compiled, it needs to be deployed to the blockchain so that you can interact with it.

To deploy the contract, you'll need a wallet. The local devnet package comes with 3 pre-funded wallets, so no faucet or seed configuration is needed for local work.

Run the deployment script:
```bash
yarn test:local
```

The test script will begin to show output from your local devnet and will progress the contract deployment and interaction programatically:

```
[05:08:00.647] INFO (55308): Wallet sync complete after 22 emissions
[05:08:00.655] INFO (55308): Providers initialized on 'local'. Ready to test!
[05:08:00.662] INFO (55308): Creating private state...
[05:08:20.560] INFO (55308): Setting the contract address...
[05:08:20.560] INFO (55308): Contract deployed at: 1c48fea2ae2e4d0c638633f399e1b98491a85e629e3ec276fce7fca82130ec6a
 ✓ src/test/hw.test.ts (2 tests) 39942ms
   ✓ Hello World Contract (local) (2)
     ✓ Deploys the contract  20253ms
     ✓ Stores Hello World!  18545ms
```

A `disconnected from ws://127.0.0.1:9944/: 1000:: Normal Closure` line during
the run is the wallet closing its subscription cleanly, not an error.

Stop the node and indexer:
```bash
yarn env:down
```

The standalone proof server keeps running. Stop it deliberately when you want
it down:

```bash
docker stop proof-server     # docker start proof-server to resume
```

Hello World! You are now ready to explore [Tutorials](https://docs.midnight.network/category/tutorials) for more detailed instructions on building DApps on Midnight!

## Deploy Contract to Live Testnet

Midnight's current public environments are **Preview**, **Preprod**, and
**Mainnet**. There is no `testnet` — that network name and its endpoints are
retired, and older scaffolds that default to it will silently target a dead
host.

To run this test script on Preview or Preprod:
1. Generate a wallet on the given network and fund it manually via the network's faucet page — [Preview](https://midnight-tmnight-preview.nethermind.dev/) or [Preprod](https://midnight-tmnight-preprod.nethermind.dev/). The faucet is a human-facing web page (no programmatic drip endpoint), so the test suite assumes the seed you supply is already funded with tNIGHT. tDUST can be delegated in 1AM or Lace Carbon (coming soon). See [Environments and endpoints](https://docs.midnight.network/relnotes/network) for reference.
1. Create `.env.<network>` and populate it based on the information in `.env.<network>.example` in this repository.
1. Start the proof server: `yarn proof:up`
1. Start the test: `yarn test:<network>` -- the wallet will sync to the network and advance the test suite programmatically.

The faucet accepts only addresses for its own network. A Preprod faucet
rejects a Preview address, and vice versa, with `Provided address is invalid`.
Check the address prefix your wallet generates before assuming the faucet is
broken.

## Known issues

### Proof server fails under compose

The `proof-server` service has been removed from `compose.yml` in this
checkout. Under compose it exits with code 1 during startup, failing every
request to `https://srs.midnight.network/` in a few milliseconds:

```
WARN midnight_base_crypto::data_provider: error sending request for url
(https://srs.midnight.network/bls_midnight_2p10). Giving up.
Error: Custom { kind: InvalidData, error: "Failed to fetch data from
https://srs.midnight.network/bls_midnight_2p10 after 3 attempts. Giving up." }
```

The same image with the same ports, environment, and network fetches
everything successfully when started with `docker run`. Ruled out during
diagnosis: DNS (resolves inside containers on both the default and the
compose network), CDN availability (`/9/` and `/8/` paths both exist), CA
certificates (mounting the host bundle changes nothing), the compose network
itself, and the `127.0.0.1` host_ip binding. Root cause not identified.

Workaround: run it standalone, as described above. Nothing else in the stack
connects to it over the Docker network — the test suite reaches it at
`localhost:6300` from the host — so it does not need to share a network with
the node and indexer.

### Healthcheck cannot run

The original `proof-server` healthcheck was:

```yaml
test: ['CMD-SHELL', 'echo > /dev/tcp/127.0.0.1/6300']
```

The image is distroless and has no shell, so `CMD-SHELL` cannot execute, and
`/dev/tcp` is a bash builtin rather than a real device. This check can never
pass, which means `docker compose up --wait` will not report the service
healthy even when it is serving correctly.
