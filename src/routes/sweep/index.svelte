<script>
  import cryptojs from "crypto-js";
  import { onMount } from "svelte";
  import * as liquid from "liquidjs-lib";
  import { mnemonicToSeedSync } from "bip39";
  import { fromBase58, fromSeed } from "bip32";

  import {
    address as Address,
    ECPair,
    Psbt,
    payments,
    networks,
    Transaction,
  } from "liquidjs-lib";

  let total = 0;
  let fee = 300;
  let network = networks.liquid;
  let btc = "6f0279e9ed041c3d710a9f57d0c02928416460c4b722ae3457a11eec381c526d";
  let password, destination, mnemonic, hex, a, empty;

  let submit = async () => {
    let decrypted = cryptojs.AES.decrypt(mnemonic, password).toString(
      cryptojs.enc.Utf8
    );
    let seed = mnemonicToSeedSync(decrypted);
    let key = fromSeed(seed, network).derivePath("m/84'/0'/0'/0/0");
    let { publicKey: pubkey, privateKey: privkey } = key;
    let base58 = key.neutered().toBase58();

    let {
      address,
      redeem: { output: redeemScript },
    } = payments.p2sh({
      redeem: payments.p2wpkh({
        pubkey,
        network,
      }),
      network,
    });

    a = address;

    let u = await fetch(
      `https://liquid.network/api/address/${address}/utxo`
    ).then((r) => r.json());
    u = u.filter((o) => o.asset === btc);

    let p = new Psbt();

    for (let o of u) {
      let raw = await fetch(`https://liquid.network/api/tx/${o.txid}/hex`).then(
        (r) => r.text()
      );

      total += o.value;
      p.addInput({
        hash: o.txid,
        index: o.vout,
        redeemScript,
        nonWitnessUtxo: Buffer.from(raw, "hex"),
      });
    }

    if (total < 1000) return (empty = true);

    p.addOutput({
      asset: btc,
      script: Address.toOutputScript(destination, network),
      value: total - fee,
    });

    p.addOutput({
      asset: btc,
      nonce: Buffer.alloc(1),
      script: Buffer.alloc(0),
      value: fee,
    });

    p.data.inputs.map((_, i) => {
      p = p.signInput(i, ECPair.fromPrivateKey(privkey)).finalizeInput(i);
    });

    hex = p.extractTransaction().toHex();
  };

  let broadcast = () => {
    post("/api/broadcast", { hex });
  };
</script>

<div class="container mx-auto mt-10 md:mt-20 space-y-5">
  <h1 class="text-3xl font-black primary-color">Rescue funds</h1>

  {#if hex}
    <div>
      Found <b>{total}</b> sats at <b>{a}</b>
    </div>

    <div>
      <div class="font-bold">Withdrawal hex</div>
      <div class="break-all">
        {hex}
      </div>
    </div>

    <div>
      <div class="font-bold">Destination address</div>
      <div class="break-all">
        {destination}
      </div>
    </div>

    <form
      class="flex flex-col mb-4 space-y-5"
      on:submit|preventDefault={broadcast}
    >
      <button
        type="submit"
        class="rounded-2xl w-80 bg-black text-white px-5 py-6 font-bold"
        >Send</button
      >
    </form>
  {:else if empty}
    <div>Address {a} is empty</div>
  {:else}
    <form
      class="flex flex-col mb-4 space-y-5"
      on:submit|preventDefault={submit}
    >
      <div class="mt-1 relative rounded-md shadow-sm">
        <label>Mnemonic</label>
        <textarea
          rows={5}
          class="form-input block w-full pl-7 pr-12"
          bind:value={mnemonic}
        />
      </div>

      <div class="mt-1 relative rounded-md shadow-sm">
        <label>Password</label>
        <input
          class="form-input block w-full pl-7 pr-12"
          bind:value={password}
        />
      </div>

      <div class="mt-1 relative rounded-md shadow-sm">
        <label>Destination address</label>
        <input
          class="form-input block w-full pl-7 pr-12"
          bind:value={destination}
        />
      </div>

      <button
        type="submit"
        class="rounded-2xl w-80 bg-black text-white px-5 py-6 font-bold"
        >Submit</button
      >
    </form>
  {/if}
</div>
