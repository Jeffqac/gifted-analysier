<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Deriv Digit Match Analyzer</title>

<style>
*{box-sizing:border-box}

body{
  margin:0;
  background:#080d18;
  color:#fff;
  font-family:Arial,sans-serif;
  padding:15px
}

.app{
  max-width:700px;
  margin:auto
}

h1{
  text-align:center;
  margin:5px 0
}

.subtitle{
  text-align:center;
  color:#9da8bd;
  margin-bottom:18px
}

.card{
  background:#121a2b;
  border-radius:15px;
  padding:16px;
  margin-bottom:14px
}

label{
  display:block;
  margin-bottom:7px
}

select,button{
  width:100%;
  padding:15px;
  border:0;
  border-radius:10px;
  font-size:16px
}

select{
  background:#222d46;
  color:white;
  margin-bottom:12px
}

button{
  background:#00c878;
  color:white;
  font-weight:bold
}

button:active{
  transform:scale(.97)
}

.status{
  background:#222d46;
  padding:13px;
  border-radius:10px;
  text-align:center;
  font-weight:bold;
  margin-bottom:14px
}

.match-title{
  text-align:center;
  color:#9da8bd;
  font-size:13px
}

.match{
  text-align:center;
  font-size:70px;
  font-weight:bold
}

.center{
  text-align:center
}

.grid{
  display:grid;
  grid-template-columns:repeat(5,1fr);
  gap:7px;
  margin-top:12px
}

.digit{
  background:#202b43;
  padding:10px 3px;
  border-radius:8px;
  text-align:center
}

.digit b{
  display:block;
  font-size:18px
}

.digit span{
  color:#aab5ca;
  font-size:11px
}

.row{
  display:flex;
  justify-content:space-between;
  padding:8px 0;
  border-bottom:1px solid #263149
}

.value{
  font-weight:bold
}

.recent{
  background:#080d18;
  border-radius:9px;
  padding:12px;
  margin-top:10px;
  word-break:break-word;
  line-height:1.8
}

.small{
  color:#8e9ab0;
  font-size:12px;
  line-height:1.5
}
</style>
</head>

<body>

<div class="app">

<h1>DERIV DIGIT MATCH</h1>

<div class="subtitle">
Live Tick & Digit Analyzer
</div>


<div class="card">

<label>Select Market</label>

<select id="market">

<option value="1HZ10V">
Volatility 10 (1s)
</option>

<option value="1HZ25V">
Volatility 25 (1s)
</option>

<option value="1HZ50V">
Volatility 50 (1s)
</option>

<option value="1HZ75V">
Volatility 75 (1s)
</option>

<option value="1HZ100V" selected>
Volatility 100 (1s)
</option>

</select>


<button
type="button"
onclick="startAnalysis()">

ANALYZE

</button>

</div>


<div id="status" class="status">
READY — PRESS ANALYZE
</div>


<div class="card">

<div class="match-title">
MATCH CANDIDATE
</div>

<div id="match" class="match">
—
</div>

<div id="signal" class="center">
Waiting for ticks...
</div>

</div>


<div class="card">

<div class="row">
<span>Ticks</span>
<span id="ticks" class="value">0</span>
</div>

<div class="row">
<span>Last Price</span>
<span id="price" class="value">—</span>
</div>

<div class="row">
<span>Last Digit</span>
<span id="lastDigit" class="value">—</span>
</div>

<div class="row">
<span>Latest Jump</span>
<span id="jump" class="value">—</span>
</div>

<div class="row">
<span>Average Jump</span>
<span id="avgJump" class="value">—</span>
</div>

</div>


<div class="card">

<b>Digit Frequencies</b>

<div class="grid">

<div class="digit"><b id="p0">0%</b><span>0</span></div>
<div class="digit"><b id="p1">0%</b><span>1</span></div>
<div class="digit"><b id="p2">0%</b><span>2</span></div>
<div class="digit"><b id="p3">0%</b><span>3</span></div>
<div class="digit"><b id="p4">0%</b><span>4</span></div>
<div class="digit"><b id="p5">0%</b><span>5</span></div>
<div class="digit"><b id="p6">0%</b><span>6</span></div>
<div class="digit"><b id="p7">0%</b><span>7</span></div>
<div class="digit"><b id="p8">0%</b><span>8</span></div>
<div class="digit"><b id="p9">0%</b><span>9</span></div>

</div>

</div>


<div class="card">

<b>Recent Digits</b>

<div id="recent" class="recent">
Waiting...
</div>

</div>


<div class="card small">

This analyzer reads live market ticks and calculates observed digit frequencies and jump statistics. The match candidate is a statistical result, not a guarantee of the next digit.

</div>

</div>


<script>

let ws = null;

let digits = [];

let prices = [];

let tickCount = 0;

let endpointNumber = 0;

let fallbackTimer = null;


/* =====================================
   START
===================================== */

function startAnalysis(){

  const status =
    document.getElementById("status");

  const market =
    document.getElementById("market").value;


  /* Immediate button response */

  status.innerHTML =
    "STARTING ANALYSIS...";


  document.getElementById("signal").innerHTML =
    "Preparing live connection...";


  document.getElementById("match").innerHTML =
    "—";


  digits = [];
  prices = [];
  tickCount = 0;


  updateScreen();


  if(ws){

    try{
      ws.close();
    }catch(e){}

  }


  /* Start with current Deriv public endpoint */

  endpointNumber = 1;

  connectToDeriv(market);

}


