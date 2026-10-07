<script>

let socket = null;
let currentDigit = null;
let analyzedDigit = null;
let tradeBusy = false;

const symbolSelect = document.getElementById("symbol");
const digitDisplay = document.getElementById("digit");
const statusDisplay = document.getElementById("status");
const analyzedDisplay = document.getElementById("analyzed");
const tradeStatus = document.getElementById("tradeStatus");
const analyzeButton = document.getElementById("analyze");
const tradeButton = document.getElementById("trade");


/* ================================
   GET LAST DIGIT
================================ */

function getLastDigit(price) {

    const text = String(price);

    const parts = text.split(".");

    if (parts.length === 1) {
        return Number(text[text.length - 1]);
    }

    const decimals = parts[1];

    return Number(decimals[decimals.length - 1]);
}


/* ================================
   CONNECT TO DERIV
================================ */

function connectToDeriv() {

    statusDisplay.textContent =
        "Connecting to Deriv...";

    socket = new WebSocket(
        "wss://api.derivws.com/trading/v1/options/ws/public"
    );


    socket.onopen = function () {

        statusDisplay.textContent =
            "Connected to Deriv";

        subscribeToTicks();

    };


    socket.onmessage = function (event) {

        try {

            const data = JSON.parse(event.data);

            console.log("DERIV:", data);


            if (data.error) {

                statusDisplay.textContent =
                    "Deriv error: " +
                    data.error.message;

                return;
            }


            if (data.msg_type === "tick") {

                const price =
                    data.tick.quote;


                currentDigit =
                    getLastDigit(price);


                digitDisplay.textContent =
                    currentDigit;


                statusDisplay.textContent =
                    "Live";


            }

        } catch (error) {

            console.error(error);

        }

    };


    socket.onerror = function (error) {

        console.error(
            "WebSocket error:",
            error
        );

        statusDisplay.textContent =
            "Connection error";

    };


    socket.onclose = function () {

        statusDisplay.textContent =
            "Disconnected - reconnecting...";


        setTimeout(
            connectToDeriv,
            3000
        );

    };

}


/* ================================
   SUBSCRIBE TO TICKS
================================ */

function subscribeToTicks() {

    const symbol =
        symbolSelect.value;


    if (
        !socket ||
        socket.readyState !== WebSocket.OPEN
    ) {
        return;
    }


    socket.send(
        JSON.stringify({

            ticks: symbol,

            subscribe: 1

        })
    );

}


/* ================================
   CHANGE MARKET
================================ */

symbolSelect.addEventListener(
    "change",
    function () {

        currentDigit = null;

        digitDisplay.textContent = "-";

        subscribeToTicks();

    }
);


/* ================================
   ANALYZE
================================ */

analyzeButton.onclick = function () {

    if (currentDigit === null) {

        statusDisplay.textContent =
            "Waiting for a live digit.";

        return;
    }


    /*
       SAVE THE EXACT DIGIT
    */

    analyzedDigit =
        currentDigit;


    analyzedDisplay.textContent =
        analyzedDigit;


    tradeStatus.textContent =
        "Ready";


    statusDisplay.textContent =
        "Digit " +
        analyzedDigit +
        " selected";


    tradeButton.disabled = false;

};


/* ================================
   TRADE
================================ */

tradeButton.onclick =
async function () {

    if (tradeBusy) {
        return;
    }


    if (analyzedDigit === null) {

        statusDisplay.textContent =
            "Analyze first.";

        return;
    }


    /*
       LOCK BUTTON
    */

    tradeBusy = true;

    tradeButton.disabled = true;


    /*
       FREEZE THE EXACT DIGIT.

       Example:

       Analyze = 5

       Even if live screen changes:

       5 → 7 → 2

       THIS TRADE REMAINS 5.
    */

    const digitToTrade =
        analyzedDigit;


    const symbol =
        symbolSelect.value;


    const stake =
        Number(
            document.getElementById("stake").value
        );


    statusDisplay.textContent =
        "Sending DIGITMATCH " +
        digitToTrade +
        "...";


    tradeStatus.textContent =
        "Sending";


    try {

        const response =
            await fetch(
                "YOUR_BACKEND_URL/trade",
                {

                    method: "POST",

                    headers: {
                        "Content-Type":
                            "application/json"
                    },

                    body: JSON.stringify({

                        digit:
                            digitToTrade,

                        symbol:
                            symbol,

                        stake:
                            stake

                    })

                }
            );


        const result =
            await response.json();


        if (!response.ok) {

            throw new Error(
                result.error ||
                "Trade failed"
            );

        }


        statusDisplay.textContent =
            "TRADED DIGIT " +
            digitToTrade +
            " ONCE";


        tradeStatus.textContent =
            "Completed";


        /*
           FORCE USER TO ANALYZE
           THE NEXT DIGIT BEFORE
           ANOTHER TRADE.
        */

        analyzedDigit = null;

        analyzedDisplay.textContent =
            "None";


    } catch (error) {

        console.error(error);

        statusDisplay.textContent =
            "Trade error: " +
            error.message;


        tradeStatus.textContent =
            "Failed";

    }


    tradeBusy = false;

    tradeButton.disabled = true;

};


/* ================================
   START
================================ */

connectToDeriv();

</script>
