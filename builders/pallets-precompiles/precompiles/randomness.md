---
title: Randomness Precompile
description: Learn about the sources of VRF randomness on Moonbeam and how to use the randomness precompile and consumer interface to generate on-chain randomness.
keywords: solidity, ethereum, randomness, VRF, moonbeam, precompiled, contracts
---

# Interacting with the Randomness Precompile

![Randomness Moonbeam Banner](/images/builders/pallets-precompiles/precompiles/randomness/randomness-banner.png)

## Introduction {: #introduction } 

Moonbeam utilizes verifiable random functions (VRF) to generate randomness that can be verified on-chain. A VRF is a cryptographic function that takes some input and produces random values, along with a proof of authenticity that these random values were generated by the submitter. The proof can be verified by anyone to ensure the random values generated were calculated correctly.

There are two available sources of randomness that provide random inputs based on block producers' VRF keys and past randomness results: [local VRF](#local-vrf) and [BABE epoch randomness](#babe-epoch-randomness). Local VRF is determined directly within Moonbeam using the collator of the block's VRF key and the last block's VRF output. On the other hand, [BABE](https://wiki.polkadot.network/docs/learn-consensus#block-production-babe){target=_blank} epoch randomness is based on all the VRF produced by the relay chain validators during a complete [epoch](https://wiki.polkadot.network/docs/glossary#epoch){target=_blank}.

For more information on the two sources of randomness, how the request and fulfillment process works, and security considerations, please refer to the [Randomness on Moonbeam](/learn/features/randomness){target=_blank} page.

Moonbeam provides a randomness precompile, which is a Solidity interface that enables smart contract developers to generate randomness via local VRF or BABE epoch randomness using the Ethereum API. Moonbeam also provides a randomness consumer Solidity contract that your contract must inherit from in order to consume fulfilled randomness requests.

This guide will show you how to use the randomness precompile and randomness consumer contract to create a lottery where the winners will randomly be selected. You'll also learn how to interact with the randomness precompile directly to perform actions such as purging an expired randomness request.

The randomness precompile is currently only available on Moonbase Alpha and is located at the following address:

=== "Moonbase Alpha"
     ```
     {{ networks.moonbase.precompiles.randomness }}
     ```

## The Randomness Solidity Interface {: #the-randomness-interface }