/* =====================================
   CONNECT
===================================== */

function connectToDeriv(market){

  const status =
    document.getElementById("status");

  let url;


  if(endpointNumber === 1){

    url =
      "wss://api.derivws.com/trading/v1/options/ws/public";

    status.innerHTML =
      "CONNECTING — CURRENT DERIV SERVER...";

  }else{

    url =
      "wss://ws.binaryws.com/websockets/v3";

    status.innerHTML =
      "TRYING DERIV BACKUP SERVER...";

  }


  try{

    ws =
      new WebSocket(url);

  }catch(error){

    tryBackup(market);

    return;

  }


  let connected = false;


  ws.onopen = function(){

    connected = true;


    if(fallbackTimer){

      clearTimeout(fallbackTimer);

    }


    status.innerHTML =
      "CONNECTED — SCANNING";


    document.getElementById("signal").innerHTML =
      "Receiving live " + market + " ticks...";


    /* Subscribe to ticks */

    ws.send(JSON.stringify({

      ticks:market,

      subscribe:1,

      req_id:100

    }));

  };


  /*
     If the connection opens but
     no useful data arrives, the
     backup will be tried.
  */

  fallbackTimer =
    setTimeout(function(){

      if(
        !connected ||
        tickCount === 0
      ){

        if(ws){

          try{
            ws.close();
          }catch(e){}

        }

        tryBackup(market);

      }

    },7000);


  ws.onmessage = function(event){

    let data;


    try{

      data =
        JSON.parse(event.data);

    }catch(e){

      return;

    }


    if(data.error){

      document.getElementById("signal").innerHTML =
        data.error.message;

      return;

    }


    if(
      data.msg_type === "tick" &&
      data.tick
    ){

      readTick(data.tick);

    }

  };


  ws.onerror = function(){

    if(!connected){

      tryBackup(market);

    }else{

      status.innerHTML =
        "CONNECTION ERROR";

    }

  };


  ws.onclose = function(){

    if(
      !connected &&
      tickCount === 0
    ){

      tryBackup(market);

    }

  };

}


/* =====================================
   BACKUP
===================================== */

function tryBackup(market){

  if(endpointNumber === 1){

    endpointNumber = 2;

    setTimeout(function(){

      connectToDeriv(market);

    },500);

  }else{

    document.getElementById("status").innerHTML =
      "COULD NOT CONNECT TO DERIV";

    document.getElementById("signal").innerHTML =
      "Both public connections failed. Check your internet connection or network WebSocket blocking.";

  }

}


/* =====================================
   READ TICK
===================================== */

function readTick(tick){

  const price =
    Number(tick.quote);


  if(!Number.isFinite(price)){
    return;
  }


  /*
    Deriv's tick response provides
    quote as the current price.
  */

  const text =
    String(tick.quote);


  const cleaned =
    text.replace(".","");


  const digit =
    Number(
      cleaned.slice(-1)
    );


  if(
    !Number.isInteger(digit)
  ){

    return;

  }


  digits.push(digit);

  prices.push(price);

  tickCount++;


  if(digits.length > 200){

    digits.shift();

  }


  if(prices.length > 200){

    prices.shift();

  }


  updateScreen();

  analyzeDigits();

}


/* =====================================
   SCREEN
===================================== */

function updateScreen(){

  document.getElementById("ticks").innerHTML =
    tickCount;


  if(prices.length){

    document.getElementById("price").innerHTML =
      prices[prices.length-1];


    document.getElementById("lastDigit").innerHTML =
      digits[digits.length-1];

  }


  if(prices.length >= 2){

    const n =
      prices.length;


    const jump =
      Math.abs(
        prices[n-1] -
        prices[n-2]
      );


    document.getElementById("jump").innerHTML =
      jump.toFixed(5);


    let total = 0;


    for(
      let i=1;
      i<prices.length;
      i++
    ){

      total +=
        Math.abs(
          prices[i] -
          prices[i-1]
        );

    }


    const average =
      total /
      (prices.length-1);


    document.getElementById("avgJump").innerHTML =
      average.toFixed(5);

  }


  document.getElementById("recent").innerHTML =
    digits
      .slice(-50)
      .join(" ");

}


/* =====================================
   DIGIT ANALYSIS
===================================== */

function analyzeDigits(){

  if(digits.length < 10){

    document.getElementById("signal").innerHTML =
      "SCANNING — " +
      digits.length +
      "/10 ticks";

    return;

  }


  const count =
    [0,0,0,0,0,0,0,0,0,0];


  for(
    let i=0;
    i<digits.length;
    i++
  ){

    count[digits[i]]++;

  }


  const total =
    digits.length;


  let bestDigit = 0;

  let bestCount = count[0];


  for(
    let i=0;
    i<10;
    i++
  ){

    const percentage =
      (count[i]/total)*100;


    document.getElementById(
      "p"+i
    ).innerHTML =
      percentage.toFixed(1)+"%";


    if(
      count[i] > bestCount
    ){

      bestCount =
        count[i];

      bestDigit =
        i;

    }

  }


  const score =
    (bestCount/total)*100;


  document.getElementById("match").innerHTML =
    bestDigit;


  document.getElementById("signal").innerHTML =
    "STRONGEST OBSERVED DIGIT: " +
    bestDigit +
    " — " +
    score.toFixed(1) +
    "%";

}


/* =====================================
   JAVASCRIPT TEST
===================================== */

console.log(
  "Deriv Match Analyzer loaded successfully."
);

</script>

</body>
</html>
