# Initializing Vault

> **:warning: This documentation assumes you have Vault installed in your machine, and not inside an Docker container.** Please [see the install docs](https://vaultproject.io/docs/install) for instuctions on how to install Vault.

## The `operator init` Command 101

The `operator init` command in Vault CLI can be only used once on an newly-installed node or if setting up an additional storage backend (e.g. if you set up Raft in an Compose/Kubernetes installation).

**Flags used in this documentation**:

* `key-shares` - Number of unseal keys to be generated and distributed to other operaators when sharing an Vault instance.
* `key-threshold` - Number to unseal keys needed to unseal the encrypted Vault storage and to decrypt the master key. Usually this should be 50-75% of total unseal keys.
* `pgp-keys` - List of GPG public keys encoded in base64 without ASCII armoring or Keybase usernames with `keybase:` prefix to be used to encrypt the unseal keys

## Generating unseal keys

Once you run `vault operator init`, that will be the last time you can see your unseal keys and also your root key. Once you use `clear` or `exit`, these will be lost forever. You can optionally redirect output to a file, just in case.

### On CLI with Keybase

If you're new to GPG, it's advised to use Keybase because it handles the tendeious parts like entering your key's password, storage and even trust. This method assumes that you installed Keybase CLI on the host machine since Keybase CLI isn't available on Alpine Linux inside our web terminal container.

> **Note:** Keybase support is available only in the command-line tool and not via the Vault HTTP API, tools that help with initialization should use the Keybase API in order to obtain the GPG keys needed for a secure initialization if you want them to use Keybase for keys.

1. [Sign up for an Keybase account.](https://keybase.io) After signing up, it is advised to [install the CLI and GUI client (in the host machine, not inside the web terminal)](https://keybase.io/download) and either generate your GPG key or import an existing one. We also recommend to generate atleast one paper key in case yu cannot access your computer or you're in danger of getting locked out.
2. Set `VAULT_SERVER` variable to point into your Vault instance. In case you're running it locally or in Gitpod, set this variable to either `http://localhost:3000` or `https://3000-workspace-id-here.ws.gitpod-instance.tld` (`https://3000-workspace-id.ws-regionNumber.gitpod.io` where `regionNumber` is the GCP region like `eu` or `us` and an number the meta nodes/clusters had been choosen).

```sh
# For Compose-based installs on the same machine
$ export VAULT_ADDR=http://localhost:3000

# If deployed in your PaaS of choice with Dockerfile support
$ export VAULT_ADDR=https://vault-thepinsteam.up.railway.app

# For Compose-based installs in Gitpod
$ export VAULT_ADDR=https://3000-workspace-id.ws-regionNumber.gitpod.io
```

4. Run the following `operator init` command to generate unseal keys and decode them with Keybase CLI.

```sh
# First of all, start the Keybase service without KBFS and GUI. If you're on an desktop distro or installed an desktop environment
# on an server distro, you can open the Keybase GUI from the app drawer and it'll start Keybase service too.
$ run_keybase -fg
# Customize the vaules for key-shares and key-threshold for your needs, remember that the threshould should be atleast 50-75 percent
# of the total key shares generated for maximum security.
$ vault operator init -key-shares=5 -key-threshold=3 \
    # Change these values with your Keybase.io username prefixed in keybase: and possibly your
    # other Vault operators, just in case you lost your devices/paper keys
    -pgp-keys="keybase:ajhaliliphdev06,keybase:thepinsteam,keybase:rtappbot"
# Example output from the Vault docs:
# Key 1: wcBMA37rwGt6FS1VAQgAk1q8XQh6yc...
# Key 2: wcBMA0wwnMXgRzYYAQgAavqbTCxZGD...
# Key 3: wcFMA2DjqDb4YhTAARAAeTFyYxPmUd...
# ...

# Though key shares can be distributed over almost any medium, common sense and judgement are best advised to ensure no shitfuckery
# will be happened. The encrypted keys are base64 encoded before returning. The command below are from the Vault docs btw, so replace
# wcBMA37... with your base64-encoded unseal key.
$ echo "wcBMA37..." | base64 --decode | gpg -dq
```

4. Unseal the Vault with `vault operator unseaal`. Repeat this step until the **Sealed** status is now `false` when yoou do `vault status`.

```sh
# After running this, check its status at 'vault status'
vault operator unseal [decrypted-key-here]
```

### On CLI with GPG

Things start to blur when you use GNU Privacy Guard, an implementation of OpenPGP, since you need to export the public keys and decode them in base64 format. We're not looked at Vault's source code yet regarding this. While using Keybase is recommended for new users, if you don't trust Zoom or prehaps you want to send the encrypted keys with PGP, this section should work for you.

1. Make sure you have your GPG keys handy and in your keyring and your other operators' public keys. If not, generate your own GPG key (or if you have one, import it from backup) and also import your other operators' public keys.

2. Export the public keys without ASCII armoring and encode it to base64.

```sh
$ gpg --export <key-id-or-email> | base64 > user.asc
# For example, if you have Andrei Jiroh's public keys and decode it to base64, it looks like this
# when you cat your output file
# mQGNBGAaymQBDADcN3rgXqlWxG0RuhfZBSmvK0bnCZF2SkqoXH3QG9CVNRaOpxWACyJwXgjzZCHW
# D6mY1h+SqrPGr3q/CKZZ1UaJuBV1rh/9ebvQi653NHAnKr54SVGCHsSCO7rkbXlweCSyEaL2ZUKb
# ISehdfV6HJ2VM1FOXWx0zmYCHUD1iV9QNRqWa5GOBWozsVfJB30TpVvJcjEIXMXdbjlH2gTi/X8H
# vd7Vet/CXeeUJ9LD5KY3Ud1UeRpl71a9...
```

3. Run the following `operator init` command to generate unseal keys and decode them with

```sh
# Make sure GPG agent is up and running first
$ gpg-agent --daemon
# Customize the vaules for key-shares and key-threshold for your needs, remember that the threshould should be atleast 50-75 percent
# of the total key shares generated for maximum security.
$ vault operator init -key-shares=5 -key-threshold=3 \
    -pgp-keys="andreijiroh.asc,thepinsteam.asc,service-account.asc"
# Example output from the Vault docs:
# Key 1: wcBMA37rwGt6FS1VAQgAk1q8XQh6yc...
# Key 2: wcBMA0wwnMXgRzYYAQgAavqbTCxZGD...
# Key 3: wcFMA2DjqDb4YhTAARAAeTFyYxPmUd...
# ...

# Though key shares can be distributed over almost any medium, common sense and judgement are best advised to ensure no shitfuckery
# will be happened. The encrypted keys are base64 encoded before returning. The command below are from the Vault docs btw, so replace
# wcBMA37... with your base64-encoded unseal key.
$ echo "wcBMA37..." | base64 --decode | keybase pgp decrypt
```

4. Unseal the Vault with `vault operator unseaal`. Repeat this step until the **Sealed** status is now `false` when yoou do `vault status`.

```sh
# After running this, check its status at 'vault status'
vault operator unseal [decrypted-key-here]
```

### Without encrypting anyting on CLI

Okay, so you want to live on the edge? Fine, but you'll be warned that you SHOULD keep both unseal keys and root token secret. As per [the Vault docs](https://www.vaultproject.io/docs/concepts/tokens#root-tokens), once you setup correct permissions and ACLs, you can safely revoke root token.

TODO

### With the web UI

TODO

## Further Reading

* [Unsealing and Sealing Process](./unsealing-and-sealing.md)
* <https://www.vaultproject.io/docs/concepts/pgp-gpg-keybase> as the caanonical upstream source for this documentation page.
* <https://www.vaultproject.io/docs/commands/operator/init#usage> for list of available flags, including using PGP/Keybase for encrypting the root keys.
* <https://www.vaultproject.io/docs/concepts/seal> for those who want to know the why, the how, and the migration prcess between Auto Unseal and Shamir Key modes.
