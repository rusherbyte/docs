# Signed pipelines

## What are signed pipelines?

Signed Pipelines is a security feature on the Buildkite Agent that allows you to cryptographically sign steps that are uploaded to Buildkite, and then verify that signature on the Buildkite Agent before running the step. This signature helps agents establish the provenance of the jobs that they are instructed to run - asserting that the steps were uploaded by a trusted agent, and that they haven't been modified since they were uploaded. If an agent detects a signature mismatch, by default, it'll refuse to run the job.

These signatures mean that if a threat actor were able to modify a job in flight, the Agent would refuse to run it due to mismatched signatures.

<details>
  <summary>I think I've seen this before...</summary>
  This work is inspired by the older <a href="https://github.com/buildkite/buildkite-signed-pipeline"><code>buildkite-signed-pipeline</code></a> tool, which was a tool you could add to your agent instances. It had a similar idea - signing steps before they're uploaded to Buildkite, then verifying them when they're run. However, it had a number of limitations, including:
  <ul>
    <li>It had to be installed on every agent instance, leading to more configuration</li>
    <li>It only supported symmetric HS256 signatures, meaning that every verifier could also sign uploads</li>
    <li>It couldn't sign matrix steps</li>
  </ul>

  This newer version of pipeline signing is built right into the agent, and addresses all of these limitations. Being built into the agent, it's easier to configure and use.

  Many thanks to Seek.com.au, who we collaborated with on the older version of the tool, and whose prior art has been instrumental in the development of this newer version.
</details>

### What we actually sign

As mentioned above, Signed Pipelines establishes that important aspects of steps haven't been changed since pipeline upload. In more detail, we sign the following aspects of steps:
- The command
- Environment variables _as defined in the pipeline YAML_ - environment variables set by the agent, by hooks, or by the user's shell are not signed and can potentially override the environment that a step's command is started with.
- Any plugins used by the step, and their configurations
- The matrix configuration [^1]
- The step's key, if present [WIP]

[^1]: The matrix configuration is signed as a whole, rather than each individual matrix job. This means that the signature will be the same for each job in the matrix. When signatures are verified for matrix jobs, we double-check that the job we've received is a valid construction of the matrix, and that the signature matches the matrix configuration.

Additionally, we also sign the following aspects of the build for each job:
- The pipeline's UUID [WIP]
- The organization UUID [WIP]
- The build number [WIP]
- The repo the build is running for [WIP]
- The commit the build is running for [WIP]


## Enabling signed pipelines on your agents