[Randomness.sol](https://github.com/PureStake/moonbeam/blob/master/precompiles/randomness/Randomness.sol#L4-L11){target=_blank} is a Solidity interface that allows developers to interact with the precompile's methods.

The interface includes functions, constants, events, and enums, as covered in the following sections.

### Functions {: #functions }

The interface includes the following functions:

- **relayEpochIndex**() — returns the current relay epoch index, where an epoch represents real time and not a block number
- **requiredDeposit**() — returns the deposit required to perform a randomness request
- **getRequestStatus**(*uint256* requestId) — returns the request status of a given randomness request
- **getRequest**(*uint256* requestId) — returns the request details of a given randomness request
- **requestLocalVRFRandomWords**(*address* refundAddress, *uint256* fee, *uint64* gasLimit, *bytes32* salt, *uint8* numWords, *uint64* delay) — request random words generated from the parachain VRF
- **requestRelayBabeEpochRandomWords**(*address* refundAddress, *uint256* fee, *uint64* gasLimit, *bytes32* salt, *uint8* numWords) — request random words generated from the relay chain BABE consensus 
- **fulfillRequest**(*uint256* requestId) — fulfill the request which will call the consumer contract method [`fulfillRandomWords`](#:~:text=rawFulfillRandomWords(uint256 requestId, uint256[] memory randomWords)). Fees of the caller are refunded if the request is fulfillable
- **increaseRequestFee**(*uint256* requestId, *uint256* feeIncrease) — increases the fee associated with a given randomness request. This is needed if the gas price increases significantly before the request is fulfilled
- **purgeExpiredRequest**(*uint256* requestId) — removes a given expired request from storage and transfers the request fees to the caller and the deposit back to the 

Where the inputs that need to be provided can be defined as:

- **requestId** - the ID of the randomness request
- **refundAddress** - the address receiving the left-over fees after the fulfillment
- **fee** - the amount to set aside to pay for the fulfillment
- **gasLimit** - the gas limit to use for the fulfillment
- **salt** - a string that is mixed with the randomness seed to obtain different random words
- **numWords** - the number of random words requested, up to the maximum number of random words
- **delay** - the number of blocks that must pass before the request can be fulfilled. This value will need to be between the minimum and maximum number of blocks before a local VRF request can be fulfilled
- **feeIncrease** - the amount to increase fees by

### Constants {: #constants }

The interface includes the following constants:

- **MAX_RANDOM_WORDS** - the maximum number of random words being requested 
- **MIN_VRF_BLOCKS_DELAY** - the minimum number of blocks before a request can be fulfilled for local VRF requests
- **MAX_VRF_BLOCKS_DELAY** - the maximum number of blocks before a request can be fulfilled for local VRF requests
- **REQUEST_DEPOSIT_AMOUNT** - the deposit amount needed to request random words. There is one deposit per request

=== "Moonbase Alpha"
    |        Variable        |                             Value                              |
    |:----------------------:|:--------------------------------------------------------------:|
    |    MAX_RANDOM_WORDS    |   {{ networks.moonbase.randomness.max_random_words }} words    |
    |  MIN_VRF_BLOCKS_DELAY  | {{ networks.moonbase.randomness.min_vrf_blocks_delay }} blocks |
    |  MAX_VRF_BLOCKS_DELAY  | {{ networks.moonbase.randomness.max_vrf_blocks_delay }} blocks |
    | REQUEST_DEPOSIT_AMOUNT | {{ networks.moonbase.randomness.req_deposit_amount.dev }} DEV  |

### Events {: #events }

The interface includes the following events:

- **FulfillmentSucceeded**() - emitted when the request has been successfully executed
- **FulfillmentFailed**() - emitted when the request has failed to execute fulfillment

### Enums {: #enums }

The interface includes the following enums:

- **RequestStatus** - the status of the request, which can be `DoesNotExist` (0), `Pending` (1), `Ready` (2), or `Expired` (3)
- **RandomnessSource** - the type of the randomness source, which can be `LocalVRF` (0) or `RelayBabeEpoch` (1)

## The Randomness Consumer Solidity Interface {: #randomness-consumer-solidity-interface }

The [`RandomnessConsumer.sol`](https://github.com/PureStake/moonbeam/blob/4e2a5785424be6faa01cd14e90155d9d2ec734ee/precompiles/randomness/RandomnessConsumer.sol){target=_blank} Solidity interface makes it easy for smart contracts to interact with the randomness precompile. Using the randomness consumer ensures the fulfillment comes from the randomness precompile. 

The consumer interface includes the following functions:

- **fulfillRandomWords**(*uint256* requestId, *uint256[] memory* randomWords) - handles the VRF response for  given request. This method is triggered by a call to `rawFulfillRandomWords`
- **rawFulfillRandomWords**(*uint256* requestId, *uint256[] memory* randomWords) - executed when the [`fulfillRequest` function](#:~:text=fulfillRequest(uint256 requestId)) of the randomness precompile is called. The origin of the call is validated, ensuring the randomness precompile is the origin, and then the `fulfillRandomWords` method is called

## Request & Fulfill Process {: #request-and-fulfill-process }

To consume randomness, you must have a contract that does the following:

  - Imports the `Randomness.sol` precompile and `RandomnessConsumer.sol` interface
  - Inherits from the `RandomnessConsumer.sol` interface
  - Requests randomness through the precompile's [`requestLocalVRFRandomWords` method](#:~:text=requestLocalVRFRandomWords) or [`requestRelayBabeEpochRandomWords` method](#:~:text=requestRelayBabeEpochRandomWords), depending on the source of randomness you want to use
  - Requests fulfillment through the precompile's [`fulfillRequest` method](#:~:text=fulfillRequest)
  - Consumes randomness through a `fulfillRandomWords` method with the same [signature as the `fulfillRandomWords` method](#:~:text=fulfillRandomWords(uint256 requestId, uint256[] memory randomWords)) of the `RandomnessConsumer.sol` contract

When randomness is requested through the precompile's `requestLocalVRFRandomWords` or `requestRelayBabeEpochRandomWords` method, a fee is set aside to pay for the fulfillment of the request. When using local VRF, to increase unpredictability, a specified delay period (in blocks) must pass before the request can be fulfilled. At the very least, the delay period must be greater than one block. For BABE epoch randomness, you do not need to specify a delay but can fulfill the request at the beginning of the 2nd epoch following the current one.

After the delay, fulfillment of the request can be manually executed by anyone through the `fulfillRequest` method using the fee that was initially set aside for the request.

When fulfilling the randomness request via the precompile's `fulfillRequest` method, the [`rawFulfillRandomWords`](#:~:text=rawFulfillRandomWords(uint256 requestId, uint256[] memory randomWords)) function in the `RandomnessConsumer.sol` contract will be called, which will verify that the sender is the randomness precompile. From there, [`fulfillRandomWords`](#:~:text=fulfillRandomWords(uint256 requestId, uint256[] memory randomWords)) is called and the requested number of random words are computed using the current block's randomness result and a given salt and returned. If the fulfillment was successful, the [`FulfillmentSucceeded` event](#:~:text=FulfillmentSucceeded) will be emitted; otherwise the [`FulfillmentFailed` event](#:~:text=FulfillmentFailed) will be emitted. 

For fulfilled requests, the cost of execution will be refunded from the request fee to the caller of `fulfillRequest`. Then any excess fees and the request deposit are transferred to the specified refund address.

Your contract's `fulfillRandomWords` callback is responsible for handling the fulfillment. For example, in a lottery contract, the callback would use the random words to choose a winner and payout the winnings.

If a request expires it can be purged through the precompile's [`purgeExpiredRequest` function](/buildxers/pallets-precompiles/precompiles/randomness/#:~:text=purgeExpiredRequest){target=_blank}. When this function is called the request fee is paid out to the caller and the deposit will be returned to the original requester.

The happy path for a randomness request is shown in the following diagram:

![Randomness request happy path diagram](/images/learn/features/randomness/randomness-1.png)

## Interact with the Solidity Interfaces via Lottery Contract {: #interact-with-the-solidity-interfaces }

In the following sections of this tutorial, you'll learn how to interact with the randomness precompile, in addition to a lottery contract that requires you to have multiple accounts which you can use to participate in a lottery. The default lottery contract sets the minimum number of participants to three, however you can feel free to change the number in the contract. 

### Checking Prerequisites {: #checking-prerequisites } 

Assuming you use the default contract, you will need to have the following:

- [MetaMask installed and connected to Moonbase Alpha](/tokens/connect/metamask/){target=_blank}
- Create or have three accounts on Moonbase Alpha to test out the lottery contract
- All of the accounts will need to be funded with `DEV` tokens.
 --8<-- 'text/faucet/faucet-list-item.md'

### Example Lottery Contract {: #example-contract }

In this tutorial, you'll interact with a lottery contract that uses the randomness precompile and consumer. You'll be generating random words which will be used to select the winner of the lottery fairly. You can find a copy of the lottery contract that will be used for this tutorial, [`RandomnessLotteryDemo.sol`](https://raw.githubusercontent.com/PureStake/moonbeam-docs/blob/master/.snippets/code/randomness/RandomnessLotteryDemo.sol){target=_blank}, in the Moonbeam Docs GitHub repository.

The lottery contract imports the `Randomness.sol` precompile and the `RandomnessConsumer.sol` interface, and inherits from the consumer contract. In the constructor of the contract, you can specify the source of randomness to be either local VRF or BABE epoch randomness.

In general, the lottery contract includes functionality to check the status of the randomness request which will be used to determine whether the lottery is still accepting participants, if it has started, or if it has expired. It will use the `requestLocalVRFRandomWords` or `requestRelayBabeEpochRandomWords` function to select the random words, depending on which source of randomness you want to use. In addition, it will implement the `fulfillRandomWords` method and the callback will fulfill the request and use the random words to randomly pick the lottery winners.

There are also some constants in the contract that can be edited as you see fit, especially the `SALT_PREFIX` which can be used to produce unique results.

### Remix Set Up {: #remix-set-up } 

You can interact with the randomness precompile and consumer using [Remix](https://remix.ethereum.org/){target=_blank}. To add the interfaces to Remix and follow along with the tutorial, you will need to:

1. Get a copy of [`RandomnessLotteryDemo.sol`](https://github.com/PureStake/moonbeam/blob/4e2a5785424be6faa01cd14e90155d9d2ec734ee/tests/contracts/solidity/RandomnessLotteryDemo.sol){target=_blank}
2. Paste the file contents into a Remix file named **RandomnessLotteryDemo.sol**

![Add contracts to Remix](/images/builders/pallets-precompiles/precompiles/randomness/randomness-1.png)

### Compile & Deploy the Lottery Contract {: #compile-lottery }

Next, you will need to compile the `RandomnessLotteryDemo.sol` file in Remix:

1. Make sure that you have the **RandomnessLotteryDemo.sol** file open
2. Click on the **Compile** tab, second from top
3. To compile the contract, click on **Compile RandomnessLotteryDemo.sol**

![Compile RandomnessLotteryDemo](/images/builders/pallets-precompiles/precompiles/randomness/randomness-2.png)

If the contract was compiled successfully, you will see a green checkmark next to the **Compile** tab.

Once the contract has been compiled, you can deploy the contract by taking the following steps:

1. Click on the **Deploy and Run** tab directly below the **Compile** tab in Remix
2. Make sure **Injected Provider - Metamask** is selected in the **ENVIRONMENT** dropdown. Once selected, you might be prompted by MetaMask to connect your account to Remix
3. Make sure the correct account is displayed under **ACCOUNT**
4. You'll need to pay the required deposit for a randomness request. For this contract, the required deposit is {{ networks.moonbase.randomness.req_deposit_amount.dev }} DEV. So set the value to `{{ networks.moonbase.randomness.req_deposit_amount.wei }}` and choose **Wei** from the dropdown on the right
5. Ensure **RandomnessLotteryDemo - RandomnessLotteryDemo.sol** is selected in the **CONTRACT** dropdown
6. Next to **Deploy** enter the source of randomness. This corresponds to the [`RandomnessSource`](#enums) enum. For local VRF, enter `0`, and for BABE epoch randomness, enter `1`. To follow along with this example, you can select `0` and click **Deploy**
7. Confirm the MetaMask transaction that appears by clicking **Confirm**

![Deploy RandomnessLotteryDemo](/images/builders/pallets-precompiles/precompiles/randomness/randomness-3.png)

The **RANDOMNESSLOTTERYDEMO** contract will appear in the list of **Deployed Contracts**.

### Participate in the Lottery {: #participate-in-lottery }

The default contract has a minimum requirement of three participants. To participate you can take the following steps:

1. Make sure you've switched to the account you want to participate with in MetaMask. You can verify the account that is connected under **ACCOUNT** 
2. Enter the amount you want to contribute to the lottery in the **VALUE** field. It must be greater than or equal to the `PARTICIPATION_FEE` which is set to `100000 gwei` in the default contract
3. Click on **participate**
4. Confirm the transaction in MetaMask

![Participate in the lottery](/images/builders/pallets-precompiles/precompiles/randomness/randomness-4.png)

Since there is a minimum of three participants required to start the lottery, you'll need to repeat these steps until you've participated from three different accounts.

### Start the Lottery {: #start-the-lottery }

If you take a closer look at the `RandomnessLotteryDemo.sol` contract's `startLottery` function, you'll notice that it has the `onlyOwner` modifier. As such, you will need to make sure that you switch back to the account that deployed the lottery contract before starting the lottery.

To start the lottery and submit the randomness request, which will call the precompile's `requestLocalVRFRandomWords`, you can take the following steps:

1. Confirm the account is the owner
2. To start the lottery you need to pay a fee which will be used to fulfill the randomness request. You can set the **VALUE** to `200000` and choose **Gwei**. The excess fee will be returned to the `msg.sender`
3. Click on **startLottery**
4. Confirm the transaction in MetaMask

![Start the lottery](/images/builders/pallets-precompiles/precompiles/randomness/randomness-5.png)

Once the transaction goes through, the lottery will start and no more participants will be able to join. Before you can fulfill the randomness request in order to pick the winners, you'll need to wait the delay. The default `VRF_BLOCKS_DELAY` is set to `2` blocks.

### Pick the Winners {: #pick-the-winners }

To fulfill the request, you can do so using the `fulfillRequest` function which will use the contract's `requestId` variable to call the `fulfillRequest` function of the randomness precompile. If successful, the request will be fulfilled and generate the random words and execute the `fulfillRandomWords` function defined in the `RandomnessLotteryDemo.sol` contract. The `fulfillRandomWords` function callback then calls `pickWinners` and the jackpot is distributed to the randomly selected winners. In addition, the cost of execution will be refunded from the request fee to the caller of `fulfillRequest`. Then any excess fees and the request deposit are transferred to the specified refund address.

You can initiate the fulfillment from any account after the delay has passed, to do so you'll need to:

1. Ensure you're connected to the account that you want to fulfill the request from, it can be any account you choose
2. Click on **fulfillRequest**
3. Confirm the transaction in MetaMask

![Fulfill the randomness request](/images/builders/pallets-precompiles/precompiles/randomness/randomness-6.png)

If the transaction reverts with the following error you may need to call `increaseRequestFee`:

```
{ 
  "code": -32603,
  "message": "VM Exception while processing transaction: revert",
  "data": "0x476173206c696d69742061742063757272656e74207072696365206d757374206265206c657373207468616e206665657320616c6c6f74746564"
}
```

The `data` field converted to ASCII text reads: `Gas limit at current price must be less than fees allotted`. As such, you can use the `increaseRequestFee` function to increase the fees for the transaction and try again.

Once the transaction goes through, you can take a look at the transaction details in the Remix console. If you scroll down til you see the **logs**, you should see four events and the event details. The events you should see are:

- **`"event": "Ended"`** - event sent when the lottery ends, which emits the number of participants, the jackpot, and the total winners. Defined in the `RandomnessLotteryDemo.sol` contract
- **`"event": "Awarded"`** - event sent when a winner is awarded, which should get emitted twice since there are two winners per the default contract. It emits the winner, the random word, and the amount won. Defined in the `RandomnessLotteryDemo.sol` contract
- **`"event": "FulFillmentSucceeded"`** - event sent when a request has been fulfilled successfully. Defined in the `Randomness.sol` precompile

![Fulfill the randomness request](/images/builders/pallets-precompiles/precompiles/randomness/randomness-7.png)

Congratulations! You've successfully used the randomness precompile and consumer to participate in and start a lottery, and use the generated random words to select a winner.

## Interact with the Precompile Solidity Interface Directly {: #interact-directly }

In addition to interacting with the randomness precompile via a smart contract, you can also interact with it directly in Remix to perform operations such as creating a randomness request, checking on the status of a request, and purging expired requests. Remember, you need to have a contract that inherits from the consumer contract in order to fulfill requests, as such if you fulfill a request using the precompile directly there will be no way to consume the results.

### Remix Set Up {: #remix-set-up } 

To add the interfaces to Remix and follow along with this section of the tutorial, you will need to:

1. Get a copy of [`Randomness.sol`](https://github.com/PureStake/moonbeam/blob/master/precompiles/randomness/Randomness.sol){target=_blank}
2. Paste the file contents into a Remix file named **Randomness.sol**

![Add precompile to Remix](/images/builders/pallets-precompiles/precompiles/randomness/randomness-8.png)

### Compile & Access the Randomness Precompile {: #compile-randomness }

Next, you will need to compile the `Randomness.sol` file in Remix. To get started, make sure you have the **Randomness.sol** file open and take the following steps:

1. Click on the **Compile** tab, second from top
2. To compile the contract, click on **Compile Randomness.sol**

If the contract was compiled successfully, you will see a green checkmark next to the **Compile** tab.

Instead of deploying the randomness precompile, you will access the interface given the address of the precompiled contract:

1. Click on the **Deploy and Run** tab directly below the **Compile** tab in Remix. Please note the precompiled contract is already deployed
2. Make sure **Injected Provider - Metamask** is selected in the **ENVIRONMENT** dropdown. Once selected, you might be prompted by MetaMask to connect your account to Remix
3. Make sure the correct account is displayed under **ACCOUNT**
4. Ensure **Randomness - Randomness.sol** is selected in the **CONTRACT** dropdown. Since this is a precompiled contract, there is no need to deploy any code. Instead we are going to provide the address of the precompile in the **At Address** Field
5. Provide the address of the batch precompile: `{{ networks.moonbase.precompiles.randomness }}` and click **At Address**

![Access the address](/images/builders/pallets-precompiles/precompiles/randomness/randomness-9.png)

The **RANDOMNESS** precompile will appear in the list of **Deployed Contracts**. You will use this to fulfill the randomness request made from the lottery contract later on in this tutorial.

### Get Request Status & Purge Expired Request {: #get-request-status-and-purge }

Anyone can purge expired requests. You do not need to be the one who requested the randomness to be able to purge it. When you purge an expired request, the request fees will be transferred to you, and the deposit for the request will be returned to the original requester.

To purge a request, first you have to make sure that the request has expired. To do so, you can verify the status of a request using the `getRequestStatus` function of the precompile. The number that is returned from this call corresponds to the index of the value in the [`RequestStatus`](#enums) enum. As a result, you'll want to verify the number returned is `3` for `Expired`.

Once you've verified that the request is expired, you can call the `purgeExpiredRequest` function to purge the request. 

To verify and purge a request, you can take the following steps:

1. Expand the **RANDOMNESS** contract
2. Enter the request ID of the request you want to verify has expired and click on **getRequestStatus**
3. The response will appear just underneath the function. Verify that you received a `3`
4. Expand the **purgeExpiredRequest** function and enter the request ID
5. Click on **transact**
6. MetaMask will pop-up and you can confirm the transaction

![Purge an exired request](/images/builders/pallets-precompiles/precompiles/randomness/randomness-10.png)

Once the transaction goes through, you can verify the request has been purged by calling the **getRequestStatus** function again with the same request ID. You should receive a status of `0`, or `DoesNotExist`. You can also expect the amount of the request fees to be transferred to your account.