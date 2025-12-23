# x402 Kaia

> Build and demo paywalled experiences on Kaia Kairos Testnet in minutes. This fork is in alpha.

This repo packages everything you need to stand up the x402 payment flow on Kaia: a facilitator, a token-gated resource server, and a sample client that pays for access.

## Prerequisites

- Node.js 18+ and pnpm 10.7+ (corepack enable pnpm if you need to install it)
- git and a POSIX shell (bash/zsh)
- Basic familiarity with editing .env files on your machine

## Quick Start (TL;DR)

```bash
git clone https://github.com/ayo-klaytn/x402-kaia
cd x402-kaia
chmod +x setup.sh
./setup.sh
```

Then open three terminals, all from the repo root:

1. Facilitator – `cd examples/typescript/facilitator && cp .env-local .env` → fill repective private key field → pnpm dev

2. Resource server – `cd examples/typescript/servers/express && cp .env-local .env` → set ADDRESS to the account that receives funds → pnpm dev

3. Client – `cd examples/typescript/clients/axios && cp .env-local .env` → set PRIVATE_KEY for the paying account → pnpm dev

You should see successful payments logged in all three terminals as the client purchase is successful.

## Install project dependencies

The `setup.sh` script installs and builds the monorepo packages and the TypeScript examples.

```bash
chmod +x setup.sh
./setup.sh
```

If you prefer to do it manually:

```bash
cd typescript && pnpm install && pnpm build
cd ../examples/typescript && pnpm install && pnpm build
```

## Run the facilitator

Terminal 1:

```bash
cd examples/typescript/facilitator
cp .env-local .env
```

Update `.env` with your fee payer credentials:

- `PRIVATE_KEY` – account that signs and pays fees
- Optional `PORT` (defaults to `3000`)

OR

Use our hosted testnet facilitator [WIP]

You will need to update the `.env` in the next step

Start the facilitator:

```bash
pnpm dev
```

You should see Server listening on `http://localhost:3000`.

## Run the resource server

Terminal 2:

```bash
cd examples/typescript/servers/express
cp .env-local .env
```

Configure `.env`:

- `FACILITATOR_URL` – typically http://localhost:3000
- `NETWORK` – leave as kairos-testnet
- `ADDRESS` – Kaia account that will receive the payment (can match the facilitator account or be different)

Start the server:

```bash
pnpm dev
```

## Run the client

Terminal 3:

```
cd examples/typescript/clients/axios
cp .env-local .env
```

