
Authors: Algys Iyevlev, Aleksey Makeyev, Vyacheslav Melnikov, Sergey Prilutsky, Vladimir Khramov.

# Configuring, launching and interacting with contracts: constructors

The same smart contract may often be utilised to solve various business cases. Where a case is simple, parameters of
constructor function for a smart contract vary from case to case; in more complicated cases, some code snippets may
be included into the resulting smart contract source code, some are excluded. For instance, in a simple case, in ERC20 token the totalSupply
is set as the parameter for the constructor function (from this point on, we will interpret the "smart contract constructor"
as an engine provided and utilised on Smartz platform); not to confuse it with a constructor in terms
of object oriented programming, we will call the latter a "constructor function". In a more complicated case, a burn action may be
available in a token. This can be implemented by means of adding a correspondent feature into the contract code, or through
inheritance: by means of connecting a token contract that has a Burn function (see `Burnable Token` in [erc20_token_constructor.py], for instance
in [erc20_token_constructor.py](https://github.com/smartzplatform/SDK/blob/ba8230d39e94f70a30e716f4f1e48ddd4e702432/constructor_examples/erc20_token_constructor.py)) as a parent contract.
This may require adding an additional logic that helps align the parent-class contract with the child-class contract.
(see `transfer`, `transferFrom`, `burn` in [SmartzToken](https://github.com/smartzplatform/sale/blob/6a00b30ccaa3dabc515ad7dfd29bbd85848c9603/contracts/SmartzToken.sol)).

After a smart contract is launched in blockchain, it's required to provide smart contract users with DApp
that enables convenient interaction with the smart contract. Here we should consider some observations
described in the paragraph above: having passed a number of preliminary steps of various complexity, you will be able to use the same DApp across
multiple similar business cases. Besides, a simple DApp may be automatically generated based on
the smart contract.

We consider it to be economically unviable to engage a developer for resolution of each new case. On the other hand,
there's a kind of joke that says that "a real programmer must be lazy". It illustrates the idea that the code
written by a good programmer is properly structured and convenient to use, which means that a programmer will spend less time
to complete a new similar task. Based on these assumptions, we have developed a concept of a smart contract
constructor. The underlying mechanism of a smart contract constructor implements exactly the ideas described above.

## Functioning of smart contract constructors

There are three major parties that take part in configuring, launching and interacting with the smart contract: developer
of the smart contract, user, and Smartz platform. The developer arranges a smart contract in the form of a constructor and makes it
as reusable as he deems appropriate. The user launches the smart contract in blockchain,
interacts with it there, and securely stores private keys of their blockchain accounts without ever
making the keys available to anyone else. The Smartz platform executes the constructor and all the auxiliary operations, which simplifies development and
the use of smart contracts.

### Configuring and launching a smart contract

#### Interaction between Smartz and smart contract constructor

Smart contract constructor (hereinafter: "constructor") is a source code written in any given programming
language (Python, javascript, Java, Perl, Ruby, PHP etc.) that has a certain programming interface.
Interaction with Smartz represents a call for any methods by the platform.

The constructor code, and therefore,
the contract codes that it generates, is an open-source code, which makes it possible to audit the code. Before execution
a necessary compilation occurs as well as caching of the obtained output (e.g. in case of Java we obtain a .class file; in case of Python — 
a .pyc file).

Low-level interaction with the constructor occurs within a dedicated constructor service. Its logic is described below.
If you need to call a constructor's method, the platform will execute the compiled constructor
code in a siloed Docker container. Memory usage, process count, and process runtime are restricted in the container,
as well as interaction with the drive subsystem and network. A constructor file and library that provides
access to internal API Smartz are assembled into the container. Creating such a container is quite a resource-intensive operation (appr. 1–2
seconds), so containers are normally reused. The statistical part of the constructor service that is executed in the container
loads a particular constructor using the mechanics of the related language (e.g., in case of Java this is ClassLoader, in case of
Python this is importlib) and calls the required method after sending parameters as call arguments. The execution result is returned
to the constructor service.

#### Input data for constructor

Constructor describes the input data that it expects user to provide.
This data may be numerical values, strings, blockchain addresses or even more complicated data (e.g., collections and structures). Besides,
it's recommended to apply restrictions to the data (e.g., set maximum number of signatures in the multi-signed contract), so that
users can adjust the data even in the interface, and developers don't have to repeatedly
implement data validation logic. We call this description the "data schema".

For the purposes of describing the schema, we use ready-to-use near-standard solutions intended for any given domain. This
may provide the following advantages:
* Some developers are already aware of them. Those who are not, may reuse the knowledge gained while working on
other projects.
* There are ready-to-use tools and libraries, for instance, for validating and generating interfaces.

Smartz is going to support the following formats of input data description [json schema](http://json-schema.org) and
[OpenAPI Specification](https://swagger.io/specification/) (used in [swagger](https://swagger.io/)). Besides,
Smartz provides ready-to-use descriptions for some data types, e.g., Ethereum address or unix timestamp in terms of
appropriate schemas.

Again, the constructor's data schema may be optionally complemented with display schema (`ui_schema`). This schema impacts rendering
of the data input interface.

When a user wants to configure and launch a smart contract, they go to the appropriate Smartz page.
From this page, they send an RPC query to the platform, and the platform gets the data schema and display schema for the particular constructor by means of
calling constructor's `get_params` API. Schemas are sent via the platform back to the client browser where
they are processed by the Smartz client code. Based on data and display schema a user interface is automatically generated for
constructor configuring (using a [react jsonschema form](https://github.com/mozilla-services/react-jsonschema-form) component).
In this interface, the most convenient widgets are rendered for each particular type
(e.g., unix timestamp — a calendar that supports specifying date and time, an address book for blockchain addresses).
When a user sends the form with parameters to the constructor, they are first quickly validated for their consistency
with the front-end data schema and incorrectly completed fields are highlighted as required. If the data passes the validation successfully,
it is then sent to Smartz.

#### Generating and compiling a smart contract

On the back-end side, Smartz gets input data for the constructor and first validates its compliance with the data
schema. In case errors are found, error information is sent back to the client with a drilldown for each field
(using [python module for working with json schema](https://github.com/Julian/jsonschema)).
If no errors are found, input data are sent to the "construct" call.

It's assumed that
the constructor first performs an additional, more complicated validation. For instance, in case of a multisig wallet
there's an obvious condition saying that the number of owners may not be less than the signature quorum. It does not seem viable
to implement such a condition at the data schema level. Such validation may result either in a generic
error that is not specific for any particular field (as in the above example), or an error that is related to a
specific field. Both cases are included in an appropriate API response, returned to Smartz, sent to
and rendered on the client.

Then the smart contract code is generated. Generation setup is the responsibility of the constructor developers,
and they are free to select their own approach. Smartz will provide a library to simplify generation and
increase its reliability. For instance, it will contain methods for secure value substitution in a smart contract template (if
no substitution occurs, this is automatically perceived as an error, most probably a misprint), and
for verification to check whether all substitutions are properly made. Besides, Smartz will provide a set of well-known template engines.
(e.g. [Mustache](https://mustache.github.io), [Jinja](http://jinja.pocoo.org), [Thymeleaf](https://www.thymeleaf.org), etc.).

In addition to generation performed by the constructor, Smartz adds to the source code instructions for sending a fee for
contract launching to the developer and the platform.

Generated output is a source code for a smart contract written in any given language (e.g., solidity). Then the source code is
sent to the compilation infrastructure. This is a dedicated service that uses a set of containers to compile
the code for a smart contract using the right version of the right language (e.g., C++ with the Solc compiler version is used for Ethereum).
All other things equal, we have to use the highest permitted stable version of the language.
Along with that, a higher degree of isolation is achieved from other parts of the platform.

If compilation errors occur, the compilation service returns a standard compiler's output stream that is saved in
the Smartz storage. At the same time, we perform deduplication and compression to the records related to the same constructor because
the chances are that there's a significant amount of duplication across the records. Developer will be notified of the compilation
issues via a notification engine, since compilation errors are no regular situation. Any user
input errors must be managed and prevented at the constructor stage, and after that data and code
must be harmonised.

In case of successful compilation we get the binary code (e.g. EVM bytecode, web assembly etc.) and
an interface (e.g. ABI) as a result. They're sent to user browser via Smartz.

Along with that, the constructor's post-construct method is called where the input data are sent (the same data that was sent
to construct) together with the resulting smart contract interface. Based on these parameters, the constructor is able to enrich
the standard DApp that is generated by the platform with the additional data (see below). At this step, the so-called
"instance" is shaped as part of the platform, and the contract's source code, binary code, interface, and
additional information for DApp are included in its description.

#### Uploading a smart contract into blockchain

After the compilation step, the user has the option of uploading the resulting smart contract into blockchain in the Smartz interface.
The entire upload procedure is performed on the user side by means of interaction with blockchain in the browser. In case of
the Ethereum network, you may use just any browser that manages user accounts and provides the
[web3] interface (https://github.com/ethereum/wiki/wiki/JavaScript-API) (e.g. Chrome and similar browsers with
[MetaMask] (https://metamask.io) or [Mist] extension (https://github.com/ethereum/mist)). In case of EOS, you can use any
browser that manages user accounts and provides the [eosjs] interface (https://github.com/EOSIO/eosjs).

Smartz creates and tries to send a contract upload transaction to the network. At the same time, the browser asks the user to
sign the transaction and specify information about fees charged by the network, developer's fees and Smartz platform-related fees. After signing the transaction,
it is sent to the network, and Smartz expects miners to include it in the blockchain.  When the transaction is included in the blockchain,
the platform registers the required blockchain-related information in an instance of the constructor (e.g., blockchain address of the contract and ID of the network,
to which the contract was uploaded).

After successfully uploading the contract, Smartz is going to automatically verify contract code in the background on specialised block
explorers (e.g. [etherscan](https://etherscan.io)) unless user explicitly forbade this action.

The user receives the contract's blockchain address and a link to DApp, and the instance is added into the list of user instances.

### Interacting with smart contracts

On the page of the contract instance, Smartz builds DApp GUI based on:
* blockchain parameters of the contract instance
* blockchain interface of the contract instance
* additional information provided in `post_construct` call, like:
- user-friendly names and descriptions for the functions, their parameters and returned values
- refined types of parameters and returned values, e.g.:
- date & time type for unix timestamp that is actually stored as a plain number and represented as such in standard DApp interfaces
- hash-file type
- etc.
- management of sorting order and grouping of elements

First of all, interface comprises general information about the contract: instance name, address, balance, owners (where
applicable to any given blockchain).

#### Functions providing interaction with smart contracts

The core building blocks for an interface are the functions (or methods) of the contract. Platform functions are divided into three
core groups: contract poll functions (read-only) without parameters; contract poll functions with parameters; functions to
send transactions to contract. The output of poll functions is only relevant for the block that was
the last block in a basic chain at the moment of executing the function. Any interaction with the contract is performed via
an appropriate blockchain interface that is provided by the client browser (see above).

Contract poll functions without parameters are used all at once — each of them is called simultaneously along with getting and
rendering the call output. Besides, the developer attributes some of them to the instance
dashboard. Instance dashboard is displayed in the first turn, including displaying on the summary pages for the instances
of the user.

Contract poll functions with parameters may be called via appropriate control widget after the user
enters the required parameters. The output is obtained asynchronously via a blockchain poll; however, you don't have to create
a transaction and wait until it is accepted by blockchain. Therefore, users call this function free of charge.

Functions to send transactions to the contract enable creating a transaction and normally modify the state of the contract and
blockchain. After the user enters the call parameters, Smartz creates a transaction and offers the user the option to
sign it. A network token (currency) may also be sent within the transaction, e.g., Ether. This may require withdrawing the contract developer's
fee, if the developer enabled a monetisation option. After the transaction is signed by the user,
it is sent to blockchain, and the platform waits asynchronously until the transaction is accepted by blockchain. When this happens,
the user is informed in the interface of the instance.

#### Parameters of the functions, widgets

To create a rich DApp interface, you should be able to bring a description for any given input or output parameter
of the function as close as possible to the concrete area where this smart contract operates. E.g., from the point of view of blockchain,
a parameter for two various contract functions may be an integer (e.g., `unit256` in Ethereum). As such, 
these functions will firstly be using the same DApp interface, and secondly, this interface will be quite poor: it's going to be just a field where the numerical value is entered. However, in some cases,
date and time may actually be saved as a numerical value (unix timestamp), and in other cases the amount in Ethers is specified in wei.
Having this information, Smartz may in some cases render a calendar widget instead of a field for entering a numeric value and in other cases a convenient
widget for entering the amount in Ethers without specifying a huge number of wei that enables entering an amount in
USD.

Description of the input parameters for the functions may be provided by the constructor developers via the data schema with its core element
being either an array, or an object (if the parameters named are supported in the interface of specific blockchain).
To describe the schemas, the same technologies are used as those for describing user input data when designing
the smart contract. With that, the platform determines ready-to-use data types — blockchain addresses, date and time, amount of
cryptocurrency, hash-strings, file hash, etc. For each function, Smartz renders a control widget
based on the schema of the function's input parameters, using [react jsonschema form](https://github.com/mozilla-services/react-jsonschema-form) as a basis.
When the user enters parameters, they are validated for compliance with the schema, and after that they
are converted to the format of a specific blockchain. For instance, when using a web3 library, you should save numerical values in Ethereum blockchain
as instances of [BigNumber](http://mikemcl.github.io/bignumber.js/). Retrieved values
are sent to the code that shapes the transaction.

The output of the functions may also be complemented with the data schema. Here all the same techniques are applied as described above, but
in the reverse sequence. First of all, data are decoded from the blockchain format. Then, based on the schema
provided by the constructor developer, the value is rendered in the most user-friendly way.


## SDK

A software development kit (SDK) is available at https://github.com/smartzplatform/SDK. It includes API documentation,
data schemas, launch and debugging tools for constructors, and completed samples of constructor code.

API documentation contains a complete description of the platform interface that enables interaction with the constructor
for all supported languages used to develop constructors. The constructor interface is the key interface, e.g.:
[for Python](https://github.com/smartzplatform/SDK/blob/ba8230d39e94f70a30e716f4f1e48ddd4e702432/api/smartz/api/constructor_engine.py).

In previous sections we mentioned that Smartz provides some ready-to-use data types that may be utilised in data
schemas, e.g. blockchain addresses and unix timestamp. All of these are available in the SDK. For instance, there are the following
definitions available for Ethereum as json schema:
```javascript
{
"$schema": "http://json-schema.org/draft-04/schema#",
"definitions": {
"address": {
"type": "string",
"pattern": "^(?:0[Xx])?[0-9a-fA-F]{40}$"
},

"addressArray": {
"type": "array",
"items": { "$ref": "#/definitions/address" }
},

"unixTime": {
"type": "integer",
"minimum": 1,
"maximum": 2147483647
}
// ...
```

Developers value the speed of developing and debugging code. For this reason, Smartz SDK provides tools for a fast launch and 
tools for debugging on the local developer's machine without having to load constructors to Smartz.
For instance, you can check the data schema being created with the help of the console command:
```bash
./SDK/bin/run-constructor.py get_params smartz/backend/constructor_examples/SMRDistributionVault.py
```

And you can use the following console command to call the constructor and retrieve the smart contract code:
```bash
./SDK/bin/run-constructor.py construct --fields-file fields.json smartz/backend/constructor_examples/SMRDistributionVault.py
```

Moreover, you can check the automatically generated DApp interface based on the smart contract interface:
the rest of the data will be retrieved from the constructor.

The local environment on the developer's machine cannot fully comply with the Smartz environment, so the next,
final step of debugging will be to load and launch the contract in the Smartz test environment that fully
complies with the working environment; the only difference is that the loaded constructors won't be available to other users
of the platform.

Support of the development tool will be integrated as a plug-in into the most popular IDE.

A major part of constructors developed by the Smartz team will be represented by examples illustrating the use
of the platform and development best practices [see here](https://github.com/smartzplatform/SDK/tree/ba8230d39e94f70a30e716f4f1e48ddd4e702432/constructor_examples).


## API

Platform users may access Smartz not only from a browser. The following API will be available.
1. So, we make the `get_params` API call available and retrieve the constructor's parameters.
2. By means of the `construct` API call, you can call the constructor and retrieve the binary code for the contract along with other information.
Then you should create and sign a contract creation transaction with a private key that is still stored
on the client side.
3. The retrieved signed transaction must be sent to the network with the help of the `broadcast_tx` call that sets
optimal fee values depending on the transaction amount and the task urgency.

Steps 2 and 3 will be combined within a single service that is provided to the customer either as source codes assembled with the help of
[docker](https://www.docker.com), or as a docker image, whatever the customer selects. Therefore, the service does not require the customer
to trust Smartz and is entirely controllable by the customer. The service has access to
private keys that are stored either in nodes of appropriate blockchains or in [HashiCorp Vault](https://www.vaultproject.io).
The service provides asynchronous API for a single-step solution of the final task: uploading the right contract with the specified
parameters into blockchain. The customer's IT pros do not have to be advanced users of blockchain or any specific technology used to support
the infrastructure of relevant blockchains — it's enough just to be able to create and securely store the private keys and to work
with the classic REST API. Using docker enables solution deployment virtually in any customer infrastructure, including
clouds.
Use of the service is not mandatory and is left to the customer's discretion.

Similarly, you can sign a transaction for the role specified in the constructor instance and
send it via `broadcast_tx` or via the service.

A list of client-launched instances will also be accessible through API.


## Integration with Github

A typical part of the smart contract development infrastructure is a separate repository at [Github](https://github.com),
containing source code of the contract split into multiple files, list of smart contract dependencies, autotests, etc.
Normally, the smart contract constructor may not be automatically retrieved from any of the above, if only because
it contains a lot of extra information.

We expect the smart contract constructor to reside in
the "smartz" subdirectory in the smart contract repository. We also expect it to contain the developer's public Smartz API key,
a human-friendly name and description of the constructor as well as the logo, images and other auxiliary materials.
In this case, Smartz will be able to automatically upload the constructor to the platform as a pre-release version, if the developer specifies the URL of their
repository containing the contract.

Smartz is also able to monitor new releases in the contract repository using the public Github API. A pre-release may be created
on the platform for each release with the release description added as extra information.

You can also work with private repositories when you're granted read access for them.

Pre-releases on Smartz become available to the general public via moderation. They may be sent to moderation either automatically
or manually, on the developer's request. It may be inadvisable to always select automated sending
because moderation implies a number of restrictions for query frequency, and the code audit is paid.


## Integration with Travis CI

Another typical part of the smart contract development infrastructure is continuous integration services
(CI), like [Travis CI](https://travis-ci.org). Github + Travis CI combination is also quite typical. Autotests are
automatically executed for new changes made to the source code, and the results of the automated test of
the changes become clear. If Travis CI or other continuous integration systems are connected, then only the changes that
have successfully passed autotests may be uploaded to Smartz as pre-releases.

You will also be able to configure pre-release based on the outcomes of successful project assembly within the continuous integration
system. The process is very similar to the one described in the previous section: you will need the constructor, name, description and other
materials as well as the developer's private Smartz API key. Whatever is required will be packaged into a single package and sent to
Smartz. Smartz provides a script that enables executing the specified actions by starting a single command, including as a docker image.


## Contract systems

Just as the programs written with the use of OOP consist of multiple interoperable classes, so the advanced
blockchain applications consist of multiple interoperable contracts. We call it a contract system. A typical
example is an ICO contract system where there is one or more crowdsale contracts containing the price calculation
logic etc.; token contract, and storage contract to store raised funds which enables participants to get
their money back if the project does not reach its soft cap.

Until this moment we have viewed the smart contract source code as a kind of atomic and a simultaneously self-contained unit,
but contract systems require some additional actions from the platform while compiling and uploading to blockchain.

In the event of a system of contracts, the constructor is represented by an archive (tag.gz, zip, jar etc.) containing
whatever is required to provide the platform with the source code of smart contracts adapted according to a specific query
of the user. In most cases it will be the source code templates for the contracts together with the constructor logic packaged in separate files, and
a migration file (see below).

The user query is processed within the platform in the following way.
The constructor service unpacks the archive in a siloed docker environment.
Depending upon the language used for writing the constructor, the archive may contain an entry point, e.g. a
constructor.py file executed by the platform. It receives input data just as it happens with a
single contract. We have already discussed details of the constructor service earlier.

It is presumed that the constructor performs all required actions to prepare the code: substitutes the retrieved values
into configuration files, runs template engines etc. As a result, the constructor must send a set of the source code
artifacts to the platform. Each artifact has a name and contains a link to the file in the directory that needs to be compiled
in order to get a binary artifact representation.

Then the constructor service sends the directory and artifact-related information to the compilation service where a binary representation is available for
each artifact. The compilation details were discussed earlier.

Then you should send the retrieved artifacts to user browser and launch contracts in blockchain.

### Uploading smart contract system to blockchain

The most delicate thing about supporting contract systems is uploading the system to blockchain. This
is conditioned firstly, by the requirement related to contract binding, and secondly, by restrictions imposed on
transactions by blockchains (e.g., gas limit in Ethereum block). It's not enough just to upload a sequence of contracts to blockchain,
you should set proper bindings for them. Binding may be set both during the upload of a specific contract using the constructor
function arguments, and after that, by sending transactions (e.g. calls of setters).
This process is normally described using so called "migrations," and to describe it properly a certain expressivity of the common programming language
is advisable (e.g. in [truffle framework](http://truffleframework.com) migrations are written in javascript).

In Smartz, migrations will be written in a special declarative language that is created specifically for this task (DSL).
This is done mostly in order to provide security for users and platform. Although the constructors are going through
an audit, we still consider it unsafe and premature to provide migrations with direct access to blockchain APIs (e.g. web3)
and blockchain user accounts, even if every action requires explicit user confirmation.
In the future, these restrictions may be eliminated or at least softened after proper in-depth development. Migrations written in javascript may
be accepted for specific contracts as required, but only after they pass thorough audit.

The language of migrations will contain the following primitives:

* Uploading a previously retrieved artifact into blockchain.
* Calling a specified function of one of the uploaded artifacts.
* Constants, data retrieved from users, and addresses of already uploaded artifacts may  
become call arguments (incl. contract constructor functions).
* Opportunity to specify a user-friendly description of the current step.

Artifacts, user input data and the migration handler will be uploaded to the client's browser.
The migration language will be interpreted by the platform code and perform the required action using blockchain APIs and
the data listed above.

For instance, we get the following migration for the above-mentioned ICO contract system:

```c++
token = deploy('Token', fields.owners);
funds = deploy('FundsRegistry', fields.owners);

sale = deploy('Crowdsale', fields.owners, token.address, funds.address);

token.transact('setController', sale.address);
funds.transact('setController', sale.address);
```

### Crosschain contract systems

Crosschain contract systems are a set of contracts that reside in various blockchains, but
are interoperable. Interoperability may be direct (e.g. using block relay or
para-chains (for instance, see [Polkadot](https://medium.com/polkadot-network/polkadot-the-parachain-3808040a769a)),
or orchestrated using DApp (for instance see [POA Bridge UI](https://bridge.poa.net)). A typical example
of a crosschain contract system may be an atomic swap that contains two contracts, one per each
blockchain.

To support crosschain contract systems, Smartz modifies the above approach in the following way: First,
a constructor for each artifact notifies to which blockchain it belongs. The compilation service uses this knowledge to
call the appropriate compilation infrastructure. An example of such an output from the constructor in case of an atomic swap is
given below:

```json
{
"EthereumSide": {
"file": "ethereum/contracts/AtomicSwap.sol",
"blockchain": "ethereum"
},

"EOSSide": {
"file": "EOS/src/AtomicSwap.cpp",
"blockchain": "EOS"
}
}
```

Second, while uploading a smart contract system to blockchains, the contract upload directive for the migrations language
automatically gets the blockchain from the artifact description to which it should upload the contract. With that, the parameters
of function call like contract addresses are still available for any blockchain for executing the contract connections
(if setting a direct connection between contracts is required, see above). Therefore, the migration script in this case
is fairly simple:

```c++
deploy('EthereumSide');
deploy('EOSSide');
```

Again, you will be able to establish parameters for the blockchain to which the contract is uploaded, by creating a blockchain specification. For instance,
the Ethereum code base supports blockchains with a different network_id parameter (network_id = 1 — main network, network_id = 4 —
test rinkeby network), and there are Ethereum forks that use this engine, e.g., network_id = 99 — Core,
the main public blockchain [POA Network](https://poa.network). If a bridge between two Ethereum-like
blockchains is loaded, we get the following migration:

```c++
EthereumMainnet = define_network('Ethereum', network_id=1);
POACore = define_network('Ethereum', network_id=99);

deploy('ForeignBridge', network=EthereumMainnet);
deploy('HomeBridge', network=POACore);
```


# Contract management

The basic mechanics of contract management are described in the "Interaction with Smart Contracts" section. Here we are going to dwell on
the additional benefits that Smartz provides.

## Custom DApps

In a simple case, the user interface of a constructor instance is generated automatically based on the programming
interface and additional prompts from the developer; this process is described above. This may be insufficient to provide a rich
and convenient DApp. Smartz provides developers with the opportunity to upload and provide a fully-fledged
DApp UI based on javascript or TypeScript. This approach gives maximum freedom
to developers, though it requires special attention at the audit stage. We do not consider it advisable
to be complacent with half-complete measures, e.g. designing an intermediate abstraction that is a limited set of
overridden graphic primitives and operations because this will require more effort and will most probably fail to solve the initial
task.

Support will be added for DApp user interfaces implemented as SPA (Single Page Application), including
those that use react, Angular, Vue.js frameworks.

Each constructor will only be able to have a single DApp because there's no need
to use any code generation mechanisms at this step. Instead, when launching a custom DApp the platform will send
the following parameters to it:

* user input data for constructor (if applicable, see Custom DApp Marketplace below)
* contract "coordinates": blockchain specifications (see "Crosschain Contract Systems"), contract address
* contract interface(s)

Based on this data, the custom DApp will be able to properly customise the user interface.

Custom DApp must be presented with the draft source codes and the necessary resources. An audit will be performed for it before
publication. To ensure security, a whitelist of dependencies will be added that may be used for the project.
To ensure reliability, the required version of dependencies will be cached on Smartz servers and
used if the main code source (e.g., npm repository or individual packages) is unavailable or
compromised.

Spoofing of packages in the npm repository has already taken place as well as deployment of
[malicious packages](https://www.twilio.com/blog/2017/08/find-projects-infected-by-malicious-npm-packages.html).
Many accounts that are used to download npm packages are [poorly protected](https://www.bleepingcomputer.com/news/security/52-percent-of-all-javascript-npm-packages-could-have-been-hacked-via-weak-credentials/).
Besides the whitelists mentioned above, an integrity control engine will be used in order to protect DApp integrity and eliminate the requirement
to trust the Smartz platform.

1. The developer must use package-lock.json (in addition to integrity control, and this will also ensure a stable-functioning, repeatable application).
2. At the time of DApp interface assembly, integrity control is applied to all dependencies via npm engine.
`Standard Subresource Integrity`. Then the hash is computed for an assembled application.
3. Integrity control code is executed at the time of launching on the user's side. This is an open, non-minified compact
code, and its integrity may be checked by means of the hashing process. Integrity control code compares hash applications
and published it together with its [identicon](https://en.wikipedia.org/wiki/Identicon).

To prevent custom DApps from impacting each other or the platform, they will be placed into different Smartz subdomains,
that are unique for each constructor instance.

Static resources (e.g. images and CSS) should be placed into a `static` subdirectory; they will be automatically
placed onto Smartz servers and made available in the DApp interface with an appropriate filepath that starts with `/static/`.


## Local DApps

In order to completely eliminate user dependency from Smartz platform, the interface of any user-launched DApp or any DApp accessible
to the user, may be later downloaded from the platform and saved locally
in a fully operational state (there's no point in saving the other DApp component — contracts because they're executed in relevant
blockchains).

The local copy is an archive that contains javascript or TypeScript logic and resources. It may run in any
browser or application that provides interfaces for the relevant blockchains (web3.js, eosjs etc.).

The platform will also be able to issue a so `package-lock` for a DApp interface and its source code,
so that the user might assemble the interface themselves, which eliminates the requirement to trust Smartz
and enables a complete DApp audit for identifying bookmarks or unwanted logic before launching.

Local DApps will be available both to the interfaces that are automatically generated by the platform, and to custom DApps.

Local DApps implementation on the platform side has maximum resemblance with the DApps running in web mode. In both modes, DApp
consists of resources, a static part (that does not modify from instance to instance): that may be a part of the platform code
(for automatically generated interfaces), or DApp developer code (for custom DApps); and dynamic part (it
comprises user input data, contract "coordinates, contract interfaces).

In one instance, the static part may be published in the web, in the other instance it is archived. As concerns web DApp,
the dynamic part is included in a webpage of constructor instance; for the local DApp, it is saved as json in a javascript file and
connected to the DApp.

Local DApps are provided with integrity control mechanisms described in the previous section.


## Control panels

Because custom DApps are not closely connected with the specific constructor instance, but adapted dynamically
based on the information about the instance (see Custom DApps), and because interfaces and smart contract code may be open-source, 
the developer therefore has an opportunity to develop DApp interfaces for the contracts created by other developers.

This dedicated platform-based constructor type is called a dashboard. It functions the following way.

The platform calls gt_params and renders an interface for user input data. User input is validated.

No smart contracts are compiled at this step, instead a `post_construct` method is invoked. The method must provide
coordinates for the contract that will be managed by this dashboard instance.

The further mechanics are similar to those previously described. The platform assembles a DApp interface, and in the case of a web interface, the platform
adds the required data stored in Smartz (contract coordinates etc). In the case of a local DApp, the assembled
application and user input data will be output as an archive.


## Custom DApp marketplace

Dashboards will also receive a dedicated marketplace that is similar to the marketplace for constructors. Developers may create
DApp interfaces for the contracts created by other developers.

There's a number of examples of these dashboards:

* A dashboard for managing specific popular tokens. It is basically an opportunity to create a "better" wallet
that is specific for any given token and based on its mechanics that otherwise can't be obtained through the ERC20 interface. Such a
dashboard will certainly contain information about token contract coordinates.
* Dashboard for ERC20 tokens. It is more convenient and information-rich than a traditional Mist interface.


## Contract system management

The concept of contract system management is not significantly different from managing an individual contract as described above.
As concerns automatically generated interfaces, they are generated for each specific contract in the system. Besides,
the platform provides an opportunity to specify the sequence of interfaces constituting the systems and the structure of a general dashboard
for the system. In case of a custom DApp, the application can access all the required data (user data, input data, coordinates
of the contracts etc.), and the author of the DApp may freely arrange the interface in the system.

Transactions may be sent to an appropriate contract in the appropriate blockchain by means of a relevant client library, using
blockchain parameters, address and contract interface.


# Platform

In addition to providing the previously described functions of design and DApp management, the platform is a rich web portal.

## Authentication

A number of various user authentication methods will be supported.

**Signature-based private key authentication**. This is a version of a challenge-response protocol. The platform generates
a unique message that says that the message was created for the purposes of authentication on Smartz and contains
user address and current time. The platform interface gives the option of signing the message
with a private key within a specified period of time and sends it for verification to Smartz. The signature is verified using a user's public key.
For the signature, for instance, you can use 
[message signing in MetaMask](https://medium.com/metamask/the-new-secure-way-to-sign-data-in-your-browser-6af9dd2a1527).

**OAuth2-based authentication**. Social authentication providers will be used, like Facebook,
Github, Google+.

**Conventional password-based authentication**. Hashed passwords are stored in the database with an additional
random parameter (so-called "salt") in order to prevent brute-force attacks.

However, if a single user performs authentication using various methods, this will not create multiple
accounts on the platform. Instead, a function is added that enables linking new authentication methods to the account.

Administrators will have access to the "Login As" feature that enables getting read access on behalf of any other account
to troubleshoot user issues.

Optional support of two-factor authentication is provided via Google Authenticator.


## Notification engine

An internal `notify(user, subject, message)` engine will be implemented.

The user will be able to setup notifications display (in the platform interface, by email, in Telegram) in their profile. This engine will be
used by various Smartz subsystems.


## Key entities

### Groups

Rights management within the platform is performed by means of the following groups:

* administrators
* developers
* partners (integrators)
* developer moderators
* contract moderators
* partner moderators

### Constructor

Constructors are created by developers. Key functions — creation, state modification, viewing.

Constructor states:
* draft
* under moderation
* published
* archived
* blocked

Users can see the constructor only when its state is "published". Author can see it regardless of the state.

Author may perform the following state modifications:

* draft -> moderated, archived
* published -> archived

Constructor fields include so-called "tags" — single-word, phrases (less commonly), constructor specifications
provided by the author that users may use to perform a highly relevant constructor search, e.g.: multisig,
token. Tag input interface is a typical tag input string containing prompts. Prompts will be
implemented by means of
[AnalyzingInfixSuggester](https://lucene.apache.org/core/7_2_1/suggest/org/apache/lucene/search/suggest/analyzing/AnalyzingInfixSuggester.html).
Sorting — based on usage (input into constructors) frequency.

### Constructor instance

Stores information about the constructor launched, the users that launched it, constructor details (parameters,
obtained interfaces etc.), contract coordinates.

### Constructor launch statistics

Registers any launch of a constructor in blockchain. Contains constructor instances and information about
the rewards paid to platform and developer.


## Moderation

There are three moderated entities: developers, constructors, partners. They all share a single behavior.

* There exist some applications for moderation (e.g., knowing that a user wants to become a developer, or knowing that
a new constructor has been sent for publication).

* There's an interface containing the list of applications where general information is displayed for each application. In case of large amounts of information, a link to drilldown data is provided that uploads
additional information directly to the page
* The ability to accept or decline the application using available buttons ensures synchronisation between moderators.
In this case, the list displays only the declined applications. Administrators get the opportunity to cancel acceptance of
applications for the specified moderator.

* List of applications is sorted based on the criteria specific for each entity.

* Moderators may approve or reject applications (and specify the reason for their choice). In case the application is rejected, a notification
is sent to the appropriate user.

There's also individual working logic:

* Creating an application

* For developer moderation — if a user specified in their profile that they want to become a developer
* For partner moderation — if a user specified in their profile that they want to be a partner
* For constructor creation — if an author selected "Publish Constructor"

* Sort applications in the list 

* For developer moderation — older first
* For partner moderation — older first
* For constructor creation — by business function rating

* Application approval

* For developer moderation — flagged, notification sent
* For partner moderation — flagged, notification sent
* For construction creation — state modified, notification sent


## Marketplace

### List of constructors

Within the platform, a single logic exists that handles the request for the list of constructors. It performs:
* filtering by developer
* filtering by tags
* filtering by status and audit pattern
* page-by-page iteration

**Common list of constructors**. Intended for users. Displays only constructors with the
"Published" status provided only by non-blocked developers. Filters: tags, word occurrences (full-text search by
name and description). Page-by-page iteration.

**User: developer information page**. Developer page as seen by users. List
of constructors filtered by a specific developer, filter by tag, page-by-page iteration. The filter only displays
constructors that have passed moderation.

**Developer: developer information page**. Developer page as seen by developer. List
of constructors filtered by a specific developer, filter by tag, page-by-page iteration, controls
for constructors.

**User: list of constructors launched by this user**. List of launched constructors sorted by date, with
page-by-page iteration and links.

### Statistics

**User statistics**. Statistics of launches performed by a specific user.
Summary statistics for constructors (constructor => (launches, money)).
Summary statistics by month (month => (constructor => (launches, money))).
Each number of launches is expandable into the list of launches.

**Developer statistics**. Statistics of launches for constructors created by a specific developer.
Summary statistics for constructors (constructor => (launches, money)).
Summary statistics by month (month => (constructor => (launches, money))).

### Top constructors

Automatic ratings for constructors will be implemented.

1. top paid (sorted by price)

2. top free (sorted by number of launches)

3. top new for 1 month (5 paid, 5 free, sorted as specified above) 

4. in each of the 8 popular tags: personal top list (5 paid, 5 free, sorted as specified above)

And the editorial list which is compiled manually. All top constructors — max. 10 contracts per constructor.

### Rating system

Constructors and developers are automatically assigned a rating based on user scores.
The rating system uses a ten-to-one scale. Ratings are displayed as 5 stars where some stars may be greyed out.
If a score is an odd number, only half of a star may be greyed out. Only users that launched the specified constructor or a constructor by the specified developer
may leave their scores.

Rating calculation considers not only an average score, but also the frequency and relevance of the scores. The rating may not
be calculated without actual scores available.
The rating is calculated based on the following formula:

```
rating = weighted_avg_rating / 2 + 5 * (1 − exp(−quantity / quantity_baseline))
```

where `quantity` is the number of scores,

`quantity_baseline` is a basic indicator of the number of scores calculated as an average number of scores for the entity,

`weighted_avg_rating` is a relevance-weighted average score calculated in the following way:

```
weighted_avg_rating = sum(mark_i * (max(actuality_baseline - age_i, 0))) / sum(max(actuality_baseline - age_i, 0))
```

where `sum` performs a summing of all scores,

`mark_i` is the 1st score from 0 to 10,

`age_i` is the age of the 1st score (in days),

`actuality_baseline` is the window of relevance equal to two years (in days).


## Search


## Recommender system


## High availability and fault tolerance




# Blockchain support

## Ethereum


## EOS


# Smart contract integration


## Payments


## Event API

### Telegram notifications and bots


## Calls scheduler


## Widgets API


## Decentralised audit


## Decentralised oracles


## Overuse of code in blockchains


## Integration of third-party agents and notary officers


## Handling external datasets

### Marketplace for external datasets


## Decentralised reputation system




# Smart contracts




# Mobile apps




# Smartz decentralisation



TODO

constructor patterns — config in descendant class, monolith, generation

