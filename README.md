# EOS Guardian


**EOS Guardian** is an EOS smart contract aiming to provide safer fund management with proper permission settings.

**EOS Guradian** is one of the many projects built by EOSLaoMao team.

VOTE EOSLaoMao: `eoslaomaocom`

## The origin

**EOS Guardian** originated from another smart contract [SafeDelegatebw](https://github.com/EOSLaoMao/SafeDelegatebw) which was built by EOSLaoMao team for [Bank of Staked](https://github.com/EOSLaoMao/BankofStaked-CE) creditors.

SafeDelegatebw is a contract that simply wraps up a new `delegatebw` action calling system contract's `delegatebw` action with the 5th parameter `--transfer` hardcoded to always be `false`. 

In this way, instead of granting system contract's `delegatebw` action permission to Bank of Staked(which is risky, because system contracts action could be called with `--transfer` specified to `true` which will cause EOS ownership change), creditors can deploy `SafeDelegatebw` and grant customized `delegatebw` action inside `SafeDelegatebw` contract.

The rules are simple: 

1. wrap up customized actions to lower the risks of actions provided by system contracts.
2. use lower permission to control these customized actions istead of using `active` or `owner` keys.

Then we thought about if there is a broader range of user cases that could be benefit from this approach. And here comes EOS Guardian.

## Key Functions

Khere are two functions provided by EOS Guardian:

### safedelegate

This function is similar to SafeDelegatebw smart contract. It hardcodes `--transfer` parameter to be always `false`, so that any delegate transactions trigger via `safedelegate` are guaranteed to have ZERO risk of losing any funds.

### safetransfer

This function is a little bit complex.

Image your EOS account was hacked with your `active` or `owner` key compromized. Currently by design, `active` or `owner` key could be used to do anyting including transfering all your EOS tokens out instantly.

Also, we noticed that there is an EOS account list that your should be careful doing business with. We saw cases that ECAF issued orders to freeze accounts that are associated with these high-risk accounts.

Above is where we got these security principles designing EOS Guardian:

1. Never use your `active` or `owner` key in a daily basis, use customized lower risk permissions instead.
2. There should be adjustable time based caps for your transfer while using customized permissions.
3. There should be a configurable blacklist/whitelist mechanism when you do transfers.


By using EOS permission system, we can achieve these goals in EOS Guardian.

#### 1. Global Cap Control

EOS Guardian provides a global cap control, you can specify how much you can transfered out `cap_total` given a specific `duration`(in minutes)
, you can also specify the cap per transaction `cap_tx`.

Here is the global cap settings:

```
cap_toal:     x EOS      // max amount of EOS you can transfer out in duration minutes
cap_tx:       y EOS      // max amount of EOS you can transfer out in one transaction
duration:     z          // cap_total duration, in minutes.
```

You can update these controls via `setsettings` action using your `active` permission:

```
cleos push action YOUR_ACCOUNT setsettings '{"cap_toal": "100.0000 EOS", "cap_tx": "1.0000 EOS", "duration": 60}' -p YOUR_ACCOUNT@active
```

You can use `safetransfer` to transfer EOS out using a lower risk permission `safeperm`:

```
cleos push action YOUR_ACCOUNT safertansfer '{"to": "SOME_ACCOUNT", "quantity": "1.0000 EOS", "memo": "test"}' -p YOUR_ACCOUNT@safeperm
```

Any `safetransfer` transaction will do a cap check before it trigger `transfer` in system contract. If any check fails, no funds will be transfered out.


#### 2. Whitelist

Global cap control is a default check for all `safetransfer` transactions. If you want set caps for specific accounts, you need whitelist.

The fields of whitelist table are similar to global cap control. Here is how you can add account B into your whitelist with diferrent caps and duration using your `active` permission:

```
cleos push action YOUR_ACCOUNT setwhitelist '{"account": "B", "cap_toal": "200.0000 EOS", "cap_tx": "2.0000 EOS", "duration": 360}' -p YOUR_ACCOUNT@active
```

after you added account B to whitelist account, any `safetransfer` to account B, will checking `caps_total` and `cap_tx` configured in whitelist, instead of global cap setting(whitelist override global settings).

Also, you can remove certain account from `whitelist` table using `delwhitelist` action.

```
cleos push action YOUR_ACCOUNT delwhitelist '{"account": "B"}' -p YOUR_ACCOUNT@active
```

#### 3. Blacklist

There are certains accounts we dont want to interact with. For example, accounts in ECAF orders or accounts you find suspicious. 

Simply add these accounts to `blacklist` table, any `safetransfer` will go through this table and make sure recipient account is not in it.

Here is how you can add and remove accounts from blacklist table:

```
cleos push action YOUR_ACCOUNT addblacklist '{"account": "HACKER_ACCOUNT"}' -p YOUR_ACCOUNT@active

cleos push action YOUR_ACCOUNT delblacklist '{"account": "HACKER_ACCOUNT"}' -p YOUR_ACCOUNT@active

```


## How to deploy

### 1. Build EOS Guardian contract

```
./build.sh
```

### 2. Deploy it to your account

make sure your accounts have enough RAM and deploy:

```
cleos set contract YOUR_ACCOUNT ./safeguardian/build/

```

### 3. Setup permissions

There is a script `scripts/perm.sh` to setup permissions. It will add two permissions under `active` permission.

First one, `guardianperm`, it will only be used by EOS Guardian contract which just been deployed under your account. So it only contains one code permission `YOUR_ACCOUNT@eosio.code`

Second one, `safeperm`, it will be used to fire `safetransfer` and `safedelegate`, it contains code permission and also a key for your daily uses. 

Prepare a new key pair for `safeperm` and setup like this:


```
./scripts/perm.sh YOUR_ACCOUNT SAFEPERM_PUBKEY
```

after setup, the permission structure should be like this:

```
permissions:
     owner     1:    1 OWNER_PUBKEY
        active     1:    1 ACTIVE_PUBKEY
           safeperm     1:    1 SAFEPERM_PUBKEY, 1 eosguardian1@eosio.code
           guardianperm     1:    1 eosguardian1@eosio.code

```

### Add Caps/Whitelist/Blacklist

Add global caps:


```
cleos push action YOUR_ACCOUNT setsettings '{"cap_toal": "100.0000 EOS", "cap_tx": "1.0000 EOS", "duration": 60}' -p YOUR_ACCOUNT@active
```

Add/delete Whitelist account:

```
cleos push action YOUR_ACCOUNT setwhitelist '{"account": "B", "cap_toal": "200.0000 EOS", "cap_tx": "2.0000 EOS", "duration": 360}' -p YOUR_ACCOUNT@active

cleos push action YOUR_ACCOUNT delwhitelist '{"account": "B"}' -p YOUR_ACCOUNT@active

```


Add/delete Blacklist account:

```
cleos push action YOUR_ACCOUNT addblacklist '{"account": "HACKER_ACCOUNT"}' -p YOUR_ACCOUNT@active

cleos push action YOUR_ACCOUNT delblacklist '{"account": "HACKER_ACCOUNT"}' -p YOUR_ACCOUNT@active

```

Use safetransfer and safedelegate:

```
cleos push action YOUR_ACCOUNT safertansfer '{"to": "SOME_ACCOUNT", "quantity": "1.0000 EOS", "memo": "test"}' -p YOUR_ACCOUNT@safeperm

cleos push action YOUR_ACCOUNT safedelegate '{"to": "SOME_ACCOUNT", "net_weight": "1.0000 EOS", "cpu_weight": "1.0000 EOS"}' -p YOUR_ACCOUNT@safeperm

```