Set `RESOURCE_SERVER_URL` (default http://localhost:4021), PRIVATE_KEY for the paying account, and leave ENDPOINT_PATH as `/weather`

Run the client:

```bash
pnpm dev
```

The script will:

- Request the protected endpoint
- Receive a `402 Payment Required`
- Pay through the facilitator
- Retry and receive the gated response
- Expect to see logs showing the transaction hash and the JSON payload returned from the server.

## Verify everything is wired up

- The facilitator terminal prints verify and settle calls as they succeed.
- The resource server logs incoming payments and successful responses.
- The client prints the final response body, for example:

```bash
Payment response: {
  success: true,
  transaction: '0xb6d777b62f75d66e2b24e45720ac81fd6b6c238540cd5d850a735b41116bc668',
  network: 'eip155:1001',
  payer: '0xBaf895405a01e759634b2625504ca6bf852475F4',
  requirements: {
    scheme: 'exact',
    network: 'eip155:1001',
    amount: '1000',
    asset: '0x35AD55adDAdCd1867F8d036Ed24F0431c8Ef86A6',
    payTo: '0x7b467A6962bE0ac80784F131049A25CDE27d62Fb',
    maxTimeoutSeconds: 300,
    extra: { name: 'USD Coin', version: '2' }
  }
}
```

## Principles

- **Open standard:** the x402 protocol will never force reliance on a single party
- **HTTP Native:** x402 is meant to seamlessly complement the existing HTTP request made by traditional web services, it should not mandate additional requests outside the scope of a typical client / server flow.
- **Chain and token agnostic:** we welcome contributions that add support for new chains, signing standards, or schemes, so long as they meet our acceptance criteria laid out in [CONTRIBUTING.md](https://github.com/coinbase/x402/blob/main/CONTRIBUTING.md)
- **Trust minimizing:** all payment schemes must not allow for the facilitator or resource server to move funds, other than in accordance with client intentions
- **Easy to use:** x402 needs to be 10x better than existing ways to pay on the internet. This means abstracting as many details of crypto as possible away from the client and resource server, and into the facilitator. This means the client/server should not need to think about gas, rpc, etc.

## Ecosystem

The x402 ecosystem is growing! Check out our [ecosystem page](https://x402.org/ecosystem) to see projects building with x402, including:

- Client-side integrations
- Services and endpoints
- Ecosystem infrastructure and tooling
- Learning and community resources

Want to add your project to the ecosystem? See our [demo site README](https://github.com/coinbase/x402/tree/main/typescript/site#adding-your-project-to-the-ecosystem) for detailed instructions on how to submit your project.

**Roadmap:** see [ROADMAP.md](https://github.com/coinbase/x402/blob/main/ROADMAP.md)

**Documentation:** see [docs/](./docs/) for the GitBook documentation source

## Terms:

- `resource`: Something on the internet. This could be a webpage, file server, RPC service, API, any resource on the internet that accepts HTTP / HTTPS requests.
- `client`: An entity wanting to pay for a resource.
- `facilitator`: A server that facilitates verification and execution of payments for one or many networks.
- `resource server`: An HTTP server that provides an API or other resource for a client.

## Technical Goals:

- Permissionless and secure for clients, servers, and facilitators
- Minimal friction to adopt for both client and resource servers
- Minimal integration for the resource server and client (1 line for the server, 1 function for the client)
- Ability to trade off speed of response for guarantee of payment
- Extensible to different payment flows and networks

## Specification

See `specs/` for full documentation of the x402 standard/

### Typical x402 flow

x402 payments typically adhere to the following flow, but servers have a lot of flexibility. See `advanced` folders in `examples/`.
![](./static/flow.png)

The following outlines the flow of a payment using the `x402` protocol. Note that steps (1) and (2) are optional if the client already knows the payment details accepted for a resource.

1. `Client` makes an HTTP request to a `resource server`.

2. `Resource server` responds with a `402 Payment Required` status and a `PaymentRequired` b64 object return as a `PAYMENT-REQUIRED` header.

3. `Client` selects one of the `PaymentRequirements` returned by the server response and creates a `PaymentPayload` based on the `scheme` & `network` of the `PaymentRequirements` they have selected.

4. `Client` sends the HTTP request with the `PAYMENT-SIGNATURE` header containing the `PaymentPayload` to the resource server.

5. `Resource server` verifies the `PaymentPayload` is valid either via local verification or by POSTing the `PaymentPayload` and `PaymentRequirements` to the `/verify` endpoint of a `facilitator`.

6. `Facilitator` performs verification of the object based on the `scheme` and `network` of the `PaymentPayload` and returns a `Verification Response`.

7. If the `Verification Response` is valid, the resource server performs the work to fulfill the request. If the `Verification Response` is invalid, the resource server returns a `402 Payment Required` status and a `Payment Required Response` JSON object in the response body.

8. `Resource server` either settles the payment by interacting with a blockchain directly, or by POSTing the `Payment Payload` and `Payment PaymentRequirements` to the `/settle` endpoint of a `facilitator server`.

9. `Facilitator server` submits the payment to the blockchain based on the `scheme` and `network` of the `Payment Payload`.

10. `Facilitator server` waits for the payment to be confirmed on the blockchain.

11. `Facilitator server` returns a `Payment Execution Response` to the resource server.

12. `Resource server` returns a `200 OK` response to the `Client` with the resource they requested as the body of the HTTP response, and a `PAYMENT-RESPONSE` header containing the `Settlement Response` as Base64 encoded JSON if the payment was executed successfully.

### Schemes

A scheme is a logical way of moving money.

Blockchains allow for a large number of flexible ways to move money. To help facilitate an expanding number of payment use cases, the `x402` protocol is extensible to different ways of settling payments via its `scheme` field.

Each payment scheme may have different operational functionality depending on what actions are necessary to fulfill the payment.
For example `exact`, the first scheme shipping as part of the protocol, would have different behavior than `upto`. `exact` transfers a specific amount (ex: pay $1 to read an article), while a theoretical `upto` would transfer up to an amount, based on the resources consumed during a request (ex: generating tokens from an LLM).

See `specs/schemes` for more details on schemes, and see `specs/schemes/exact/scheme_exact_evm.md` to see the first proposed scheme for exact payment on EVM chains.

### Schemes vs Networks

Because a scheme is a logical way of moving money, the way a scheme is implemented can be different for different blockchains. (ex: the way you need to implement `exact` on Ethereum is very different from the way you need to implement `exact` on Solana).

Clients and facilitators must explicitly support different `(scheme, network)` pairs in order to be able to create proper payloads and verify / settle payments.
