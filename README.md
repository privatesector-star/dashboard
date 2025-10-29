<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Bitcoin Transaction Viewer</title>
  <style>
    body { font-family: Arial, sans-serif; max-width: 900px; margin: 2rem auto; }
    input[type=text] { width: 80%; padding: 8px; font-size: 14px; }
    button { padding: 8px 12px; font-size: 14px; }
    pre { background:#f7f7f7; padding:12px; overflow:auto }
    .hint { color: #666; font-size: 13px;}
  </style>
</head>
<body>
  <h1>Bitcoin Transaction Screener</h1>
  <p class="hint">Enter a Bitcoin txid OR turn on "Simulate Example"</p>

  <div>
    <input id="txid" type="text" placeholder="Paste Bitcoin transaction ID..." />
    <label><input id="sim" type="checkbox" /> Simulate Example (Fee $750 → Token $250,000)</label>
    <button id="scan">Scan</button>
  </div>

  <h3>Result</h3>
  <pre id="out">No scan yet.</pre>

  <script>
    async function btcUsd() {
      const r = await fetch("https://api.coingecko.com/api/v3/simple/price?ids=bitcoin&vs_currencies=usd");
      return (await r.json()).bitcoin.usd;
    }

    document.getElementById('scan').onclick = async () => {
      const out = document.getElementById('out');
      const txid = document.getElementById('txid').value.trim();
      const simulate = document.getElementById('sim').checked;

      // Simulation requested
      if (simulate) {
        out.textContent = JSON.stringify({
          simulated: true,
          txid: "SIMULATED-EXAMPLE",
          fee_usd: 750,
          token_value_usd: 250000,
          message: "Showing example: A $750 transaction fee for a token worth $250,000"
        }, null, 2);
        return;
      }

      if (!txid) {
        out.textContent = "❗ Enter a txid or enable simulation.";
        return;
      }

      out.textContent = "Fetching transaction...";

      try {
        const r = await fetch(`https://blockstream.info/api/tx/${txid}`);
        const tx = await r.json();
        const price = await btcUsd();

        let satIn = 0, satOut = 0;
        tx.vin.forEach(v => satIn += (v.prevout?.value || 0));
        tx.vout.forEach(v => satOut += (v.value || 0));
        const feeSat = satIn - satOut;
        const feeBtc = feeSat / 1e8;
        const feeUsd = +(feeBtc * price).toFixed(2);

        // Detect largest output
        const largest = tx.vout.sort((a,b)=>b.value - a.value)[0];
        const tokenUsd = +((largest.value / 1e8) * price).toFixed(2);

        out.textContent = JSON.stringify({
          txid,
          fee: { btc: feeBtc, usd: feeUsd },
          possible_token_value_usd: tokenUsd,
          note: "Largest output treated as token/purchase estimate.",
        }, null, 2);

      } catch (e) {
        out.textContent = "Error: " + e;
      }
    };
  </script>
</body>
</html>
