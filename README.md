<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Gifted Digit Analyzer</title>

  <style>
    * {
      box-sizing: border-box;
    }

    body {
      margin: 0;
      font-family: Arial, sans-serif;
      background: #0b1020;
      color: white;
    }

    .container {
      width: min(95%, 700px);
      margin: auto;
      padding: 20px 0 40px;
    }

    h1 {
      text-align: center;
      margin-bottom: 5px;
    }

    .subtitle {
      text-align: center;
      color: #9ca3af;
      margin-bottom: 20px;
    }

    .card {
      background: #151c30;
      border: 1px solid #27304a;
      border-radius: 15px;
      padding: 18px;
      margin-bottom: 15px;
    }

    label {
      display: block;
      margin-bottom: 7px;
      color: #cbd5e1;
      font-size: 14px;
    }

    select,
    input {
      width: 100%;
      padding: 13px;
      border-radius: 9px;
      border: 1px solid #36415e;
      background: #0e1528;
      color: white;
      font-size: 16px;
      margin-bottom: 14px;
    }

    button {
      width: 100%;
      padding: 15px;
      border: none;
      border-radius: 10px;
      font-size: 17px;
      font-weight: bold;
      cursor: pointer;
      margin-top: 8px;
    }

    #connectBtn {
      background: #2563eb;
      color: white;
    }

    #analyzeBtn {
      background: #7c3aed;
      color: white;
    }

    #tradeBtn {
      background: #16a34a;
      color: white;
    }

    button:disabled {
      opacity: 0.45;
      cursor: not-allowed;
    }

    .status {
      text-align: center;
      padding: 10px;
      border-radius: 8px;
      background: #0e1528;
      margin-bottom: 15px;
    }

    .digit {
      font-size: 65px;
      text-align: center;
      font-weight: bold;
      margin: 10px 0;
    }

    .prediction {
      font-size: 25px;
      text-align: center;
      margin: 10px 0;
    }

    .grid {
      display: grid;
      grid-template-columns: repeat(5, 1fr);
      gap: 8px;
    }

    .digitBox {
      background: #0e1528;
      border: 1px solid #27304a;
      border-radius: 8px;
      padding: 10px 4px;
      text-align: center;
    }

    .digitBox strong {
      display: block;
      font-size: 20px;
    }

    .digitBox span {
      color: #9ca3af;
      font-size: 12px;
    }

    #log {
      max-height: 220px;
      overflow-y: auto;
      font-size: 13px;
      color: #cbd5e1;
    }

    .logLine {
      border-bottom: 1px solid #27304a;
      padding: 7px 0;
    }

    .warning {
      color: #fbbf24;
    }

    .success {
      color: #4ade80;
    }

    .danger {
      color: #f87171;
    }
  </style>
</head>

<body>

<div class="container">

  <h1>Gifted Digit Analyzer</h1>
  <div class="subtitle">Deriv Live Digit Analysis</div>

  <div class="card">

    <div id="connectionStatus" class="status">
      Disconnected
    </div>

    <label>Market</label>

    <select id="symbol">
      <option value="R_10">Volatility 10 Index</option>
      <option value="R_25">Volatility 25 Index</option>
      <option value="R_50">Volatility 50 Index</option>
      <option value="R_75">Volatility 75 Index</option>
      <option value="R_100">Volatility 100 Index</option>
    </select>

    <label>Stake</label>

    <input
      id="stake"
      type="number"
      value="1"
      min="0.35"
      step="0.01"
    >

    <button id="connectBtn">
      CONNECT MARKET
    </button>

    <button id="analyzeBtn" disabled>
      ANALYZE
    </button>

    <button id="tradeBtn" disabled>
      TRADE ONE MATCH
    </button>

  </div>

  <div class="card">

    <h3>Latest Tick</h3>

    <div id="latestPrice" class="digit">
      --
    </div>

    <div>
      Last digit:
      <strong id="lastDigit">-</strong>
    </div>

  </div>

  <div class="card">

    <h3>Prediction</h3>

    <div id="prediction" class="prediction">
      No prediction
    </div>

    <div id="confidence" class="status">
      Waiting for analysis
    </div>

  </div>

  <div class="card">

    <h3>Digit Statistics</h3>

    <div id="digitGrid" class="grid"></div>

  </div>

  <div class="card">

    <h3>Trade Status</h3>

    <div id="tradeStatus" class="status">
      No trade
    </div>

  </div>

  <div class="card">

    <h3>Activity</h3>

    <div id="log"></div>

  </div>

