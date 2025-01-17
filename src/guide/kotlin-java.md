# Kotlin/Java Guide

## 1. Iroha 2 Client Setup

In this part we shall cover the main things to look out for if you want to
use Iroha 2 in your Kotlin application. Instead of providing the complete
basics, we shall assume knowledge of the most widely used concepts, explain
the unusual, and provide some instructions for creating your own Iroha
2-compatible client.

We assume that you know how to create a new package and have basic
understanding of the fundamental Kotlin code. Specifically, we shall assume
that you know how to build and deploy your program on the target platforms.
The Iroha 2 JVM-compatible SDKs are as much a work-in-progress as the rest
of this guide, and significantly more so than the Rust library.

Without further ado, here's a part of an example `build.gradle.kts` file,
specifically, the `repositories` and `dependencies` sections:

```kotlin
repositories {
    // Use Maven Central for resolving dependencies
    mavenCentral()
    // Use Jitpack
    maven { url = uri("https://jitpack.io") }
}

dependencies {
    // Align versions of all Kotlin components
    implementation(platform("org.jetbrains.kotlin:kotlin-bom"))
    // Use the Kotlin JDK 8 standard library
    implementation("org.jetbrains.kotlin:kotlin-stdlib-jdk8")
    // Load the dependency used by the application
    implementation("com.google.guava:guava:31.0.1-jre")
    // Use the Kotlin test library
    testImplementation("org.jetbrains.kotlin:kotlin-test")
    // Use the Kotlin JUnit integration
    testImplementation("org.jetbrains.kotlin:kotlin-test-junit")
    // Load Iroha-related dependencies
    implementation("com.github.hyperledger.iroha-java:client:SNAPSHOT")
    implementation("com.github.hyperledger.iroha-java:block:SNAPSHOT")
    implementation("com.github.hyperledger.iroha-java:model:SNAPSHOT")
    implementation("com.github.hyperledger.iroha-java:test-tools:SNAPSHOT")
}
```

You **should** replace the SNAPSHOT in the above configuration with the
latest `iroha-java` snapshot.

