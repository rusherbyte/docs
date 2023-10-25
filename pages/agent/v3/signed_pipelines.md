# Signed pipelines

> 🚧 Experimental feature
> Signed pipelines are experimental, and these docs (and the commands and arguments referenced in them) are subject to change. This is also an opt-in feature, so if you'd like to try using signed pipelines, reach out to [support](https://buildkite.com/support).

Signed pipelines are a security feature where pipelines are cryptographically signed when uploaded to Buildkite. Agents then verify the signature before running the job. If an agent detects a signature mismatch, it'll refuse to run the job.

The signature guarantees the origin of jobs by asserting:

- The jobs were uploaded from a trusted source.
- The jobs haven't been modified after upload.

These signatures mean that if a threat actor could modify a job in flight, the agent would refuse to run it due to mismatched signatures.

<details>
  <summary>🤔 I think I've seen this before...</summary>
  <p>This work is inspired by the <a href="https://github.com/buildkite/buildkite-signed-pipeline"><code>buildkite-signed-pipeline</code></a> tool, which you could add to your agent instances. It had a similar idea—signing steps before they're uploaded to Buildkite, then verifying them when they're run. However, it had some limitations, including:</p>
  <ul>
    <li>It had to be installed on every agent instance, leading to more configuration.</li>
    <li>It only supported symmetric HS256 signatures, meaning that every verifier could also sign uploads.</li>
    <li>It couldn't sign <a href="/docs/pipelines/build-matrix">matrix steps</a>.</li>
  </ul>
  <p>This newer version of pipeline signing is built right into the agent and addresses all of these limitations. Being built into the agent, it's also easier to configure and use.</p>
  <p>Many thanks to <a href="seek.com.au">SEEK</a>, who we collaborated with on the older version of the tool, and whose prior art has been instrumental in the development of this newer version.</p>
</details>

## Pipeline signatures

Pipeline signatures establish that important aspects of steps haven't been changed since they were uploaded.

The following fields are included in the signature for each step:

- **Commands.**
- **Environment variables defined in the pipeline YAML.** Environment variables set by the agent, hooks, or the user's shell are not signed and can override the environment a step's command is started with.
- **Plugins and plugin configuration.**
- **Matrix configuration.** The matrix configuration is signed as a whole rather than each individual matrix job. This means the signature is the same for each job in the matrix. When signatures are verified for matrix jobs, the agent double-checks that the job it received is a valid construction of the matrix and that the signature matches the matrix configuration.
- **The step key.** If there is a step key present. [WIP]

Additionally, the following aspects of the build for each job are included in the signature:

- The pipeline's UUID [WIP]
- The organization UUID [WIP]
- The build number [WIP]
- The repo the build is running for [WIP]
- The commit the build is running for [WIP]


## Enabling signed pipelines on your agents

You'll need to configure your agents and update pipeline definitions to enable signed pipelines.

### 1. Generate a key pair

