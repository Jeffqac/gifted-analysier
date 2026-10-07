<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1.0">
<title>Deriv Last Digit Trader</title>

<style>
*{box-sizing:border-box}

body{
  margin:0;
  font-family:Arial,sans-serif;
  background:#0b1020;
  color:#fff;
  padding:20px;
}

.container{
  max-width:500px;
  margin:auto;
  background:#151c31;
  padding:25px;
  border-radius:18px;
  text-align:center;
}

h1{
  margin-top:0;
}

.label{
  color:#aab3cc;
  margin-top:20px;
}

#digit{
  font-size:100px;
  font-weight:bold;
  margin:10px 0;
}

#status{
  min-height:25px;
  margin:15px 0;
}

input,select,button{
  width:100%;
  padding:15px;
  margin-top:12px;
  border-radius:10px;
  border:0;
  font-size:16px;
}

button{
  font-weight:bold;
  cursor:pointer;
}

#analyze{
  background:#2563eb;
  color:white;
}

#trade{
  background:#16a34a;
  color:white;
}

button:disabled{
  background:#555!important;
  cursor:not-allowed;
}

.info{
  background:#0d1426;
  padding:15px;
  border-radius:10px;
  margin-top:20px;
  text-align:left;
}
</style>
</head>

<body>

<div class="container">

<h1>LAST DIGIT TRADER</h1>

<div class="label">LIVE LAST DIGIT</div>

<div id="digit">-</div>

<div id="status">Connecting...</div>

<select id="symbol">
<option value="1HZ100V">Volatility 100 (1s)</option>
<option value="1HZ75V">Volatility 75 (1s)</option>
<option value="1HZ50V">Volatility 50 (1s)</option>
<option value="1HZ25V">Volatility 25 (1s)</option>
<option value="1HZ10V">Volatility 10 (1s)</option>
</select>

<input
 id="stake"
 type="number"
 value="1"
 min="0.35"
 step="0.01"
 placeholder="Stake"
/>

<button id="analyze">
ANALYZE CURRENT DIGIT
</button>

<button id="trade" disabled>
TRADE CURRENT DIGIT ONCE
</button>

<div class="info">

<div>
<strong>Analyzed digit:</strong>
<span id="analyzed">None</span>
</div>

<div style="margin-top:10px">
<strong>Trade status:</strong>
<span id="tradeStatus">Waiting</span>
</div>

</div>

</div>


<script>

/* =====================================================
   SETTINGS
   ===================================================== */

/*
   PUT YOUR SECURE TRADING BACKEND URL HERE.

   Example:

   const BACKEND_URL =
   "https://your-server.example.com";

   DO NOT PUT YOUR DERIV TOKEN HERE.
*/

const BACKEND_URL = "YOUR_BACKEND_URL";


/* =====================================================
   VARIABLES
   ===================================================== */

let socket = null;

let currentDigit = null;

let analyzedDigit = null;

let tradeBusy = false;


/* =====================================================
   GET LAST DIGIT
   ===================================================== */

function getLastDigit(value){

    let text = String(value);

    /*
       Remove decimal point.
    */

    text = text.replace(".", "");

    /*
       Get the final number.
    */

    return Number(text[text.length - 1]);
}


/* =====================================================
   CONNECT TO DERIV LIVE MARKET
   ===================================================== */

function connect(){

    socket = new WebSocket(
        "wss://ws.derivws.com/websockets/v3?app_id=1089"
    );


    socket.onopen = function(){

        document.getElementById("status").textContent =
            "Connected";

        subscribe();

    };


    socket.onmessage = function(event){

        const data = JSON.parse(event.data);


        if(data.msg_type !== "tick"){
            return;
        }


        const quote = data.tick.quote;


        /*
           Get the live last digit.
        */

        currentDigit = getLastDigit(quote);


        /*
           Display it.
        */

        document.getElementById("digit").textContent =
            currentDigit;

    };


    socket.onerror = function(){

        document.getElementById("status").textContent =
            "Connection error";

    };


    socket.onclose = function(){

        document.getElementById("status").textContent =
            "Disconnected. Reconnecting...";

        setTimeout(connect,3000);

    };

}