Snapshot versions match the Git commits. To get the latest snapshot, simply
visit the
[`iroha-java`](https://github.com/hyperledger/iroha-java/tree/iroha2-dev)
repository on the `iroha-2-dev` branch and copy the short hash of the last
commit on the main page.

![](/img/iroha_java_hash.png)

You can also check the
[commit history](https://github.com/hyperledger/iroha-java/commits/iroha2-dev)
and copy the commit hash of a previous commit.

![](/img/iroha_java_commits.png)

This will give you the latest development release of Iroha 2.

## 2. Configuring Iroha 2

<!-- Check: a reference about future releases or work in progress -->

At present, the Kotlin SDK doesn't have any classes to interact with the
configuration. Instead, you are provided with a ready-made `Iroha2Client`
that reads the configuration from the environment variables and/or the
resident `config.json` in the working directory.

If you are so inclined, you can have a look at the `testcontainers` module,
and see how the `Iroha2Config` is implemented.

<<<@/snippets/IrohaConfig.kotlin

## 3. Registering a Domain

Registering a domain is one of the easier operations. The usual boilerplate
code, that often only serves to instantiate a client from an on-disk
configuration file, is unnecessary. Instead, you have to deal with a few
imports:

```kotlin
import org.junit.jupiter.api.extension.ExtendWith
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions
import jp.co.soramitsu.iroha2.engine.IrohaRunnerExtension
import jp.co.soramitsu.iroha2.Iroha2Client
import jp.co.soramitsu.iroha2.engine.WithIroha
import kotlinx.coroutines.runBlocking
import kotlin.test.assertEquals
import java.util.concurrent.TimeUnit
import jp.co.soramitsu.iroha2.generated.datamodel.account.Id as AccountId
```

We shall write this example in the form of a test class, hence the presence
of test-related packages. Note the presence of `coroutines.runBlocking`.
Iroha makes extensive use of asynchronous programming (in Rust
terminology), hence blocking is not necessarily the only mode of
interaction with the Iroha 2 code.

We have started by creating a mutable lazy-initialised client. This client
is passed an instance of a domain registration box, which we get as a
result of evaluating `registerDomain(domainName)`. Then the client is sent
a transaction which consists of that one instruction. And that's it.

<<<@/snippets/InstructionsTest.kt#java_register_domain{kotlin}

Well, almost. You may have noticed that we had to do this on behalf of
`aliceAccountId`. This is because any transaction on the Iroha 2 blockchain
has to be done by an account. This is a special account that must already
exist on the blockchain. You can ensure that point by reading through
`genesis.json` and seeing that **_alice_** indeed has an account, with a
public key. Furthermore, the account's public key must be included in the
configuration. If either of these two is missing, you will not be able to
register an account, and will be greeted by an exception of an appropriate
type.

## 4. Registering an Account

Registering an account is more involved than the aforementioned functions.
Previously, we only had to worry about submitting a single instruction,
with a single string-based registration box (in Rust terminology, the
heap-allocated reference types are all called boxes).

When registering an account, there are a few more variables. The account
can only be registered to an existing domain. Also, an account typically
has to have a key pair. So if e.g. _alice@wonderland_ was registering an
account for _white_rabbit@looking_glass_, she should provide his public
key.

It is tempting to generate both the private and public keys at this time,
but it isn't the brightest idea. Remember that _the white_rabbit_ trusts
_you, alice@wonderland,_ to create an account for them in the domain
_looking_glass_, **but doesn't want you to have access to that account
after creation**.

If you gave _white_rabbit_ a key that you generated yourself, how would
they know if you don't have a copy of their private key? Instead, the best
way is to **ask** _white_rabbit_ to generate a new key-pair, and give you
the public half of it.

Similarly to the previous example, we provide the instructions in the form
of a test:

<<<@/snippets/InstructionsTest.kt#java_register_account{kotlin}

As you can see, for _illustrative purposes_, we have generated a new
key-pair. We converted that key-pair into an Iroha-compatible format using
`toIrohaPublicKey`, and added the public key to the instruction to register
an account.

Again, it's important to note that we are using _alice@wonderland_ as a
proxy to interact with the blockchain, hence her credentials also appear in
the transaction.

## 5. Registering and minting assets

Iroha has been built with few
[underlying assumptions](./blockchain/assets.md) about what the assets need
to be in terms of their value type and characteristics (fungible or
non-fungible, mintable or non-mintable).

::: info

<!-- Check: a reference about future releases or work in progress -->

The non-mintable assets are a relatively recent addition to Iroha 2, thus
registering and minting such assets is not presently possible through the
Kotlin SDK.

:::

<<<@/snippets/InstructionsTest.kt#java_register_asset{kotlin}

<<<@/snippets/InstructionsTest.kt#java_mint_asset{kotlin}

Note that our original intention was to register an asset named
_time#looking_glass_ that was non-mintable. Due to a technical limitation
we cannot prevent that asset from being minted. However, we can ensure that
the late bunny is always late: _alice@wonderland_ can mint time but only to
her account initially.

If she tried to mint an asset that was registered using a different client,
which was non-mintable, this attempt would have been rejected, _and Alice
alongside her long-eared, perpetually stressed friend would have no way of
making more time_.

## 6. Visualizing outputs

Finally, we should talk about visualising data. The Rust API is currently
the most complete in terms of available queries and instructions. After
all, this is the language in which Iroha 2 was built. Kotlin, by contrast,
supports only some features.

There are two possible event filters: `PipelineEventFilter` and
`DataEventFilter`, we shall focus on the former. This filter sieves events
pertaining to the process of submitting a transaction, executing a
transaction and committing it to a block.

```kotlin
import jp.co.soramitsu.iroha2.generated.datamodel.events.EventFilter.Pipeline
import jp.co.soramitsu.iroha2.generated.datamodel.events.pipeline.EventFilter
import jp.co.soramitsu.iroha2.generated.datamodel.events.pipeline.EntityType.Transaction
import jp.co.soramitsu.iroha2.generated.crypto.hash.Hash

val hash: ByteArray
val eventFilter = Pipeline(EventFilter(Transaction(), Hash(hash)))
```

What this short code snippet does is the following: It creates an event
pipeline filter that checks if a transaction with the specified hash was
submitted/rejected. This can then be used to see if the transaction we
submitted was processed correctly and provide feedback to the end-user.

## 7. Samples in pure Java

<<<@/snippets/JavaTest.java