Behind the scenes, signed pipelines use [JSON Web Signing (JWS)](https://datatracker.ietf.org/doc/html/rfc7797) to generate signatures. You'll need to generate a [JSON Web Key Set (JWKS)](https://datatracker.ietf.org/doc/html/rfc7517) to sign and verify your pipelines with, then configure your agents to use those keys.

Luckily, the agent has you covered! A JWKS generation tool is built into the agent, which you can use to generate a key pair for you. To use it, you'll need to [install the agent on your machine](/docs/agent/v3/installation), and then run:

```bash
buildkite-agent tool keygen --alg <algorithm> --key-id <key-id>
```

Replacing the following:

- `<algorithm>` with the signing algorithm you want to use.
- `<key-id>` with the key ID you want to use.

For example, to generate an EdDSA key pair with a key ID of `my-key-id`, you'd run:

```bash
buildkite-agent tool keygen --alg EdDSA --key-id my-key-id
```

The agent generates two JWKS in your current directory: one private and one public. You can then use these keys to sign and verify your pipelines.

If no key ID is provided, the agent will generate a random one for you.

Note that the value of `--alg` must be a valid [JSON Web Signing Algorithm](https://datatracker.ietf.org/doc/html/rfc7518#section-3). The agent does not support the `none` algorithm, or any signature algorithms in the `RS` series (any signing algorithms that use RSASSA-PKCS1 v1.5 signatures, `RS256` for example).

<details>
  <summary>Why doesn't the agent support RSASSA-PKCS1 v1.5 signatures?</summary>
  <p>In short, RSASSA-PKCS1 v1.5 signatures are less secure than the newer RSA-PSS signatures. While RSASSA-PKCS1 v1.5 signatures are still relatively secure, we want to encourage our users to use the most secure algorithms possible, so when using RSA keys, we only support RSA-PSS signatures. We also recommend looking into ECDSA and EdDSA signatures, which are more secure than RSA signatures.</p>
</details>

#### Algorithm options

Signed pipelines support many different signing algorithms, which you can simplify into two categories:

- **Symmetric**—where signing and verification use the same keys.
- **Asymmetric**—where signing and verification use different keys.

We recommend using an asymmetric signing algorithm, as it creates a permissions boundary between the agents that upload and sign pipelines, and agents that run other jobs and can verify signatures. This means that if an attacker compromises an agent with verification keys, they can't modify the steps without detection.

When using asymmetric signing algorithms, we recommend having multiple disjoint pools of agents, each using a different [queue](/docs/agent/v3/queues). One pool should be the _uploaders_ and have access to the private keys. Another pool should be the _runners_ and have access to the public keys. This creates a security boundary between the agents that upload and sign pipelines and the agents that run jobs and verify signatures.

Regarding your specific algorithm choice, any of the supported asymmetric signing algorithms are fine and will be secure. If you're not sure which one to use, `EdDSA` is proven to be secure, has a modern design, wasn't designed by a Nation State Actor, and produces nice short signatures.

For more information on what algorithms the agent supports, run:

```sh
buildkite-agent tool keygen --help
```

### 2. Configure the agents

Next, you need to configure your agents to use the keys you generated. On agents that upload pipelines, add the following to the agent's config file:

```ini
job-signing-jwks-path=<path to private keyset>
job-signing-key-id=<the key id you generated earlier>
job-verification-jwks-path=<path to public keyset>
```

This ensures that whenever those agents upload steps to Buildkite, they'll generate signatures using the private key you generated earlier. It also ensures that those agents verify the signatures of any steps they run, using the public key.

On instances that verify jobs, add:

```ini
job-verification-jwks-path=<path to verification keys>
```

### 3. Sign all steps

So far, you've configured agents to sign and verify any steps they upload and run. But there's another source of steps: the Buildkite dashboard. Rather than defining steps in your repository, you can also define steps in a pipeline's settings. The most common use for this is to have a single step that uploads a pipeline definition from [a YAML file in the repository](/docs/pipelines/defining-steps#step-defaults-pipeline-dot-yml-file). These steps should also be signed.

> 🚧 Non-YAML steps
> You must use YAML to sign steps configured in the Buildkite dashboard. If you don't use YAML, you'll need to [migrate to YAML steps](/docs/tutorials/pipeline_upgrade) before continuing.

To sign steps configured in the Buildkite dashboard, you need to add static signatures to the YAML:

1. Copy the existing YAML steps definition from the editor in the Builkdite dashboard.
1. Paste the step definitions into a file on your machine.
1. Generate a new YAML definition with signatures by running:

    ```sh
    buildkite-agent pipeline upload --jwks-file-path <path-to-private-keys> --signing-key-id <key-id> --dry-run --format yaml <path-to-pipeline-file>
    ```

    Replacing the following:
    * `<path-to-private-keys>` with the path to your private keys.
    * `<key-id>` with the key ID.
    * `<path-to-pipeline-file>` with the path to the step definitions you saved from the Buildkite dashboard.

    This outputs the pipeline YAML with the signatures added to the steps.

1. Copy the updated step definitions back into the editor in the Buildkite dashboard.
1. Save the new definition.

For example, if you had a file called `pipeline.yaml` in the current directory, and you wanted to sign it with the key generated earlier, you'd run:

```bash
buildkite-agent pipeline upload --jwks-file-path /path/to/private/key.json --signing-key-id my-key-id --dry-run --format yaml pipeline.yaml
```

## Rotating signing keys

Because signed pipelines use JWKS as their key format, rotating keys is easy.

To rotate your keys:

1. Generate a new key pair.
1. Add the new keys to your existing key sets. Be careful not to mix public and private keys.

Once the updated key sets are in place, you can update the `signing-key-id` on your signing agents to use the new key ID. Verifying agents will automatically use the public key with the matching key ID, if it's present.