</div>

<script>

const state = {

  ws: null,

  connected: false,

  analyzing: false,

  tradeArmed: false,

  tradeLocked: false,

  ticks: [],

  lastPrice: null,

  prediction: null,

  contractId: null,

  symbol: "R_10",

  stake: 1,

  requestId: 1

};

const $ = id => document.getElementById(id);

function log(message, type = "") {

  const row = document.createElement("div");

  row.className = "logLine " + type;

  row.textContent =
    new Date().toLocaleTimeString() + " — " + message;

  $("log").prepend(row);

}

function setStatus(message) {

  $("connectionStatus").textContent = message;

}

function nextRequestId() {

  return state.requestId++;

}

function getLastDigit(price) {

  const text = String(price);

  const parts = text.split(".");

  if (parts.length < 2) {

    return Number(text[text.length - 1]);

  }

  const decimals = parts[1];

  return Number(decimals[decimals.length - 1]);

}

function updateLatestTick(price) {

  const digit = getLastDigit(price);

  state.lastPrice = price;

  $("latestPrice").textContent = price;

  $("lastDigit").textContent = digit;

  state.ticks.push({

    price: price,

    digit: digit,

    time: Date.now()

  });

  if (state.ticks.length > 2000) {

    state.ticks.shift();

  }

  if (state.analyzing) {

    updateStatistics();

  }

}

function calculateStatistics() {

  const ticks = state.ticks.slice(-1000);

  const counts = Array(10).fill(0);

  const recent = ticks.slice(-100);

  const recentCounts = Array(10).fill(0);

  ticks.forEach(t => {

    counts[t.digit]++;

  });

  recent.forEach(t => {

    recentCounts[t.digit]++;

  });

  return {

    counts,

    recentCounts,

    total: ticks.length

  };

}

function choosePrediction() {

  if (state.ticks.length < 100) {

    return null;

  }

  const stats = calculateStatistics();

  const scores = [];

  for (let d = 0; d <= 9; d++) {

    const longFrequency =
      stats.counts[d] / Math.max(stats.total, 1);

    const recentFrequency =
      stats.recentCounts[d] / 100;

    /*
      This is intentionally only a statistical score.
      It does NOT claim to predict a random future digit.
    */

    const score =
      (longFrequency * 0.4) +
      (recentFrequency * 0.6);

    scores.push({

      digit: d,

      score: score,

      longFrequency: longFrequency,

      recentFrequency: recentFrequency

    });

  }

  scores.sort((a, b) => b.score - a.score);

  return scores[0];

}

function updateStatistics() {

  const stats = calculateStatistics();

  const grid = $("digitGrid");

  grid.innerHTML = "";

  for (let d = 0; d <= 9; d++) {

    const count = stats.counts[d];

    const percentage =
      stats.total ?
      ((count / stats.total) * 100).toFixed(1) :
      "0.0";

    const box =
      document.createElement("div");

    box.className = "digitBox";

    box.innerHTML = `
      <strong>${d}</strong>
      <span>${count} (${percentage}%)</span>
    `;

    grid.appendChild(box);

  }

}

function analyze() {

  if (state.ticks.length < 100) {

    $("prediction").textContent =
      "Collecting ticks...";

    $("confidence").textContent =
      `${state.ticks.length}/100 ticks`;

    return;

  }

  const prediction = choosePrediction();

  if (!prediction) return;

  state.prediction = prediction.digit;

  $("prediction").textContent =
    `DIGIT ${prediction.digit}`;

  $("confidence").textContent =
    `Statistical score: ${(prediction.score * 100).toFixed(2)}%`;

  log(
    `Analysis candidate: digit ${prediction.digit}`,
    "warning"
  );

  $("tradeBtn").disabled = false;

}