### 1. Generating a key pair
Behind the scenes, Signed Pipelines uses [JSON Web Signing (JWS)](https://datatracker.ietf.org/doc/html/rfc7797) to generate its signatures. This means that you'll need to generate a [JSON Web Key Set (JWKS)](https://datatracker.ietf.org/doc/html/rfc7517) to sign and verify your pipelines with, then configure your agents to use those keys.

Luckily, the Buildkite Agent has you covered! There's a JWKS generation tool built into the agent, which you can use to generate a key pair for you. To run it, you'll need to [install the agent on your machine](/docs/agent/v3/installation), and then run:
```bash
buildkite-agent tool keygen --alg <algorithm> --key-id <key-id>
```

replacing `<algorithm>` and `<key-id>` with the algorithm you want to use, and the key ID you want to use. For example, to generate an EdDSA key pair with a key ID of `my-key-id`, you'd run:
```bash
buildkite-agent tool keygen --alg EdDSA --key-id my-key-id
```

The agent will then generate two JSON Web Key Sets for you, one private and one public, and output them to files in the current directory. You can then use these keys to sign and verify your pipelines, respectively.

If no key id is provided, the agent will generate a random one for you.

Note that the value of `--alg` must be a valid [JSON Web Signing Algorithm](https://datatracker.ietf.org/doc/html/rfc7518#section-3). The agent does not support the `none` algorithm, or any signature algorithms in the `RS` series (any signing algorithms that use RSASSA-PKCS1 v1.5 signatures, `RS256` for example).

<details>
  <summary>Why doesn't the agent support RSASSA-PKCS1 v1.5 signatures?</summary>
  In short, it's because RSASSA-PKCS1 v1.5 signatures are generally regarded to be less secure than the newer RSA-PSS signatures. While RSASSA-PKCS1 v1.5 signatures are still largely known to be relatively secure, we want to encourage our users to use the most secure algorithms possible, so when using RSA keys, we only support RSA-PSS signatures. We also recommend looking into ECDSA and EdDSA signatures, which are generally regarded to be more secure than RSA signatures.
</details>

#### What algorithm should I use?

Signed Pipelines supports a number of different signing algorithms, which can largely be broken down into two categories: symmetric (where signing and verification use the same keys), and asymmetric signing algorithms (where signing and verification use different keys).

We generally recommend using an asymmetric signing algorithm, as it creates a permissions boundary between the agents that upload and sign pipelines, and agents that run other jobs and have the ability to verify signatures. This means that were an attacker able to compromise instances with verification keys, they wouldn't be able to modify the steps that are run on your agents.

When using asymmetric signing algorithms, we recommend having multiple disjoint pools of agents, each using a different [queue](docs/agent/v3/queues). One pool should be the "uploaders", and have access to the private keys, and another pool should be the "runners", and have access to the public keys. This creates a security boundary between the agents that upload and sign pipelines, and the agents that run jobs and verify signatures.

With regards to specific algorithm choice, basically any of the asymmetric signing algorithms we support are fine and will be secure. If you're not sure which one to use, `EdDSA` is proven to be secure, has a modern design, wasn't designed by a Nation State Actor, and produces nice short signatures.

For more information on what algorithms the agent supports, run `buildkite-agent tool keygen --help`.

### 2. Making your agents sign steps they upload 

Now that your keys are generated, we need to configure your agents to use them. On agents that will be uploading pipelines, add the following to your agent's config file:
```ini
job-signing-jwks-path=<path to private keyset>
signing-key-id=<the key id you generated earlier>
job-verification-jwks-path=<path to public keyset>
```

This will ensure that whenever those agents upload steps to Buildkite, they'll generate signatures for those steps using the private key we generated earlier. It also ensures that those agents will verify the signatures of any steps they run, using the public key.

On instances that will verify jobs, add:
```ini
job-verification-jwks-path=<path to verification keys>
```

### 3. Signing the pipeline's configured steps

BENNO: MAKE THESE LINKS GO TO PLACES BEFORE YOU MERGE THIS

Builds in Buildkite pipelines are initialized using steps that are configured on the pipeline itself. In the course of running a job, the Buildkite Agent can [upload further steps to the pipeline](link to some info about this elsewhere in the docs). We will refer to the former as "configured steps" and the latter as "uploaded steps". It's common for the configured steps to be a single step that uploads the remainder of pipeline from [a YAML file in the repository](link to info about ./buildkite.yaml).

> Some older Buildkite pipelines don't use YAML for their configured steps - these can't be signed, and will have to be migrated to use the new format to be used with signed pipelines - see [Migrating to YAML steps](/docs/tutorials/pipeline_upgrade)

You may have noticed that when we configured the signing instances, we gave them the capability to both sign and verify jobs. For maximum security, the configured steps - usually a single step that uploads (and signs!) the remainder of the pipeline - should themselves be signed.

The solution here is to add static signatures to the steps in your pipeline's configured steps. To do this, copy the YAML from the editor on the pipeline's page in Buildkite, and paste it into a file on your machine. Then, run:
```bash
buildkite-agent pipeline upload --jwks-file-path <path to private keys> --signing-key-id <key id> --dry-run --format yaml <path to pipeline file>
```

For example, if you had a file called `pipeline.yml` in the current directory, and you wanted to sign it with the key we generated earlier, you'd run:
```bash
buildkite-agent pipeline upload --jwks-file-path /path/to/private/key.json --signing-key-id my-key-id --dry-run --format yaml configured-steps.yml
```

This will output the pipeline YAML, with the signatures added to the steps. You can then copy this YAML back into the editor on the pipeline's page in Buildkite, and save it.

## Rotating signing keys

Because signed pipelines use JWKS as their key format, rotating keys is easy. To rotate your keys, simply generate a new key pair, and add the new keys to your existing key sets, being careful not to mix public and private keys. Once the updated key sets are in place, you can update the `signing-key-id` on your signing agents to use the new key ID. Verifying agents will automatically use the public key with the matching key ID, if it's present.