/* =====================================================
   SUBSCRIBE TO MARKET
   ===================================================== */

function subscribe(){

    const symbol =
        document.getElementById("symbol").value;


    socket.send(JSON.stringify({

        ticks:symbol,

        subscribe:1

    }));

}


/* =====================================================
   CHANGE MARKET
   ===================================================== */

document.getElementById("symbol").addEventListener(
    "change",
    function(){

        if(
            socket &&
            socket.readyState === WebSocket.OPEN
        ){

            subscribe();

        }

    }
);


/* =====================================================
   ANALYZE
   ===================================================== */

document.getElementById("analyze").onclick =
function(){

    if(currentDigit === null){

        document.getElementById("status").textContent =
            "Waiting for live digit...";

        return;

    }


    /*
       THIS IS THE IMPORTANT PART.

       The exact digit currently displayed
       becomes the digit selected for trading.
    */

    analyzedDigit = currentDigit;


    document.getElementById("analyzed").textContent =
        analyzedDigit;


    document.getElementById("tradeStatus").textContent =
        "Ready to trade";


    document.getElementById("status").textContent =
        "Digit " + analyzedDigit +
        " selected";


    /*
       Enable Trade.
    */

    document.getElementById("trade").disabled =
        false;

};


/* =====================================================
   TRADE EXACT ANALYZED DIGIT
   ===================================================== */

document.getElementById("trade").onclick =
async function(){

    /*
       Stop double clicks.
    */

    if(tradeBusy){
        return;
    }


    /*
       Make sure Analyze was pressed.
    */

    if(analyzedDigit === null){

        document.getElementById("status").textContent =
            "Analyze a digit first.";

        return;

    }


    /*
       LOCK IMMEDIATELY.
    */

    tradeBusy = true;

    document.getElementById("trade").disabled =
        true;


    /*
       FREEZE THE EXACT DIGIT.

       If Analyze selected 5,
       digitToTrade remains 5.

       Even if the next tick becomes 7,
       this trade remains 5.
    */

    const digitToTrade =
        analyzedDigit;


    const symbol =
        document.getElementById("symbol").value;


    const stake =
        Number(
            document.getElementById("stake").value
        );


    document.getElementById("status").textContent =
        "Trading DIGITMATCH " +
        digitToTrade +
        "...";


    document.getElementById("tradeStatus").textContent =
        "Sending";


    try{

        /*
           Send the exact selected digit
           to the secure trading server.
        */

        const response =
            await fetch(
                BACKEND_URL + "/trade",
                {
                    method:"POST",

                    headers:{
                        "Content-Type":
                            "application/json"
                    },

                    body:JSON.stringify({

                        digit:digitToTrade,

                        symbol:symbol,

                        stake:stake

                    })
                }
            );


        const result =
            await response.json();


        if(!response.ok){

            throw new Error(
                result.error ||
                "Trade failed"
            );

        }


        document.getElementById("status").textContent =
            "TRADE SENT: DIGITMATCH " +
            digitToTrade;


        document.getElementById("tradeStatus").textContent =
            "Trade completed";


        /*
           Clear the selected digit.

           Another Analyze is required
           before another trade.
        */

        analyzedDigit = null;

        document.getElementById("analyzed").textContent =
            "None";

    }


    catch(error){

        document.getElementById("status").textContent =
            "ERROR: " + error.message;


        document.getElementById("tradeStatus").textContent =
            "Failed";

    }


    finally{

        tradeBusy = false;

        /*
           Keep Trade disabled until
           another digit is analyzed.
        */

        document.getElementById("trade").disabled =
            true;

    }

};


/* =====================================================
   START
   ===================================================== */

connect();

</script>

</body>
</html>