function connectMarket() {

  if (state.ws) {

    state.ws.close();

  }

  state.symbol = $("symbol").value;

  const url =
    "wss://api.derivws.com/trading/v1/options/ws/public";

  state.ws = new WebSocket(url);

  setStatus("Connecting...");

  state.ws.onopen = () => {

    state.connected = true;

    setStatus("Connected");

    $("connectBtn").disabled = true;

    $("analyzeBtn").disabled = false;

    log("Connected to Deriv public market stream.", "success");

    state.ws.send(JSON.stringify({

      ticks: state.symbol,

      subscribe: 1,

      req_id: nextRequestId()

    }));

  };

  state.ws.onmessage = event => {

    let data;

    try {

      data = JSON.parse(event.data);

    } catch {

      return;

    }

    if (data.error) {

      log(
        "Deriv error: " + data.error.message,
        "danger"
      );

      return;

    }

    if (data.msg_type === "tick") {

      updateLatestTick(data.tick.quote);

      /*
        If the user has armed a trade, the next qualifying
        tick can trigger the trade workflow.
      */

      if (
        state.tradeArmed &&
        !state.tradeLocked &&
        state.prediction !== null
      ) {

        state.tradeArmed = false;

        state.tradeLocked = true;

        $("tradeBtn").disabled = true;

        executeTrade(state.prediction);

      }

    }

  };

  state.ws.onerror = () => {

    setStatus("Connection error");

    log("WebSocket connection error.", "danger");

  };

  state.ws.onclose = () => {

    state.connected = false;

    $("connectBtn").disabled = false;

    setStatus("Disconnected");

    log("Market connection closed.", "danger");

  };

}

function armTrade() {

  if (!state.connected) {

    alert("Connect to Deriv first.");

    return;

  }

  if (state.tradeLocked) {

    alert(
      "Trade is locked. Wait for the current trade to finish."
    );

    return;

  }

  const prediction = choosePrediction();

  if (prediction === null) {

    alert(
      "Collect at least 100 ticks before trading."
    );

    return;

  }

  state.prediction = prediction.digit;

  state.tradeArmed = true;

  $("tradeStatus").textContent =
    `ARMED — waiting for next tick. Target digit: ${state.prediction}`;

  log(
    `Trade armed for DIGITMATCH ${state.prediction}.`,
    "warning"
  );

}

function send(message) {

  if (
    !state.ws ||
    state.ws.readyState !== WebSocket.OPEN
  ) {

    throw new Error("WebSocket is not connected.");

  }

  state.ws.send(JSON.stringify(message));

}

function executeTrade(digit) {

  state.stake =
    Number($("stake").value);

  if (
    !Number.isFinite(state.stake) ||
    state.stake <= 0
  ) {

    finishTrade();

    alert("Invalid stake.");

    return;

  }

  $("tradeStatus").textContent =
    `Requesting DIGITMATCH ${digit}...`;

  log(
    `Requesting DIGITMATCH proposal for digit ${digit}.`
  );

  /*
    IMPORTANT:
    This is where an AUTHENTICATED trading WebSocket
    must be used for real purchases.

    The public market socket is intentionally NOT used
    to buy real contracts.
  */

  requestTradeProposal(digit);

}

function requestTradeProposal(digit) {

  /*
    PLACEHOLDER FOR AUTHENTICATED SOCKET

    The authenticated socket should send:

    {
      proposal: 1,
      amount: state.stake,
      basis: "stake",
      contract_type: "DIGITMATCH",
      currency: "USD",
      duration: 1,
      duration_unit: "t",
      barrier: String(digit),
      underlying_symbol: state.symbol
    }

    Then wait for:

    msg_type === "proposal"

    and receive:

    data.proposal.id
    data.proposal.ask_price

    Then send:

    {
      buy: data.proposal.id,
      price: Number(data.proposal.ask_price)
    }

    The real authenticated connection must be
    implemented server-side.
  */

  $("tradeStatus").textContent =
    "Trade engine ready — authenticated trading connection required.";

  log(
    "Market analysis works. Real purchase requires the authenticated backend.",
    "warning"
  );

  /*
    For now we deliberately DO NOT send a real-money
    buy request from the public browser connection.
  */

  finishTrade();

}

function finishTrade() {

  state.tradeLocked = false;

  state.tradeArmed = false;

  $("tradeBtn").disabled = false;

}

$("connectBtn").addEventListener(
  "click",
  connectMarket
);

$("analyzeBtn").addEventListener(
  "click",
  analyze
);

$("tradeBtn").addEventListener(
  "click",
  armTrade
);

$("symbol").addEventListener(
  "change",
  () => {

    if (state.ws) {

      state.ws.close();

    }

    state.ticks = [];

    state.prediction = null;

    $("prediction").textContent =
      "No prediction";

    $("lastDigit").textContent = "-";

    $("latestPrice").textContent = "--";

  }
);

</script>

</body>
</html>
