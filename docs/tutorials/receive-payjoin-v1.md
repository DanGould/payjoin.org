# Receive Payjoin V1

The `receive` module contains types and methods used to receive payjoin via BIP78.
Usage is pretty simple:

1. Build a  [BIP 21](https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki) using [`payjoin::PjUriBuilder`](crate::Uri)::from_str
2. Listen for a sender's request on the `pj` endpoint
3. Parse the request using [`UncheckedProposal::from_request()`](crate::receive::UncheckedProposal::from_request())
4. Validate the proposal using the `check` methods to guide you.
5. Assuming the proposal is valid, augment it into a payjoin with the available `try_preserving_privacy` and `contribute` methods
6. Extract the payjoin PSBT and sign it
7. Respond to the sender's http request with the signed PSBT as payload.

## Receive a Payjoin

The `receive` feature provides all of the check methods, PSBT data manipulation, coin selection, and transport structures to receive payjoin and handle errors in a privacy preserving way.

Receiving payjoin entails listening to a secure http endpoint for inbound requests. The endpoint is displayed in the `pj` parameter of a [bip 21](https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki) request URI.

The [reference implementation annotated below](https://github.com/payjoin/rust-payjoin/tree/master/payjoin-cli) uses `rouille` sync http server and Bitcoin Core RPC.

```
fn receive_payjoin(
    bitcoind: bitcoincore_rpc::Client,
    amount_arg: &str,
    endpoint_arg: &str,
) -> Result<()>
```

### 1. Generate a pj_uri BIP21 using `payjoin::Uri::from_str`

A BIP 21 URI supporting payjoin contains at minimum a bitcoin address and a secure `pj` endpoint.

```
let pj_receiver_address = bitcoind.get_new_address(None, None)?;
let amount = Amount::from_sat(amount_arg.parse()?);
let pj_uri_string = format!(
    "{}?amount={}&pj={}",
    pj_receiver_address.to_qr_uri(),
    amount.to_btc(),
    endpoint_arg
);
let pj_uri = Uri::from_str(&pj_uri_string)
    .map_err(|e| anyhow!("Constructed a bad URI string from args: {}", e))?;
let _pj_uri = pj_uri
    .check_pj_supported()
    .map_err(|e| anyhow!("Constructed URI does not support payjoin: {}", e))?;
```

### 2. Listen for a sender's request on the `pj` endpoint

Start a webserver of your choice to respond to payjoin protocol POST messages. The reference `payjoin-cli` implementation uses `rouille` sync http server.

```
rouille::start_server(self.config.pj_host.clone(), move |req| self.handle_web_request(req));
// ...
fn handle_web_request(&self, req: &Request) -> Response {
    log::debug!("Received request: {:?}", req);
    match (req.method(), req.url().as_ref()) {
        // ...
        ("POST", _) => self
            .handle_payjoin_post(req)
            .map_err(|e| match e {
                Error::BadRequest(e) => {
                    log::error!("Error handling request: {}", e);
                    Response::text(e.to_string()).with_status_code(400)
                }
                e => {
                    log::error!("Error handling request: {}", e);
                    Response::text(e.to_string()).with_status_code(500)
                }
            })
            .unwrap_or_else(|err_resp| err_resp),
        _ => Response::empty_404(),
    }
}
```

### 3. Parse an incoming request using [`UncheckedProposal::from_request()`](crate::receive::UncheckedProposal::from_request())

Parse the incoming HTTP request and check that it follows protocol.

```
let headers = Headers(req.headers());
let proposal = payjoin::receive::UncheckedProposal::from_request(
    req.data().context("Failed to read request body")?,
    req.raw_query_string(),
    headers,
)?;
```

Headers are parsed using the [`payjoin::receiver::Headers`] Trait so that the library can iterate through them, ideally without cloning.

```
struct Headers<'a>(rouille::HeadersIter<'a>);
impl payjoin::receive::Headers for Headers<'_> {
    fn get_header(&self, key: &str) -> Option<&str> {
        let mut copy = self.0.clone(); //! lol
        copy.find(|(k, _)| k.eq_ignore_ascii_case(key)).map(|(_, v)| v)
    }
}
```

### 4. Validate the proposal using the `check` methods

Check the sender's Original PSBT to prevent it from stealing, probing to fingerprint its wallet, and to avoid privacy gotchas.

```
// in a payment processor where the sender could go offline, this is where you schedule to broadcast the original_tx
let _to_broadcast_in_failure_case = proposal.extract_tx_to_schedule_broadcast();

// The network is used for checks later
let network = match bitcoind.get_blockchain_info()?.chain.as_str() {
    "main" => bitcoin::Network::Bitcoin,
    "test" => bitcoin::Network::Testnet,
    "regtest" => bitcoin::Network::Regtest,
    _ => return Err(ReceiveError::Other(anyhow!("Unknown network"))),
};
```

#### Check 1: Can the Original PSBT be Broadcast?

We need to know this transaction is consensus-valid.

```
let checked_1 = proposal.check_broadcast_suitability(None, |tx| {
        let raw_tx = bitcoin::consensus::encode::serialize(&tx).to_hex();
        let mempool_results = self
            .bitcoind
            .test_mempool_accept(&[raw_tx])
            .map_err(|e| Error::Server(e.into()))?;
        match mempool_results.first() {
            Some(result) => Ok(result.allowed),
            None => Err(Error::Server(
                anyhow!("No mempool results returned on broadcast check").into(),
            )),
        }
    })?;
```

If writing a payment processor, schedule that this transaction is broadcast as fallback if the payjoin fails after a timeout. BTCPay broadcasts fallback after two minutes.

#### Check 2: Is the sender trying to make us sign our own inputs?

```
let checked_2 = checked_1.check_inputs_not_owned(|input| {
        if let Ok(address) = bitcoin::Address::from_script(input, network) {
            self.bitcoind
                .get_address_info(&address)
                .map(|info| info.is_mine.unwrap_or(false))
                .map_err(|e| Error::Server(e.into()))
        } else {
            Ok(false)
        }
    })?;
```

#### Check 3: Are there mixed input scripts, breaking stenographic privacy?

```
let checked_3 = checked_2.check_no_mixed_input_scripts()?;
```

#### Check 4: Have we seen this input before?

Non-interactive i.e. payment processors should be careful to keep track of request inputs or else a malicious sender may try and probe multiple responses containing the receiver utxos, clustering their wallet.

```
let mut checked_4 = checked_3.check_no_inputs_seen_before(|input| {
    Ok(!self.insert_input_seen_before(*input).map_err(|e| Error::Server(e.into()))?)
})?;
```

### 5. Augment a valid proposal to preserve privacy

Here's where the PSBT is modified. Inputs may be added to break common input ownership heurstic. There are a number of ways to select coins and break common input heuristic but fail to preserve privacy because of  Unnecessary Input Heuristic (UIH). Until February 2023, even BTCPay occasionally made these errors. Privacy preserving coin selection as implemented in `try_preserving_privacy` is precarious to implement yourself may be the most sensitive and valuable part of this kit.

Output substitution is another way to improve privacy and increase functionality. For example, if the Original PSBT output address paying the receiver is coming from a static URI, a new address may be generated on the fly to avoid address reuse. This can even be done from a watch-only wallet. Output substitution may also be used to consolidate incoming funds to a remote cold wallet, break an output into smaller UTXOs to fulfill exchange orders, open lightning channels, and more.

```
// Distinguish our outputs to augment with input amount
let mut payjoin = checked_4.identify_receiver_outputs(|output_script| {
    if let Ok(address) = bitcoin::Address::from_script(output_script, network) {
        self.bitcoind
            .get_address_info(&address)
            .map(|info| info.is_mine.unwrap_or(false))
            .map_err(|e| Error::Server(e.into()))
    } else {
        Ok(false)
    }
})?;
// Select receiver payjoin inputs.
_ = try_contributing_inputs(&mut payjoin, bitcoind)
    .map_err(|e| log::warn!("Failed to contribute inputs: {}", e));

let receiver_substitute_address = bitcoind.get_new_address(None, None)?;
payjoin.substitute_output_address(receiver_substitute_address);

// ...

fn try_contributing_inputs(
    payjoin: &mut PayjoinProposal,
    bitcoind: &bitcoincore_rpc::Client,
) -> Result<()> {
    use bitcoin::OutPoint;

    let available_inputs = bitcoind
        .list_unspent(None, None, None, None, None)
        .context("Failed to list unspent from bitcoind")?;
    let candidate_inputs: HashMap<Amount, OutPoint> = available_inputs
        .iter()
        .map(|i| (i.amount, OutPoint { txid: i.txid, vout: i.vout }))
        .collect();

    let selected_outpoint = payjoin.try_preserving_privacy(candidate_inputs).expect("gg");
    let selected_utxo = available_inputs
        .iter()
        .find(|i| i.txid == selected_outpoint.txid && i.vout == selected_outpoint.vout)
        .context("This shouldn't happen. Failed to retrieve the privacy preserving utxo from those we provided to the seclector.")?;
    log::debug!("selected utxo: {:#?}", selected_utxo);

    // calculate receiver payjoin outputs given receiver payjoin inputs and original_psbt
    let txo_to_contribute = bitcoin::TxOut {
        value: selected_utxo.amount.to_sat(),
        script_pubkey: selected_utxo.script_pub_key.clone(),
    };
    let outpoint_to_contribute =
        bitcoin::OutPoint { txid: selected_utxo.txid, vout: selected_utxo.vout };
    payjoin.contribute_witness_input(txo_to_contribute, outpoint_to_contribute);
    Ok(())
}
```

Using methods for coin selection not provided by this library may have dire implications for privacy. Significant in-depth research and careful implementation iteration has gone into privacy preserving transaction construction. [Here's a good starting point from the JoinMarket repo to being a deep dive of your own](https://gist.github.com/AdamISZ/4551b947789d3216bacfcb7af25e029e#gistcomment-2796539).

### 6. Extract the payjoin PSBT and sign it

Fees are applied to the augmented Payjoin Proposal PSBT using calculation factoring both receiver's preferred feerate and the sender's fee-related [optional parameters](https://github.com/bitcoin/bips/blob/master/bip-0078.mediawiki#optional-parameters). The current `apply_fee` method is primitive, disregarding PSBT fee estimation and only adding fees coming from the sender's budget. When more accurate tools are available to calculate a PSBT's fee-dependent weight (solved, more complicated than it sounds, but unimplemented in rust-bitcoin), this `apply_fee` should be improved.

```
let min_feerate_sat_per_vb = 1;
let payjoin_proposal_psbt = payjoin.apply_fee(Some(min_feerate_sat_per_vb))?;

log::debug!("Extracted PSBT: {:#?}", payjoin_proposal_psbt);
// Sign payjoin psbt
let payjoin_base64_string =
    base64::encode(bitcoin::consensus::serialize(&payjoin_proposal_psbt));
// `wallet_process_psbt` adds available utxo data and finalizes
let payjoin_proposal_psbt =
    bitcoind.wallet_process_psbt(&payjoin_base64_string, sign: None, sighash_type: None, bip32derivs: Some(false))?.psbt;
let payjoin_proposal_psbt =
    Psbt::from_str(&payjoin_proposal_psbt).with_context(|| "Failed to parse PSBT")?;
```

### 7. Respond to the sender's http request with the signed PSBT as payload

BIP 78 senders require specific PSBT validation constraints regulated by prepare_psbt. PSBTv0 was not designed to support input/output modification, so the protocol requires this precise preparation step. A future PSBTv2 payjoin protocol may not.

It is critical to pay special care when returning error response messages. Responding with internal errors can make a receiver vulnerable to sender probing attacks which cluster UTXOs.

```
let payjoin_proposal_psbt = payjoin.prepare_psbt(payjoin_proposal_psbt)?;
let payload = base64::encode(bitcoin::consensus::serialize(&payjoin_proposal_psbt));
Ok(Response::text(payload))
```

📥 That's how one receives a payjoin.
