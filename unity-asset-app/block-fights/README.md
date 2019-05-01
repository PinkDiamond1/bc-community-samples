# Unity App

You scan your face and give birth to a hologram your own personal DNA - defined by an ERC 721 token. You buy your hologram fight moves (each one has a different value, eg punch = 3, kick =4, dance move/roundhouse (combined move) = 10. Your hologram finds another hologram to fight. The holograms agrees on a fight for a set amount of mana (energy). Holograms lock in 3 fight moves (each fight move is a different transaction amount) The fight begins! Holograms trade blows! (transactions weighed by fight move value ). The first hologram who gets to zero health (zero balance) loses. Winning hologram cashes out.

The fight begins with setting up a payment channel on a Layer 2 server hosted on Azure and settled to the Ethereum mainnet when the fight is finished, or challenged when ended early. The Unity engine is used for generating the fighting avatars, their moves, and interactions.

# Layer 2 Payment Channels

Effectively an offchain wallet, hosted in the browser, which utilizes Indra payment channels. Inspired by the SpankCard and Austin Griffith's burner wallet.

Modified for Block Fights with exposed channel details and authorisation for API calls from Unity. Each payment channel becomes a health bar, and each of the opponents attacks, a payment request.

See it live at:
[https://youtu.be/oSgYWfBFkiw](https://youtu.be/oSgYWfBFkiw)

Unity App which was too big for Github can be found here: [Unity Files](https://www.dropbox.com/sh/xeyloezlomcc3pv/AAB4h1HQDGb1JXkAlZJa-ei2a?dl=0) 

## Contents
- [Overview](#overview)
    - [Local Development](#local-development)
    - [Developing Client Alongside](#developing-connext-client-alongside)
- [Integrating into your App](#integrating-into-your-app)
    - [NPM Package](#npm-package)
    - [Autosigner](#autosigner)
    - [Instantiating the Connext Client](#instantiating-the-connext-client)
    - [Making Deposits to Channels](#making-deposits-to-channels)
    - [Withdrawals](#withdrawals)
    - [Collateralization](#collateralization)

## Overview

### Local development

Prerequisites
 - Node 9+
 - Docker
 - Make

1. Make sure you have Indra running on your Azure server. Check out the instructions in the [indra repo](https://github.com/ConnextProject/indra).

TL;DR run:

```
git clone https://github.com/ConnextProject/indra.git
cd indra
npm start
```

2. Deploy

From the Block Fights's project root (eg `git clone https://github.com/ConnextProject/card.git && cd card`), run one of the following:

Using a local webpack dev server:

```
npm install
npm start
```

Using a containerized webpack dev server:

```
make start
```

3. Check it out

 - If you started with `npm start`, browse to `http://localhost:3000`
 - If you started with `make start`, browse to `http://localhost`

4. Run tests

During local development, start the test watcher with:

```
npm run start-test
```

This will start an ongoing e2e tester that will re-run any time the tests are changed. Works well with webpack dev server but you'll have to manually re-trigger the tests after changing the card's source code.

You can also run the more heavy-duty e2e tests that will be run as part of CI integration with:

```
npm run test
```

### Developing Block Fight's Connext Client Alongside

Assuming indra has been cloned & started in the parent directory, run the following from the card repo:

```
bash ops/link-connext.sh
npm restart
```

The above will create a local copy of the connext client that you can mess with. None of you changes in this local client will be reflected in indra, make sure to copy over any changes worth keeping.

### NPM package

The Connext client is a lightweight NPM package that can be found here:

https://www.npmjs.com/package/connext

Installation:

`npm i connext`

`import { getConnextClient } from "connext/dist/Connext.js";`

### Autosigner

In the card, you deposit to a hot wallet that we've generated. This is because interactions with payment channels require a number of signatures and confirmations on behalf of the end user. It's very, very bad UX to make a user sign 4 messages with Metamask to complete a single payment. Our solution is to generate a hot wallet, store the private keys locally, and use a custom Web3 provider to automatically sign transactions with that wallet. We strongly recommend that you follow a similar process.

We instantiate Web3 in App.js using our [custom provider](https://github.com/ConnextProject/card/tree/master/src/utils/web3) as follows:

```
  async setWeb3(rpc) {
    let rpcUrl, hubUrl;

    // SET RPC
    switch (rpc) {
      case "LOCALHOST":
        rpcUrl = localProvider;
        hubUrl = hubUrlLocal;
        break;
      case "RINKEBY":
        rpcUrl = rinkebyProvider;
        hubUrl = hubUrlRinkeby;
        break;
      case "MAINNET":
        rpcUrl = mainnetProvider;
        hubUrl = hubUrlMainnet;
        break;
      default:
        throw new Error(`Unrecognized rpc: ${rpc}`);
    }
    console.log("Custom provider with rpc:", rpcUrl);

    // Ask permission to view accounts
    let windowId;
    if (window.ethereum) {
      window.web3 = new Web3(window.ethereum);
      windowId = await window.web3.eth.net.getId();
    }

    // Set provider options to current RPC
    const providerOpts = new ProviderOptions(store, rpcUrl).approving();

    // Create provider
    const provider = clientProvider(providerOpts);

    // Instantiate Web3 using provider
    const customWeb3 = new Web3(provider);

    // Get network ID to set guardrails
    const customId = await customWeb3.eth.net.getId();

    // NOTE: token/contract/hubWallet ddresses are set to state while initializing connext
    this.setState({ customWeb3, hubUrl });
    if (windowId && windowId !== customId) {
      alert(`Your card is set to ${JSON.stringify(rpc)}. To avoid losing funds, please make sure your metamask and card are using the same network.`);
    }
    return;
  }
  ```

### Instantiating the Connext Client

Once you've instantiated Web3 (whether through a custom provider or the Metamask injection), you need to start up the Connext Client. We do this by creating a Connext object in App.js and then passing it as a prop to any components that require it.

```
async setConnext() {
    const { address, customWeb3, hubUrl } = this.state;

    const opts = {
      web3: customWeb3,
      hubUrl, //url of hub,
      user: address
    };
    console.log("Setting up connext with opts:", opts);

    // *** Instantiate the connext client ***
    const connext = await getConnextClient(opts);
    console.log(`Successfully set up connext! Connext config:`);
    console.log(`  - tokenAddress: ${connext.opts.tokenAddress}`);
    console.log(`  - hubAddress: ${connext.opts.hubAddress}`);
    console.log(`  - contractAddress: ${connext.opts.contractAddress}`);
    console.log(`  - ethNetworkId: ${connext.opts.ethNetworkId}`);
    this.setState({
      connext,
      tokenAddress: connext.opts.tokenAddress,
      channelManagerAddress: connext.opts.contractAddress,
      hubWalletAddress: connext.opts.hubAddress,
      ethNetworkId: connext.opts.ethNetworkId
    });
  }
  ```

  Because channel state changes when users take action, you'll likely want to poll state so that your components are working with the latest channel state:

  ```
    async pollConnextState() {
    // Get connext object
    let connext = this.state.connext;

    // Register listeners
    connext.on("onStateChange", state => {
      console.log("Connext state changed:", state);
      this.setState({
        channelState: state.persistent.channel,
        connextState: state,
        runtime: state.runtime,
        exchangeRate: state.runtime.exchangeRate ? state.runtime.exchangeRate.rates.USD : 0
      });
    });

    // start polling
    await connext.start();
  }
  ```

 ### Making Deposits to Channels

 Depositing to a channel requires invoking `connext.deposit()`, referencing the Connext object that you created when you instantiated the Client. `connext.deposit()` is asynchronous, and accepts a deposit object containing strings of Wei and token values:

 ```
const params = {
  amountWei: "10"
  amountToken: "10"
};

await connext.deposit(params);
```

If you're not using an autosigner, you can simply wrap deposit in a component. If you are using an autosigner, however, we recommend that you have users deposit to the hot wallet that you generated and run a poller that periodically sweeps funds from the wallet into the channel itself (leaving enough in the hot wallet for gas). We implement this in App.js as follows, and set it to run on an interval:

```
async autoDeposit() {
    const { address, tokenContract, customWeb3, connextState, tokenAddress } = this.state;
    const balance = await customWeb3.eth.getBalance(address);
    let tokenBalance = "0";
    try {
      tokenBalance = await tokenContract.methods.balanceOf(address).call();
    } catch (e) {
      console.warn(
        `Error fetching token balance, are you sure the token address (addr: ${tokenAddress}) is correct for the selected network (id: ${await customWeb3.eth.net.getId()}))? Error: ${
          e.message
        }`
      );
    }

    if (balance !== "0" || tokenBalance !== "0") {
      if (eth.utils.bigNumberify(balance).lte(DEPOSIT_MINIMUM_WEI)) {
        // don't autodeposit anything under the threshold
        return;
      }
      // only proceed with deposit request if you can deposit
      if (!connextState || !connextState.runtime.canDeposit) {
        // console.log("Cannot deposit");
        return;
      }

      // Set deposit amounts
      const actualDeposit = {
        amountWei: eth.utils
          .bigNumberify(balance)
          .sub(DEPOSIT_MINIMUM_WEI)
          .toString(),
        amountToken: tokenBalance
      };

      // exit if no deposit
      if (actualDeposit.amountWei === "0" && actualDeposit.amountToken === "0") {
        console.log(`Actual deposit is 0, not depositing.`);
        return;
      }

      console.log(`Depositing: ${JSON.stringify(actualDeposit, null, 2)}`);

      // Make actual deposit
      let depositRes = await this.state.connext.deposit(actualDeposit);
    }
  }
  ```


### Withdrawals

`connext.withdraw()` allows you to withdraw part or all of your funds from a channel to an external address. Called on the Connext object, it accepts parameters indicating the amount to withdraw in tokens and/or Wei. It also includes a `tokensToSell` parameter that, at your/your user's discretion, will automatically swap those tokens for ETH and withdraw your balance in ETH rather than tokens. This is helpful for onboarding to/offboarding from ecosystems with a native token or a specific desired denomination.

Implementation:

```
withdrawalVal: {
        withdrawalWeiUser: "10",
        tokensToSell: "0",
        withdrawalTokenUser: "0",
        weiToSell: "0",
        recipient: "0x0..."
      }

await connext.withdraw(withdrawalVal);
```

Because the card is effectively a hot wallet, we've set our implementation of `connext.withdraw()` to withdraw all funds from the channel; however, in practice users can withdraw however much or little they'd like.


### Collateralization

`connext.requestCollateral()` is used to have the hub collateralize the payee's channel so that they can receive payments.

The hub has a minimum it will collateralize into any channel. This deposit minimum is set to 10 DAI and is enforced on every user deposit, collateral call, but NOT enforced on withdrawals.

The hub is triggered to check the collateral needs of the user after any payment is sent to them, regardless if the payment was successful. As you are implementing the client, you can use a failed payment to kickoff the autocollateral mechanism.

The autocollateral mechanism determines the amount the hub should deposit by checking all recent payments made to recipients, and deposits the 1.5 times the amount to cover the collateral. A minimum of 10 DAI and maximum of 169 DAI are enforced here to reduce collateral burden on the hub.

On cashout, you have the ability to implement "partial withdrawals", which would leave some tokens (and potentially wei) in the channel. On withdrawing, the hub will leave enough DAI in the channel to facilitate exchanges for the remaining user wei balance, up to the exchange limit of 69 DAI. This way, the hub is optimistically collateralizing for future exchanges.

Note that these collateral limitations mean that there is a hard cap to the size of payments the hub can facilitate (169 DAI). However, there is no limit to how much eth the user can store in their side of the channel. The hub will just refuse to facilitate payments or exchanges that it cannot collateralize, but the user will not have to deposit more ETH.

Right now, submitting a withdrawal request with zero value will decollateralize a user's channel entirely.

This presents a few practical challenges: hub operators must decide how to allocate their reserves to minimize (a) the number of payments that fail due to uncollateralized channels and (b) the amount of funds locked up in channels. In addition, implementers should adhere to some best practices to reduce loads on the hub and minimize the chance of payment delays. Because this is new technology, we're still exploring the best ways to handle collateralization and hub reserve management and will update this section as we learn more.
